# Disko 1

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded a .dd file -- a raw disk image, meaning a bit-for-bit copy of an entire disk. First tried OSFMount to mount and browse it like a normal drive, but couldn't find anything by just clicking around. Figured out I needed a proper forensic analysis tool to actually search through it. Landed on Autopsy -- the free, industry standard DFIR tool. Loaded the .dd file directly into Autopsy, enabled keyword search with Solr indexing, searched for "picoCTF" and the flag came back from file slack space.

**Flag:** `picoCTF{1t5_ju5t_4_5tr1n9_e3408eef}`

**What I learned:** OSFMount lets you browse a disk image like a normal drive -- good for a quick first look. Autopsy is a full forensic analysis tool that reads raw disk sectors, recovers deleted files, and searches places normal file browsing can't reach. DFIR (Digital Forensics and Incident Response) is the discipline focused on investigating what happened after a breach -- reconstructing attacker timelines, identifying what was accessed, and preserving evidence properly.

**File slack space:** Hard drives store data in fixed-size blocks. Files rarely fill a block exactly, leaving leftover space at the end called file slack. That leftover space isn't empty -- it contains whatever data was physically written there before, because the OS never bothers wiping it when a file is deleted. It just removes the pointer (the reference telling the system where the file lives) and marks the space available. The actual data stays on the disk until something new overwrites it directly. Autopsy bypasses the file system entirely and reads raw sectors, which is how it finds data that's invisible to normal file browsing.

From an attacker perspective, slack space is a place to hide data that won't show up in Explorer. From an investigator perspective it's a place where deleted files, old documents, and traces of attacker activity accumulate -- and a common source of evidence in real criminal cases. Knowing where to look and recognizing what's significant when you find it is what separates a skilled DFIR investigator from someone just running tools.

---

*[← Back to Forensics](README.md)*
