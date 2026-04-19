# Scan Surprise

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

SSHd into the challenge instance, ran `ls` to list the files in the directory, and found `flag.png`. Opened it and it was a QR code -- a barcode that encodes data visually. Scanned it with my phone and the flag appeared as a title in the browser.

**Flag:** `picoCTF{p33k_@_b00_19eccd10}`

**What I learned:** Data can be encoded visually, not just as text or binary. QR codes are a real-world data encoding method -- they map patterns of black and white squares to characters. From a forensics perspective, an image file is not always just an image. It can be a data carrier. In this case the data was the flag, but in the real world QR codes have been used in phishing attacks -- attackers embed malicious URLs in QR codes knowing that most email security tools scan text links but not images. A user scans it with their phone, which has weaker security controls than a corporate laptop, and gets sent somewhere malicious. This technique is called QRLjacking or quishing (QR phishing).

---

*[← Back to Forensics](README.md)*
