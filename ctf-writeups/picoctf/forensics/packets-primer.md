# Packets Primer

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Opened the packet capture in Wireshark, scrolled through the packets while watching the hex data window, and the flag was sitting in plain text in the payload of the 4th packet. Typed it manually and moved on.

**Flag:** `picoCTF{p4ck37_5h4rk_ceccaa7f}`

**What I learned:** More reps at packet analysis and navigating Wireshark. Simple challenge but the real world mapping is what made it worth thinking about.

**Why it matters:**

In a real SOC you'd never open a PCAP cold and start scrolling. You'd already have context -- an alert from your SIEM or EDR flagging suspicious behavior, with a source IP, destination IP, timestamp, and port number attached. Those become your Wireshark filters. You narrow thousands of packets down to the relevant conversation, follow the TCP stream to reconstruct what was actually sent, and make the human judgment call on whether it's malicious.

As an analyst you're looking for specific things: data being sent out that shouldn't be, commands being received from an external server, beaconing (a machine calling home to the same IP on a regular interval -- a classic malware behavior), or protocols being abused in unexpected ways like DNS tunneling, where data is hidden inside DNS queries to sneak past firewalls.

More often than not in a real investigation the payload won't be in plaintext like it was here. Attackers encrypt their traffic specifically to prevent this kind of analysis. That's where EDR comes in -- the endpoint agent on the compromised machine has visibility into what processes were running, what files were touched, and what data was accessed, giving you the context the encrypted packet alone can't provide. Network traffic and endpoint telemetry together tell the full story.

---

*[← Back to Forensics](README.md)*
