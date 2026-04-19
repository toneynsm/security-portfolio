# Corrupted File

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded the file -- no extension, opened as gibberish in Notepad, ExifTool gave nothing useful. Researched how to read a file header manually and ran a PowerShell command to pull the first 20 bytes in hex:

```
Get-Content "C:\Users\nicks\Downloads\file" -Encoding Byte -TotalCount 20 | ForEach-Object { '{0:X2}' -f $_ }
```

Output started with `5C 78` -- a quick lookup confirmed a valid JPEG should start with `FF D8 FF E0`. The header was corrupted. Renaming to `.jpg` didn't work because the bytes were wrong regardless of the extension. Ran `Format-Hex` to get a full hexdump and looked at the structure. Downloaded HxD, a free hex editor that lets you view and edit raw file data at the byte level, manually corrected the first bytes to match the valid JPEG signature, saved it with a `.jpg` extension, opened it, and got the flag.

**Flag:** `picoCTF{r3st0r1ng_th3_by73s_1512b52a}`

**What I learned:** Every file format has magic bytes -- a specific sequence of bytes at the very beginning of the file that identifies what it actually is, regardless of its extension. JPEG files always start with `FF D8 FF E0`. PNG files always start with `89 50 4E 47`. When those bytes are wrong or missing, nothing can read the file correctly. Hex is base-16 -- it maps one byte to exactly two characters, making raw binary data readable to humans. A hex editor like HxD is a direct window into the raw bytes of any file.

**The security context:**

As a forensic investigator, reading file data at the byte level and reconstructing it is exactly the job. You identify magic bytes to determine what a file actually is, and you correct or recover whatever needs fixing in order to read the original data -- whether it came from a memory dump, a damaged drive, or a deliberately corrupted file. The official terms for what this challenge taught are file signature analysis (identifying file types by their magic bytes rather than their extension) and file carving (extracting and reconstructing files based on their content and structure rather than file system metadata).

From a red team perspective, corrupting or stripping magic bytes is a basic technique to make malicious files harder to identify and analyze. From a blue team perspective, any file that doesn't match its declared extension or has an unrecognized signature is worth examining at the byte level -- the extension is just a label, the bytes are the truth.

---

*[← Back to Forensics](README.md)*
