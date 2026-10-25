# OSCP Writeups

Penetration testing walkthroughs from my OSCP preparation — TryHackMe, Hack The Box, and PWK labs. Each machine is documented with the full attack chain, the reasoning behind each step, and honest notes on what I still need to improve.

🇹🇷 Türkçe açıklama aşağıda.

## Machines

| Machine          | Platform  | Chain                                                           | Writeup |
|------------------|-----------|-----------------------------------------------------------------|---------|
| Basic Pentesting | TryHackMe | SMB → SSH → sudo                                                | [EN](basic-pentesting/README.md) · [TR](basic-pentesting/README.tr.md) |
| Vulnversity      | TryHackMe | File upload → SUID exec                                         | [EN](vulnversity/README.md) · [TR](vulnversity/README.tr.md) |
| Bounty Hacker    | TryHackMe | FTP → SSH → sudo tar                                            | [EN](bounty-hacker/README.md) · [TR](bounty-hacker/README.tr.md) |
| tomghost         | TryHackMe | Ghostcat → PGP crack → sudo zip                                 | [EN](tomghost/README.md) · [TR](tomghost/README.tr.md) |
| Anonymous        | TryHackMe | FTP write → root cron → SUID env                                | [EN](anonymous/README.md) · [TR](anonymous/README.tr.md) |
| Mr Robot         | TryHackMe | WordPress creds → PHP reverse shell (daemon) → SUID nmap → root | [EN](mr-robot/README.md) · [TR](mr-robot/README.tr.md) |
| Kiba             | TryHackMe | Kibana RCE (CVE-2019-7609) → cap_setuid → root                  | [EN](kiba/README.md) · [TR](kiba/README.tr.md) |

## About

Final-year Software Engineering student preparing for the OSCP certification. These writeups track my methodology as it develops — credentials and flags are redacted in line with platform policy.

---

# OSCP Çözüm Yazıları

OSCP hazırlığım sırasında yazdığım sızma testi çözümleri — TryHackMe, Hack The Box ve PWK laboratuvarları. Her makine; tam saldırı zinciri, her adımın gerekçesi ve hâlâ geliştirmem gereken noktalara dair dürüst notlarla belgelenmiştir.

Makine listesi yukarıdaki tabloda; her satırda İngilizce (EN) ve Türkçe (TR) çözüm bağlantıları var.
