# Phase 1: Recon

**Goal:** Map the DMZ network from the attacker's perspective: find live hosts, open ports, running services, and software versions. Typically, I (or any attacker) would emphasize imperceptibility; but for the blue team portion of this capstone, I need a little bit of noise. That's why you'll see -T4 in the command below.

---

## Red Team

### Host Discovery

```bash
nmap -sn 192.168.100.0/24
```

**Hosts found:**

| IP | Hostname | Notes |
|---|---|---|
| 192.168.100.20 | ubuntu-dvwa | Apache + SSH |
| 192.168.100.30 | splunk | Splunk SIEM |

---

### Port & Service Enumeration

Full port scan with version detection and default scripts, output saved for reference (via Obsidian):

```bash
nmap -sS -sV -sC -p- -T4 192.168.100.20 -oN Recon/nmap_full.txt
```

![Nmap full port scan of 192.168.100.20 — ports 22 (OpenSSH 9.6p1) and 80 (Apache 2.4.58) open](../screenshots/phase1-nmap-scan.png)

**Key findings:**

| Host | Port | Service | Version | Notes |
|---|---|---|---|---|
| .100.20 | 22 | SSH | OpenSSH 9.6p1 | Ubuntu Linux |
| .100.20 | 80 | HTTP | Apache 2.4.58 | Ubuntu default page at `/`, DVWA at `/dvwa/` |

---

### Web Application Enumeration

Navigating to `http://192.168.100.20/dvwa/` confirmed DVWA was running. Default credentials `admin / password` granted access. 

**A note on the target URL:** The scan was run directly against `/dvwa/` because the DVWA application was already known from setup. In a real engagement you wouldn't know this upfront. The correct approach would be to run Gobuster against the root URL (`http://192.168.100.20/`) first, discover `/dvwa/` as a result, and then enumerate inside it. This lab skips that discovery step since the application was installed by hand.

![DVWA home page at http://192.168.100.20/dvwa/](../screenshots/phase1-dvwa-home.png)

Directory brute force to map the application structure and find web-accessible upload paths:

```bash
gobuster dir -u http://192.168.100.20/dvwa/ -w /usr/share/wordlists/dirb/common.txt -o Recon/gobuster.txt
```

![Gobuster directory enumeration — found /config, /database, /docs, /external, /index.php, and more](../screenshots/phase1-gobuster.png)

Security level confirmed and set before exploitation.

![DVWA Security page — security level set to Low](../screenshots/phase1-dvwa-security-low.png)

**DVWA findings:**
- Default credentials: `admin / password`
- Security level set to: **Low**
- Interesting pages/functionality: File Upload at `/dvwa/vulnerabilities/upload/`, upload directory web-accessible at `/dvwa/hackable/uploads/`

---

## Blue Team — What Splunk Saw

### Detection Queries

```
index=* src_ip=192.168.100.10
| stats count by dest_port
| where count > 50
| sort -count
```

```
index=* src_ip=192.168.100.10 dest_port=80
| stats count by uri_path
| sort -count
```

### What Fired

The nmap scan hitting all 65535 ports generated a distinct spike — a single source touching hundreds of ports in seconds is a reliable scan signature. The gobuster run produced a high volume of 404s and redirects against port 80 from the same source, detectable via the URI path frequency query.

### What Was Missed

The nmap `-sS` SYN scan sends SYN packets and records the response without completing the TCP handshake — no application-layer connection is established, so web server logs see nothing. Only raw socket events at the OS level would catch it, and those weren't forwarded to Splunk in this lab.

---

## Key Takeaways

- **Version disclosure is a liability.** Apache and OpenSSH both returned exact version strings in nmap output — enough to look up known CVEs immediately.
- **Directory brute force is loud.** Hundreds of 4xx responses from one IP in seconds is trivially detectable, but only if someone's watching. In the real world, it's likely that it won't be THIS easy/informative to blue team.

---

← [Phase 0 — Setup](phase0-setup.md) | [Phase 2 — Initial Access](phase2-initial-access.md) →
