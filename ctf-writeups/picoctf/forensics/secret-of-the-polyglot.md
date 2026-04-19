# Secret of the Polyglot

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded the suspicious file. First move was running ExifTool against it to pull the metadata. The file had a `.pdf` extension and opened as a PDF, but the metadata told a different story -- `File Type: PNG`, `MIME Type: image/png`. That's the tell.

Knew at that point it was a polyglot -- a file that contains two valid file structures simultaneously, meaning it can be interpreted as two completely different file types depending on how you open it. Tried converting it to PNG in Adobe Acrobat and reading the new metadata -- nothing useful there. Installed and ran polyfile to do a deeper structural analysis -- nothing actionable came out of that either. Eventually tried the simplest thing: renamed the extension from `.pdf` to `.png` and opened it as an image. That revealed the other half of the flag. The answer was simpler than the rabbit holes I went down to find it.

**Flag:** `picoCTF{f1u3n7_1n_pn9_&_pdf_53b741d6}`

**What I learned:** A polyglot file is a single file that is simultaneously valid in two or more file formats. This works because different file formats look for their headers and structure in different parts of the file -- a PDF reader looks at one section, a PNG reader looks at another. An attacker can craft a file that satisfies both parsers at once, hiding different content in each interpretation.

**Why it matters:**

From a red team perspective this is a real malware delivery technique. A file that appears to be a harmless PDF to an email gateway or antivirus scanner might simultaneously be a valid HTML file, JavaScript file, or executable that runs when opened a different way. Security tools that only check file extensions or only parse one format can be bypassed entirely. Polyglot files have been used in real attacks to smuggle malicious payloads past defenses.

From a blue team perspective the lesson is the same one ExifTool keeps teaching -- never trust the extension. File type validation needs to happen at the byte level, not based on what the filename says. A proper security control checks the actual file signature (the magic bytes at the start of a file that identify what it really is) not just the extension a user or attacker gave it. Tools like ExifTool, `file`, and polyfile exist specifically for this -- they read the actual structure of the file, not the label on it.

---

*[← Back to Forensics](README.md)*
