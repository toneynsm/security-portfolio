# Fix Your Ship
**Category:** Forensics | **Status:** Solved

This was extremely fun. Given three corrupted files and told the first one is a PNG. Opened each file in HxD (a hex editor -- a tool that lets you read and edit the raw binary data of any file) to inspect the magic bytes (the first few bytes of a file that identify what file type it actually is, regardless of its extension).

**File 1 (PNG):**
Corrupted header: `89 00 00 00 0D 0A 1A 0A`
Correct PNG header: `89 50 4E 47 0D 0A 1A 0A`

![HxD showing file1 with the corrupted PNG header before the fix](../assets/FYS_1.png)

![HxD showing file1 after fixing the four corrupted bytes -- header now reads 89 50 4E 47](../assets/FYS_2.png)

Fixed the four corrupted bytes, saved with a `.png` extension, and opened it. The image showed ship schematics and revealed the first piece of the flag -- `jctf{starship` -- along with a hint that the second file starts with `FF` and ends with `D9`.

![Ship schematics PNG opened -- first flag piece visible, hint for file2 in the caption](../assets/FYS_3.png)

**File 2 (JPEG):**
`FF` and `D9` are the start and end markers of a JPEG file. Corrupted header read: `00 00 00 E0 00 10 4A 46`.

![HxD showing file2 with the corrupted JPEG header before the fix](../assets/FYS_4.png)

Fixed it to `FF D8 FF E0` and changed the last two bytes to `FF D9`.

![HxD showing file2 after fixing the header -- JPEG magic bytes now correct, with research confirming the format](../assets/FYS_5.png)

Saved with a `.jpg` extension and opened it. Second flag piece: `_voyager_`

![FILE TYPE BOX JPEG opened -- second flag piece visible](../assets/FYS_6.png)

**File 3 (MP4):**
The schematics image had a label reading "FILE TYPE BOX" -- a hint pointing at the third file's type. Opened it in HxD and found the magic bytes scrambled: `00 00 00 20 70 66 79 74`.

![HxD showing file3 with the scrambled ftyp box bytes](../assets/FYS_7.png)

Did some quick googling and confirmed `70 66 79 74` as a scrambled version of `66 74 79 70` -- the `ftyp` box that identifies MP4/MOV container files.

![Google confirming the hex signature points to an MP4/MOV container](../assets/FYS_8.png)

Fixed the byte order, saved with a `.mp4` extension, and opened it. The video contained the final flag piece: `apollo}`

![Windows Media Player showing the repaired MP4 -- final flag piece visible](../assets/FYS_9.png)

**Flag:** `jctf{starship_voyager_apollo}`

**What I learned:** Magic bytes are how operating systems and forensic tools actually identify file types -- not the extension. The extension is just a label anyone can change. The magic bytes are baked into the file's binary data at a specific offset and follow a standard for each format. A corrupted or misidentified file can often be repaired just by fixing those first few bytes in a hex editor. This is a fundamental forensics skill -- file carving (recovering files from raw data) works on the same principle.

**Blue team takeaway:**

File type validation should never rely on extensions alone. Any system that accepts file uploads should read the magic bytes and validate the actual file type before processing. An attacker who renames a PHP shell to `image.png` gets stopped by magic byte checking -- but not by extension checking alone.
