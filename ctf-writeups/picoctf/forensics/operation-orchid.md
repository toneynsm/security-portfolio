# Operation Orchid

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Loaded the disk image into Autopsy and ran keyword searches for everything obvious -- nothing turned up. Switched to browsing files directly under the plaintext file type filter. Found `flag.txt` but extracting it revealed nothing useful, and neither did its slack space entry. The only other interesting file was `.ash_history` -- the command history file that records every command a user typed in their terminal session. That file told the whole story:

The user created `flag.txt`, encrypted it with AES-256 using the password `unbreakablepassword1234567`, then ran `shred` to securely delete the original. `shred` is a deliberate anti-forensics move -- unlike normal deletion which just removes the pointer, shred overwrites the file's data multiple times to prevent recovery. But the encrypted copy `flag.txt.enc` was still sitting on the disk, and the history file recorded everything.

Extracted `flag.txt.enc`, SSHd into Kali, transferred the file over with SCP, and decrypted it:

```
openssl aes256 -d -salt -in flag.txt.enc -out flag.txt -k unbreakablepassword1234567
```

Flag was in the output.

**Flag:** `picoCTF{h4un71ng_p457_1d02081e}`

**What I learned:** Command history files are gold in a forensic investigation. `.ash_history`, `.bash_history`, `.zsh_history` -- these files record every command a user ran and are often overlooked by attackers covering their tracks. The attacker here was careful enough to use `shred` on the plaintext flag but forgot to wipe the history that documented the entire operation. That single oversight unraveled everything.

From an attacker perspective, proper anti-forensics means clearing or manipulating history files, not just deleting sensitive files. Running `unset HISTFILE` before a session prevents commands from being recorded at all. A completely empty or missing history file is itself suspicious to an analyst -- it signals something was deliberately wiped even if the contents are gone. The goal is to look like nothing happened, not to leave obvious evidence of a cleanup.

---

*[← Back to Forensics](README.md)*
