# Space Man
**Category:** Forensics | **Status:** DNF -- didn't solve this one in time

Downloaded `Space_Man.png` -- just a picture of a spaceship in orbit. Ran ExifTool first to check metadata and see if it was a polyglot (a file that's secretly two file types at once). Nothing suspicious in the metadata -- standard PNG, no embedded data visible.

Tried steghide next, but steghide doesn't support PNG format. Converted the PNG to JPG using an online image editor and ran steghide again with multiple passphrases based on the description hints -- "Mercury," "Apollo," "Gemini," "Gemini 8" and variations. The description hinted at "one of the most important space projects that paved the way for the Moon" and a mission that involved a "dizzying situation." Gemini 8 fits -- that mission had a serious uncontrolled spinning emergency. None of the passphrase attempts returned data though.

Tried an online steganography decoder at stylesuxx.github.io and got a hidden string: `pgfn{jm_ilawfm_zs_sw_gw_zlq_ubwt}`. Ran it through ROT brute force in CyberChef -- none of the 25 shifts produced a clean flag. Tried Vigenere decode with "gemini" and "gemini8" as the key -- no clean result.

Launched a dictionary attack against the JPG using rockyou.txt via a PowerShell loop:

```powershell
Get-Content "C:\Tools\wordlists\rockyou.txt" | ForEach-Object {
    $result = echo $_ | steghide extract -sf "C:\Users\nicks\Downloads\Space_Man.jpg" -p $_ 2>&1
    if ($result -notmatch "could not extract") {
        Write-Host "FOUND: $_"
    }
}
```

Still running when the competition closed. Ran out of time.

**What I learned:** The PowerShell loop approach works but is too slow for a live competition -- rockyou.txt has 14 million entries and steghide is slow per attempt. The right move here would have been to transfer the file to Kali immediately and run stegseek, which is what I ended up doing successfully on the Cold Wake challenge. The encoded string from the online decoder -- `pgfn{jm_ilawfm_zs_sw_gw_zlq_ubwt}` -- still hasn't been cracked. It doesn't respond to straight ROT or basic Vigenere, which suggests either a custom cipher or the online tool extracted something that wasn't actually the hidden data. Something to revisit.
