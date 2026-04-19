# CanYouSee

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded the image from the challenge. First thing I checked was the file path itself to see if anything was encoded in it -- nothing useful there. Then I started thinking about what else an image could be hiding beyond what you can see. Googled around and landed on metadata as the next logical place to look. Uploaded the file to metadata2go.com, which pulls all the hidden metadata fields out of an image, and one field stood out immediately: `attribution_url` contained what looked like a base64 string -- `cGljb0NURntNRTc0RDQ3QV9ISUREM05fNGRhYmRkY2J9Cg==`. Threw it into CyberChef, ran From Base64, and got the flag.

**Flag:** `picoCTF{ME74D47A_HIDD3N_4dabddcb}`

**What I learned:** Metadata is data about data -- it's information baked into a file that describes it without being part of the visible content. Images carry a lot of it: the device that took the photo, GPS coordinates, software used to edit it, timestamps, author fields, and more. None of that shows up when you just open the image, but it's sitting there in the file.

**Why it matters:**

From a red team perspective, metadata is an attack surface in two directions. Going out -- an attacker who already has access to a system can encode stolen data into image metadata and exfiltrate it without raising alarms. Firewalls and DLP tools (software that monitors for sensitive data leaving a network) are looking for obvious things like large transfers or suspicious domains. An image with a base64 string buried in an attribution field looks completely innocent to automated detection.

Going in -- images that organizations post publicly often have metadata that was never meant to be public. GPS coordinates revealing physical locations, internal file paths exposing naming conventions, software version info, usernames that might match internal accounts. A pentester doing recon on a target can pull all of that from images on a company's public website without ever touching their systems.

From a blue team perspective the fix is simple: strip metadata from any file before it goes public. Most organizations have no process for this, which means their public-facing images are quietly leaking internal information to anyone who thinks to look.

---

*[← Back to Forensics](README.md)*
