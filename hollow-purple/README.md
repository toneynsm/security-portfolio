# Hollow Purple

A purple team capstone project (nicknamed after one of my favorite animes) built on 4 virtual machines in VMware Workstation Pro. I simulated a full corporate network breach from initial recon all the way through data exfiltration, then built the detection and monitoring infrastructure to catch it.

Purple team means both sides. The red team attack chain ran first: planning, attacking, pivoting, exfiltrating. The blue team came second: a new SIEM was deployed, custom detection rules were written from scratch, alert noise was tuned out, and analyst workflow was practiced against simulated attacks. Both halves are fully documented here.

---

## Lab Architecture

```
VMnet1 — 192.168.100.x (DMZ)
  Kali Linux (attacker)         192.168.100.10
  Ubuntu + DVWA (target)        192.168.100.20
  Wazuh Manager (SIEM)          192.168.100.70

VMnet2 — 192.168.200.x (Internal)
  Ubuntu + DVWA (pivot)         192.168.200.20
  Metasploitable3                192.168.200.10
  Wazuh Manager (SIEM)          192.168.200.70

Wazuh dashboard                 https://192.168.78.137
```

Kali can only see the 100.x network. Metasploitable3 only lives on the 200.x network. The only path from one to the other is through Ubuntu, which has one network card on each segment. This forces lateral movement: the same technique real attackers use to move deeper into a company after getting an initial foothold.

VMware's host-only mode keeps both networks completely isolated. The VMs can only reach each other.

---

## Machines

| Machine | Role | OS | IP(s) |
|---|---|---|---|
| Kali Linux | Attacker | Kali Linux | 192.168.100.10 |
| Ubuntu + DVWA | DMZ web server, lateral movement pivot | Ubuntu 22.04 | 192.168.100.20 / 192.168.200.20 |
| Metasploitable3 | Internal legacy server | Ubuntu 14.04 | 192.168.200.10 |
| Wazuh Manager | SIEM, log indexer, dashboard | Ubuntu 22.04 | 192.168.100.70 / 192.168.200.70 |

---

## Red Team

Full attack chain from outside the network to internal data exfiltration. No automation frameworks, all manual.

| Part | What happened |
|---|---|
| [Part 0: Lab Setup](red-team/part0-setup.md) | Built the 4-VM isolated lab from scratch, including networking, DVWA, and Metasploitable3 |
| [Part 1: Recon](red-team/part1-recon.md) | Mapped the DMZ, discovered open ports and services, explored the DVWA web application |
| [Part 2: Initial Access](red-team/part2-initial-access.md) | Exploited DVWA's unrestricted file upload, uploaded a PHP web shell, upgraded to a reverse shell |
| [Part 3: Privilege Escalation](red-team/part3-privesc.md) | Abused a misconfigured SUID binary to escalate from www-data to root |
| [Part 4: Lateral Movement](red-team/part4-lateral-movement.md) | Reverse SSH tunnel, ProFTPD modcopy exploit, PwnKit privilege escalation, root on Metasploitable3 |
| [Part 5: Exfiltration](red-team/part5-exfil.md) | Located and exfiltrated the sensitive employee data file back to Kali via netcat |

---

## Blue Team

SIEM deployment, detection engineering, SOAR integration, and analyst workflow. The blue team work stands on its own: it does not mirror the red team phases one for one. It covers detection engineering, alert tuning, attack simulation, and investigation.

| Section | What is inside |
|---|---|
| [Overview](blue-team/overview.md) | How every tool works, what the full stack looks like, and why each piece exists |
| [Detection Engineering](blue-team/detection-engineering.md) | Wazuh and auditd setup, all custom detection rules, suppression rules, Shuffler SOAR integration |
| [Analyst Workflow](blue-team/analyst-workflow.md) | Atomic Red Team simulations, Wazuh investigation, DQL queries, key findings, IR lifecycle |
