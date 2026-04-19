# PicoCTF

Carnegie Mellon University's beginner-to-intermediate Capture The Flag platform. Challenges span a wide range of categories including web exploitation, forensics, cryptography, reverse engineering, and binary exploitation.

---

## Categories

| Category | Challenges | Status |
|----------|-----------|--------|
| [Web Exploitation](web-exploitation/) | 17 | All solved |
| [Forensics](forensics/) | 17 | All solved |

---

## Web Exploitation Challenges

| Challenge | Vulnerability |
|-----------|--------------|
| [Inspect HTML](web-exploitation/inspect-html.md) | Sensitive data in HTML comments |
| [Intro to Burp](web-exploitation/intro-to-burp.md) | Content-Type bypass / improper input validation |
| [Old Sessions](web-exploitation/old-sessions.md) | Session hijacking / improper session invalidation |
| [SSTI](web-exploitation/ssti.md) | Server-Side Template Injection (Jinja2 → RCE) |
| [WebDecode](web-exploitation/webdecode.md) | Sensitive data encoded in source files |
| [Head Dump](web-exploitation/head-dump.md) | Exposed heap dump endpoint |
| [Local Authority](web-exploitation/local-authority.md) | Client-side auth with hardcoded credentials |
| [Cookie Monster Secret Recipe](web-exploitation/cookie-monster-secret-recipe.md) | Sensitive data in unprotected cookie |
| [Unminify](web-exploitation/unminify.md) | Secrets exposed in minified client-side code |
| [Crack the Gate 1](web-exploitation/crack-the-gate-1.md) | ROT13-encoded backdoor header in HTML comments |
| [Bookmarklet](web-exploitation/bookmarklet.md) | Client-side script execution via DevTools |
| [Cookies](web-exploitation/cookies.md) | IDOR via sequential cookie integer |
| [Insp3ct0r](web-exploitation/insp3ct0r.md) | Flag split across HTML, CSS, and JS source files |
| [Where Are The Robots](web-exploitation/where-are-the-robots.md) | Sensitive path exposed in robots.txt |
| [GET aHEAD](web-exploitation/get-ahead.md) | HTTP verb tampering via HEAD method |
| [Includes](web-exploitation/includes.md) | Flag split across included JS and CSS files |
| [Don't Use Client Side](web-exploitation/dont-use-client-side.md) | Client-side auth with scrambled hardcoded credentials |

---

## Forensics Challenges

| Challenge | Vulnerability |
|-----------|--------------|
| [Binary Digits](forensics/binary-digits.md) | Binary encoding / data obfuscation |
| [CanYouSee](forensics/can-you-see.md) | Sensitive data in image metadata |
| [Flag in Flame](forensics/flag-in-flame.md) | Multi-layer encoding / log tampering |
| [Disko 1](forensics/disko-1.md) | Data hidden in file slack space |
| [Hidden in Plain Sight](forensics/hidden-in-plain-sight.md) | Steganography with metadata-stored passphrase |
| [Verify](forensics/verify.md) | SHA256 file integrity verification |
| [Riddle Registry](forensics/riddle-registry.md) | Sensitive data in PDF metadata |
| [Scan Surprise](forensics/scan-surprise.md) | QR code as visual data encoding |
| [Secret of the Polyglot](forensics/secret-of-the-polyglot.md) | Polyglot file hiding data across two formats |
| [Corrupted File](forensics/corrupted-file.md) | Corrupted magic bytes / file header repair |
| [RED](forensics/red.md) | LSB steganography across image color channels |
| [Information](forensics/information.md) | Sensitive data in image metadata |
| [Glory of the Garden](forensics/glory-of-the-garden.md) | Appended data steganography / file trailer injection |
| [Redaction Gone Wrong](forensics/redaction-gone-wrong.md) | Improperly redacted PDF text |
| [Packets Primer](forensics/packets-primer.md) | Plaintext flag in PCAP payload |
| [Enhance!](forensics/enhance.md) | Hidden data in SVG source markup |
| [Operation Orchid](forensics/operation-orchid.md) | AES-encrypted file recovered via shell history |
