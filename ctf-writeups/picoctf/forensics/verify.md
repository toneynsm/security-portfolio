# Verify

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

SSHd into the challenge instance and found a directory full of files -- a lot of them. The goal was to find and decrypt the right one using a provided SHA256 checksum. A SHA256 checksum is essentially a fingerprint of a file -- run the algorithm against any file and it spits out a unique string. Same file, same hash, every time. Change even one byte and the hash is completely different.

The hint pointed me to `sha256sum <directory>/*` which generates checksums for every file in a directory at once. Ran that against the whole directory, then piped it through `grep` to search the output for the checksum I was given. The matching file revealed itself immediately. Ran the provided decrypt tool against it and got the flag.

**Flag:** `picoCTF{trust_but_verify_...}`

**What I learned:** SHA256 is a one-way hashing algorithm -- it turns any file into a fixed-length string that uniquely represents its contents. It's not encryption, nothing is being hidden, it's purely verification. The point is that two different files cannot produce the same SHA256 hash, so if the hashes match, the files are identical. This is how you confirm a file hasn't been tampered with.

**Why it matters:**

From a blue team perspective, file integrity monitoring is a core detection technique. The concept is straightforward -- take SHA256 checksums of every critical file on a system (OS files, configs, executables) and store those hashes somewhere safe. Run the same checksums periodically and compare. If anything changed, the hash won't match and you know something was touched. Tools like Tripwire and AIDE automate this entirely and can alert in real time if a critical file is modified. In an incident investigation, one of the first questions is whether the attacker modified system files to maintain persistence or cover their tracks. Hashes answer that question definitively.

From a red team perspective, attackers know defenders use hashes so they try to work around them. A common technique is timestomping -- modifying a file's metadata to make it look untouched. But hashes don't care about timestamps, they care about content. The only real counter is a hash collision -- producing a different file that generates the same hash. With SHA256 that's computationally infeasible right now, which is exactly why it replaced older algorithms like MD5, which has known and documented collision vulnerabilities.

The real world connection is software verification. Every legitimate software release publishes a SHA256 checksum alongside the download. You're supposed to verify your download matches before running it. Most people skip this. That gap is exactly how supply chain attacks work -- an attacker compromises a download server, swaps the legitimate installer for a malicious one, and anyone who downloads without verifying gets infected. The SolarWinds attack is the most well-known recent example of this at enterprise scale.

---

*[← Back to Forensics](README.md)*
