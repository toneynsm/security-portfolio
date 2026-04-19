# Flag in Flame

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

The challenge presented a suspiciously large log file that had been replaced with a massive block of encoded text. Ran it through CyberChef and it decoded into a hex string embedded in an image. Used Google Lens to extract the hex text from the image without typing it manually, threw it back into CyberChef with From Hex, and got the flag.

**Flag:** `picoCTF{forensics_analysis_is_amazing_24d16895}`

**What I learned:** More reps identifying hidden data across multiple encoding layers, and better pattern recognition for hex. This also put log poisoning into perspective in a way that made SIEMs click.

**Log poisoning** is when an attacker deliberately writes malicious or obfuscated content into log files -- either to hide their activity, bury evidence in noise, or in more advanced cases, to get a server to execute code that ends up in its own logs. Replacing a log file with encoded content is an anti-forensics move -- anti-forensics meaning anything an attacker does to make an investigator's job harder.

**Why SIEMs matter here:**

Logs are the primary evidence trail in any incident investigation. If an attacker can tamper with them on the compromised machine, they corrupt the record and make attribution significantly harder. This is exactly why SIEMs are critical -- they ingest logs in real time and ship them off to a completely separate system. By the time an attacker tries to cover their tracks, the evidence is already somewhere they can't reach. The log on the machine can be destroyed. The copy in the SIEM can't.

---

*[← Back to Forensics](README.md)*
