# Enhance!

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded the SVG file -- opened in the browser as a plain white page with a black dot in the center. Ran ExifTool first, nothing suspicious in the metadata. Remembered from web exploit experience that inspecting page source often reveals hidden info, and since SVGs open in the browser like a webpage, figured it was worth a shot. Opened the source, expanded everything, and immediately noticed the flag broken up into substrings at the bottom. Put them together and got the flag.

**Flag:** `picoCTF{3nh4nc3d_aab729dd}`

**What I learned:** SVG files are not simple image files -- they're XML-based code that browsers render as images. Unlike a JPEG or PNG which is pure image data, an SVG can contain text, hidden elements, embedded files, and even executable JavaScript that never visually renders. Something can look like a blank white page with a dot and be hiding data in its source that's completely invisible to anyone just viewing it.

**How this benefits an attacker:**

An attacker can embed a payload, credentials, or instructions inside an SVG's markup that are invisible when opened normally. More importantly, SVGs can contain executable JavaScript. A malicious SVG opened in a browser executes that script automatically -- no interaction beyond opening the file. If a site allows SVG uploads without sanitizing them, an attacker uploads it once and the script runs for every user who views it. That's stored XSS (Cross Site Scripting -- injecting malicious code into a page that executes in someone else's browser).

SVGs can also contain external references -- tags that automatically reach out to a URL when the file is opened. The victim's machine calls the attacker's server just by rendering the file, logging their IP and potentially receiving a malicious payload in return. Execution happens at the point of rendering, which means whatever opens the SVG runs whatever is inside it.

**Blue team takeaway:**

SVGs should never be trusted as simple image files. Any SVG found on a compromised machine or in a suspicious email attachment should be opened in a text editor first -- read the source, don't render it. Organizations that allow SVG uploads need to sanitize them before serving them to other users, stripping script tags and external references. From a forensics perspective, source inspection is always on the checklist for any file that renders in a browser.

---

*[← Back to Forensics](README.md)*
