# DeadFace CTF 2026 — "Kingmaker" Writeups

My writeups for the challenges I solved at **DeadFace CTF 2026 ("Kingmaker")**, a beginner-to-intermediate cybersecurity capture-the-flag event. I took part as a competitor and worked through challenges across cryptography, forensics, reverse engineering, web, mobile, network, databases, and Linux privilege escalation.

I'm a cybersecurity student at UniKL, and I write these up partly to learn properly (solving and forgetting isn't learning) and partly as a public record of what I can actually do. Each writeup walks through how I approached the challenge, the commands that worked, why the technique works, and what I took away — not just the final flag.

> **Note on honesty:** every flag here is the exact string the scoreboard accepted. Where something wasn't captured cleanly, I say so. One reverse-engineering flag (Cereal) was shown as an image and I didn't copy the text down, so it's marked *not recorded* rather than reconstructed.

## Challenges

### Cryptography
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 02 | [X Marks the Spot](./02-x-marks-the-spot.md) | X Marks the Spot | Single-byte XOR, broken with frequency analysis |
| 03 | [Digital Design](./03-digital-design.md) | Digital Design | A cipher built from logic gates + A1Z26 |
| 04 | [Chess Key](./04-chess-key.md) | Chess Key | A Vigenère key hidden in a real chess game (OSINT) |

### Steganography / Digital Forensics
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 01 | [Keep Digging](./01-keep-digging.md) | Keep Digging | Editing a JPEG's height header to reveal hidden pixels |

### Reverse Engineering
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 05 | [Cereal](./05-cereal-reverse-engineering.md) | Cereal / Insectoids | Custom base64, SHA-1 check, RC4 flag decryption (Ghidra) |

### Web Security
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 06 | [Doctor Appointment](./06-doctor-appointment-sqli.md) | Doctor Appointment | SQL injection auth bypass + the API leaking the full row |

### Mobile Security
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 07 | [Aurora Patient Portal + Back to Basics](./07-ios-aurora-unzip.md) | Aurora Patient Portal, Back to Basics | Static iOS `.ipa` analysis — Info.plist, SQLite, strings |

### Network Forensics
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 08 | [Challenge Accepted](./08-challenge-accepted-pcap.md) | Challenge Accepted | Recovering an NTLMv2 NTProofStr from an LLMNR poisoning PCAP |

### Database Forensics
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 09 | [After the Lock + Ghost in the Registry](./09-sql-forensics.md) | After the Lock, Ghost in the Registry | Timestamp-vs-policy checks and anti-joins (`NOT EXISTS`) |

### Exploitation / Linux Privilege Escalation
| # | Writeup | Challenge | Key idea |
|---|---------|-----------|----------|
| 10 | [Hostbusters](./10-charon-privesc-chain.md) | Hostbusters (Charon) | An eight-step chain from a service account to root |

## The Hostbusters chain at a glance

The privilege-escalation box (`charon`, reached via `hostbusters.dctf.io`) ran as eight linked challenges, each a different real-world misconfiguration:

| Step | Account reached | Technique |
|------|-----------------|-----------|
| 1 | `svc-deploy` | Enumerate the home directory |
| 2 | `deephax` | Password in `~/.bash_history` |
| 3 | `gh0st404` | Secret in git history |
| 4 | `sunstalker` | Writable script run by cron |
| 5 | `lilith` | `sudo find` (GTFOBins) |
| 6 | `spookyboi` | `cap_setuid` capability on a binary |
| 7 | `mort1cia` | Cracking an md5crypt password with an OSINT wordlist |
| 8 | `root` | SUID binary PATH hijack |

## Repository layout

```
.
├── README.md
├── 01-keep-digging.md
├── 02-x-marks-the-spot.md
├── 03-digital-design.md
├── 04-chess-key.md
├── 05-cereal-reverse-engineering.md
├── 06-doctor-appointment-sqli.md
├── 07-ios-aurora-unzip.md
├── 08-challenge-accepted-pcap.md
├── 09-sql-forensics.md
└── 10-charon-privesc-chain.md
```

## Scope and ethics

Every target referenced here (the `*.dctf.io` hosts, the GhostTown forum, the downloadable artifacts, and the Hostbusters web-shell) was provided by the organisers as part of an authorised competition. Nothing in these writeups was performed against systems outside that scope. The techniques are shared for learning and defence; apply them only where you have explicit permission.

## Author

**Adli Firdaus** — cybersecurity student at UniKL.

- Medium: _coming soon_
- LinkedIn: (https://www.linkedin.com/in/adli-firdaus-555b862b2/?isSelfProfile=true)

If you spot a mistake or have a cleaner approach to any of these, open an issue — I'd like to hear it.
