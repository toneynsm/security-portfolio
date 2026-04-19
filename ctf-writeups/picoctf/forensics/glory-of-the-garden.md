# Glory of the Garden

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

The hint asked "what is a hex editor?" -- pretty direct. Opened the image in HxD, a program that lets you read file data at the byte level, scrolled to the very bottom of the hex dump, and the flag was sitting right there in the decoded text column at the end of the file.

**Flag:** `picoCTF{more_than_m33ts_the_3y395e12915}`

**What I learned:** This technique is called file trailer injection or appended data steganography. The flag was raw text tacked onto the end of the file after the actual image data ends. Image viewers stop reading at the end of the image and ignore everything after it, so the image opens and displays normally. But the bytes are still physically there in the file, readable by anything that looks at the raw contents.

This is actually easier to detect than traditional LSB steganography because nothing in the image itself was modified -- data was just added to the end. The addition vs. modification distinction matters: LSB steganography changes existing pixel values in ways that are invisible, making it harder to spot. Appended data leaves the original file completely intact and just tacks extra bytes onto the tail, which tools like binwalk or HxD catch immediately.

In the real world malware has been distributed this way -- an image with an executable payload appended after the image data ends. The file looks and opens like a normal image, passes a casual inspection, but something reads the raw bytes, finds the payload, and executes it. As an analyst, running binwalk against any suspicious file is standard procedure specifically because it surfaces appended data and embedded files that wouldn't show up any other way.

---

*[← Back to Forensics](README.md)*
