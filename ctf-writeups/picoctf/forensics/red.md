# RED

| Field | Details |
|-------|---------|
| Platform | PicoCTF |
| Category | Forensics |
| Status | Solved |

---

Downloaded the image -- description just said "RED" four times, not much to go on. Pulled the metadata with ExifTool and found a poem embedded in it. Read the first letter of each line: C, H, E, C, K, L, S, B. That spells CHECKLSB -- the hint was an acrostic pointing directly at the technique.

Did research into color channels, LSB steganography, and the right tool for the job -- landed on StegOnline. Uploaded the image and browsed the bit planes across all channels. Cycling through bit levels 1 through 7 showed pure color in every plane, but bit 0 across all channels showed a barcode-looking pattern -- structured data, not noise. That confirmed something was hidden at the LSB level. Checked bit 0 for R, G, B, and A simultaneously with the LSB option enabled, extracted the data, and got a series of base64 strings. Threw them into CyberChef and got the flag.

**Flag:** `picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}`

**What I learned:** Every pixel in an image has four channel values -- Red, Green, Blue, and Alpha (transparency). Each value is stored as a number from 0 to 255, which in binary is 8 bits. Those 8 bits are not all equally important. The leftmost bit contributes most of the value. The rightmost bit -- the least significant bit or LSB -- contributes almost nothing. Changing it shifts a color value by 1 out of 255, which is completely invisible to the human eye.

LSB steganography works by converting a secret message to binary and replacing the throwaway LSB of each channel value across every pixel with the next bit of the message. Spread across thousands of pixels, you can hide significant amounts of data with zero visible change to the image. StegOnline extracts those bits, reassembles them into bytes, and surfaces whatever was hidden.

**Why it matters:**

This is a real attack vector. Threat actors have used LSB steganography to hide command and control instructions inside innocent-looking images posted publicly on social media. Malware on a compromised machine downloads the image, extracts the LSBs, reads its next instructions, and executes them. From a network monitoring perspective it looks like a normal image download -- nothing suspicious. The payload is invisible without the right forensic tools and the knowledge to look for it.

From a blue team perspective, detecting LSB steganography requires exactly what this challenge taught -- knowing that structured patterns in bit planes are a signal, knowing what tools to use, and knowing what to look for in the output. A barcode pattern in a bit plane where you'd expect pure color is the tell.

---

*[← Back to Forensics](README.md)*
