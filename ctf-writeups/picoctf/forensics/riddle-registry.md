# Riddle Registry

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded a PDF from the challenge. Same instinct as before -- don't take the file at face value, check what's hiding in the metadata. Uploaded it to metadata2go.com, found a base64 string sitting in the author field, decoded it in CyberChef, and got the flag.

**Flag:** `picoCTF{puzzl3d_m3tadata_f0und!_42440c7d}`

**What I learned:** Same core concept as CanYouSee but the file type matters here. PDFs carry their own metadata separate from images -- author, creator software, creation date, modification date, company name. That's a different set of fields than an image's EXIF data, but the same principle applies: information is baked into the file that never shows up when you just open it.

**Why it matters:**

PDFs are one of the most commonly shared file types in professional environments -- proposals, invoices, contracts, reports. Organizations send them externally constantly and almost nobody strips the metadata before hitting send. From a red team perspective that's a reliable recon source. An author field might reveal an employee's full name and username. The creator software field might expose what version of Adobe or Microsoft Office the organization is running, which maps directly to known CVEs. A modification timestamp might reveal internal workflow details.

From a blue team perspective the same fix applies as with images -- strip metadata before anything goes out the door. Tools like ExifTool can do this in bulk from the command line. DLP tools can be configured to inspect outbound files for metadata before they leave the network, though that level of coverage is expensive and not common outside of larger organizations. For smaller teams, a simple policy and a one-line ExifTool command in the workflow is better than nothing.

---

*[← Back to Forensics](README.md)*
