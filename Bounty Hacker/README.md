# Bounty Hacker

**Platform:** TryHackMe · **Difficulty:** Easy · **OS:** Linux
**Chain:** FTP (anonymous) → SSH (hydra) → sudo tar → root

## Enumeration

```
21/tcp  open  ftp     vsftpd 3.0.5
22/tcp  open  ssh     OpenSSH 8.2p1 (Ubuntu)
80/tcp  open  http    Apache httpd 2.4.41 (Ubuntu)
```

Three open ports. Plan: gather info from HTTP and FTP, then use any credentials
found to get a foothold over SSH.

## Foothold

- **FTP (21) — anonymous login:** Two files inside: `locks.txt` and `task.txt`.
  - `locks.txt` → a list of 26 candidate passwords.
  - `task.txt` → a note about the Red Eye operation. The useful part wasn't the
    content but the **signature at the bottom: `lin`** → the username.
- **SSH (22) — hydra:** Brute-force with the correct user (`lin`) + password list:
  ```
  hydra -l lin -P locks.txt ssh://<IP>
  ```
  Password recovered, foothold obtained as `lin` over SSH.

## Privilege Escalation

- `sudo -l` → the only command `lin` can run as root: `(root) /bin/tar`
- On GTFOBins, `tar` → Sudo → checkpoint-action technique:
  ```
  sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
  ```
- Root shell dropped → `/root/root.txt`.

## Where I Got Stuck

- **Pointless id_rsa attempt:** Seeing an encrypted `id_rsa` in `lin`'s `.ssh`
  directory, I reflexively tried to crack it with `ssh2john` + `john`. Wrong move —
  I was already logged in as `lin`, so cracking lin's own key wouldn't move me to a
  new user. The key also didn't crack with `rockyou.txt` anyway. Roughly 40 minutes
  wasted.
- **Takeaway:** Speed isn't critical on these boxes yet, but it is on the OSCP exam.
  I need to build the right reflexes now — run `sudo -l` first thing after a
  foothold, and ask "does this move me somewhere new?" before diving into any vector.
