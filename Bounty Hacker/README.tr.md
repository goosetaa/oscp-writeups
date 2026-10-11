# Bounty Hacker

**Platform:** TryHackMe · **Zorluk:** Easy · **OS:** Linux
**Zincir:** FTP (anonymous) → SSH (hydra) → sudo tar → root

## Keşif (Enumeration)

```
21/tcp  open  ftp     vsftpd 3.0.5
22/tcp  open  ssh     OpenSSH 8.2p1 (Ubuntu)
80/tcp  open  http    Apache httpd 2.4.41 (Ubuntu)
```

Üç açık port gördüm. Planım: HTTP ve FTP'den bilgi toplamak, oradan elde ettiğim
kimlik bilgileriyle SSH üzerinden foothold almak.

## Foothold

- **FTP (21) — anonymous login:** İçeride iki dosya vardı: `locks.txt` ve `task.txt`.
  - `locks.txt` → 26 satırlık bir parola listesi.
  - `task.txt` → Red Eye operasyonuna dair bir not. Kritik olan içeriği değil,
    **notun altındaki imzaydı: `lin`** → kullanıcı adı.
- **SSH (22) — hydra:** Doğru kullanıcı (`lin`) + parola listesiyle brute-force:
  ```
  hydra -l lin -P locks.txt ssh://<IP>
  ```
  Parola çıktı, SSH ile `lin` olarak foothold alındı.

## Privilege Escalation

- `sudo -l` → `lin` kullanıcısının root olarak çalıştırabildiği tek komut: `(root) /bin/tar`
- GTFOBins'te `tar` → Sudo → checkpoint-action tekniği:
  ```
  sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
  ```
- Root shell düştü → `/root/root.txt`.

## Takıldığım Yerler

- **Gereksiz id_rsa denemesi:** `lin`'in `.ssh` dizininde şifreli bir `id_rsa`
  görünce reflekse `ssh2john` + `john` ile kırmaya giriştim. Yanlıştı — zaten `lin`
  olarak girmiştim, kendi key'ini kırmak beni yeni bir kullanıcıya taşımazdı. Ayrıca
  key `rockyou.txt` ile zaten kırılmadı. Yaklaşık 40 dakika boşa gitti.
- **Çıkarım:** Bu makinelerde şu an hız kritik değil, ama OSCP'de süre sınırlı.
  Doğru refleksleri (foothold sonrası ilk iş `sudo -l`, her vektöre dalmadan önce
  "bu beni yeni bir yere taşır mı?" sorusu) şimdiden oturtmam gerekiyor.
