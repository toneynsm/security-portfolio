# Redaction Gone Wrong

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded the PDF and opened it. Multiple lines were blacked out -- looked like someone had used a paint tool to draw black boxes over sensitive text. Ran ExifTool first out of habit, nothing useful in the metadata. Then remembered from graphic design work that drawing a shape over text in a document doesn't remove the text -- it just covers it visually. Googled to confirm: proper redaction requires the underlying data to be permanently deleted, not painted over. Selected all the text in the PDF, copied it, pasted it into a browser field, and read everything including the text hiding under the black boxes. Flag was in there.

**Flag:** `picoCTF{C4n_Y0u_S33_m3_fully}`

**What I learned:** This is a real and well-documented failure that has happened at actual government agencies and law firms. Drawing a black box over text in a PDF, Word document, or image does not remove the text -- it only obscures it visually. The underlying data is still in the file and trivially recoverable by anyone who selects and copies it. True redaction requires the text to be permanently deleted at the data level before the document is exported or distributed.

**Why it matters:**

From a red team and OSINT (Open Source Intelligence -- gathering information from publicly available sources) perspective, improperly redacted documents are a real source of sensitive information. Court filings, government reports, and legal documents have all leaked sensitive data this way -- names, addresses, classified details, and personal information that was supposed to be hidden. A researcher or attacker who downloads a public PDF and copies the "redacted" sections can recover whatever was covered.

From a blue team perspective the fix is straightforward but often overlooked: use dedicated redaction tools that permanently remove data rather than covering it, and verify redaction by attempting to copy the text before distributing any document. Adobe Acrobat has a built-in redaction tool that actually deletes the underlying content. Paint and highlight functions do not. The broader lesson is the same one this whole category keeps reinforcing -- never assume something is hidden just because you can't see it.

---

*[← Back to Forensics](README.md)*
