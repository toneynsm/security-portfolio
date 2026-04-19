# Information

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded a JPG of a cat. Right clicked and checked properties -- nothing useful. Ran ExifTool, spotted a base64 string in the metadata, threw it into CyberChef, got the flag.

**Flag:** `picoCTF{the_m3tadata_1s_modified}`

**Takeaway:** More reps on the same core skill -- metadata is never just metadata. ExifTool first, always. At this point it's reflex.

---

*[← Back to Forensics](README.md)*
