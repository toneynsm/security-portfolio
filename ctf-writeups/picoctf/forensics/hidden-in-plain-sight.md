# Hidden in Plain Sight

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded the JPG and decided it was time to retire metadata2go and move to a proper industry standard tool. Researched options, landed on ExifTool, figured out how to get it running in PowerShell via environment variables, and ran it against the image. The comment field had two base64 strings baked into it -- the first decoded to `steghide: cEF6endvcmQ=`, and that second string decoded to `pAzzword`.

Steghide is a CLI tool that hides data inside image files and locks it with a passphrase -- that's called steganography. The metadata just handed me both the tool and the password. Researched how to get steghide running on Windows, downloaded it, moved the binaries to C:\Tools, added it to PATH via system environment variables, and confirmed it was working in the CLI. Looked up the syntax, ran `steghide extract -sf filename.jpg`, entered `pAzzword` when prompted, and extracted the flag.

**Flag:** `picoCTF{h1dd3n_1n_1m4g3_1c55ccd0}`

**What I learned:** Hands-on steganalysis -- the process of detecting and extracting data hidden inside files using steganography. Steghide hides a payload inside an image in a way that doesn't visibly alter it, and protects it with a passphrase. Without the password you can't extract it.

**Why it matters:**

This is the full attack chain from a blue team perspective. The attacker embedded a hidden payload inside a normal-looking JPG and protected it with a password -- two layers of concealment. A casual investigator opening the image sees nothing. The first layer (steganography) hides that anything is there at all. The second layer (the passphrase) protects the contents even if someone suspects something is hidden.

The mistake was leaving the password in the metadata of the same file. In a real operation an attacker would never do that -- the passphrase would be communicated through a separate channel entirely. But the technique itself is real and used in the wild. For a defender, the investigative workflow is exactly what this challenge taught: extract metadata first, recognize encoded strings, decode them, then use what you find to go deeper. ExifTool is now a permanent part of that toolkit.

---

*[← Back to Forensics](README.md)*
