# Phase 0: Lab Setup

**Goal:** Build an isolated 4-VM lab that forces realistic lateral movement.

---

## Architecture

Two isolated virtual networks in VMware Workstation:
- **VMnet1 (DMZ — 192.168.100.x):** Kali, Ubuntu/DVWA, Splunk
- **VMnet2 (Internal — 192.168.200.x):** Metasploitable3, Ubuntu's second NIC

Host-only mode. The VMs can only reach each other.

---

## Kali

Static IP `192.168.100.10` assigned via `/etc/network/interfaces`. After editing and restarting networking with `sudo systemctl restart networking`, `ip a` confirmed `eth0` picked up the address.

![Kali network configuration — eth0 assigned 192.168.100.10/24](../screenshots/phase0-kali-network-setup.png)

---

## Ubuntu + DVWA

DVWA (Damn Vulnerable Web App) is a deliberately broken web application used for practice. It runs on Apache + PHP + MySQL.

### Problems hit and fixed

**apt couldn't install packages**
Ubuntu was on a host-only network with no internet. `apt` silently skipped MySQL because it couldn't reach the package repositories.

Fix: added a temporary NAT adapter (routes VM traffic through the host's internet connection), installed packages, then removed it.

**NAT adapter had no IP**
The VM had the adapter but Ubuntu's network config (netplan) didn't know about it, so it ignored it.

Fix: edited `/etc/netplan/*.yaml` to add the adapter with `dhcp4: true`, ran `sudo netplan apply`.

Ubuntu's second NIC (`ens34`) was manually configured with a static address on the 192.168.200.x subnet, making it dual-homed: one foot in the DMZ (shared with Kali), one in the internal network (shared with Metasploitable3).

![Ubuntu ens34 (internal NIC) manually assigned 192.168.200.20/24 via the installer network config screen](../screenshots/phase0-ubuntu-nic-config.png)

---

## Metasploitable3

A deliberately vulnerable VM from Rapid7 simulating a neglected internal server.

Built using **Vagrant** (Infrastructure as Code): instead of manually clicking through setup, a `Vagrantfile` describes exactly what to build and Vagrant builds it automatically, identically every time. Same philosophy as Terraform (cloud infra), Ansible (server config), Docker (containers). This was my first practical experience with IaC, and I further realized that the main advantage to IaC is speed and reduction of human error.

**Setup:**
```bash
vagrant up --provider=vmware_desktop
```

Static IP `192.168.200.10` assigned in `/etc/network/interfaces`. Fake sensitive data planted at `/opt/corpnet/finance/employee_records.csv` (fake employee names, salaries, SSNs) -- this is what I'm looking for in Phase 5 (what would be my flag if this were a CTF).

Planting required `sudo su` to write as root. The `vagrant` user couldn't write to `/opt/corpnet/finance/` directly. The file was then set to `chmod 600` (root read-only) to simulate a realistic permissions boundary an attacker would have to work around.

![Planting employee_records.csv on Metasploitable3 — sudo required; chmod 600 restricts read to root only](../screenshots/phase0-metasploitable3-plant-csv.png)

---

## Splunk

Splunk is the SIEM (security information & event management) system: it collects logs from machines across the network, indexes them, and lets you search and alert on suspicious activity. This is how the blue team sees what's happening: every login, connection, and command executed gets shipped here. Logs are just receipts for network activity.

**Universal Forwarder** installed on Ubuntu and Metasploitable3; it's just a lightweight agent that watches log files and ships new entries to Splunk at `192.168.100.30:9997` in real time. Just a plugin on those devices.

### Problem hit

**Metasploitable3 forwarder couldn't reach Splunk**
Metasploitable3 is on 200.x. Splunk is on 100.x. Different subnets with no direct path.

Fix with three changes required:
1. On Metasploitable3: route 192.168.100.x traffic via Ubuntu (`192.168.200.20`)
2. On Splunk: route 192.168.200.x traffic via Ubuntu (`192.168.100.20`)
3. On Ubuntu: enable IP forwarding (`net.ipv4.ip_forward=1`) so it passes traffic between its two NICs instead of dropping it

Without IP forwarding, Ubuntu receives packets addressed to someone else and drops them. With it enabled, it passes them along, making it a router.

Both routes made permanent: Metasploitable3 via `post-up` in `/etc/network/interfaces`, Splunk route via netplan.

**Verifying log flow:**
```
index=_internal source=*metrics.log group=tcpin_connections
| stats count by hostname
```
Both `ubuntudvwa` and `metasploitable3-ub1404` appeared. Logs flowing.

![Splunk confirming both forwarders are shipping logs — metasploitable3-ub1404 (72 events) and ubuntudvwa (6 events)](../screenshots/phase0-splunk-log-verification.png)

---

## Key Takeaways

- **Networks don't configure themselves.** Even in a 4-VM lab, routing between subnets requires explicit config. IP forwarding, static routes, and gateway settings all have to be right or traffic silently goes nowhere.
- **Protocols vs. applications.** HTTP, SMB, TCP are standards: documents defining how communication should work. Protocols themselves are abstract compared to the applications that apply them. Apache, Samba, and other software are what actually implement the protocols. This distinction makes troubleshooting significantly easier.
- **IaC is cool, and it's everywhere.** Vagrant, Terraform, Ansible, Docker all solve the same problem from different angles: replace manual, inconsistent human config with reproducible code.
- **Static routes don't persist by default.** `ip route add` is temporary and only lasts until reboot. To make routing permanent it has to live in netplan or `/etc/network/interfaces`.

---

Next: [Phase 1 — Recon](phase1-recon.md)
