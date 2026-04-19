# Binary Digits

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

The challenge handed me a .bin file. Opened it in a text editor and it was a wall of 1s and 0s -- classic binary encoding. Took it to CyberChef, dropped the data into the input, used the "From Binary" operation, and it decoded straight to the flag.

**Flag:** `picoCTF{h1dd3n_1n_th3_b1n4ry_2607862b}`

**What I learned:** Binary is just base-2 -- everything a computer stores is ultimately 1s and 0s, and those patterns map directly to text, images, executables, anything. Being able to recognize common encodings (binary, hex, base64) and know how to convert between them is a core forensics skill because this is exactly how attackers obfuscate data.

**Why it matters:**

From a red team perspective, encoding is a go-to obfuscation technique. Binary, hex, and base64 are not encryption -- they do not protect data, they just make it less immediately readable to a human glancing at it. Malware authors use encoding to hide payloads inside files that look benign, sneak past signature-based detection, and make static analysis harder at first glance. A real-world example of this is steganography, where data gets embedded inside image or audio files in a way that does not visibly alter them -- the hidden content is only recoverable if you know to look and know what tool to use.

From a blue team and forensics perspective, recognizing encoding on sight is the first step. If you pull a file off a compromised machine and the contents look like a structured pattern of characters -- long strings of 1s and 0s, blocks of hex, base64 with its telltale `==` padding at the end -- that is a signal worth following. CyberChef is the standard tool for this because it chains operations, so if something is encoded multiple times (which is common in real malware) you can stack the decode steps and work through it layer by layer.

---

*[← Back to Forensics](README.md)*
