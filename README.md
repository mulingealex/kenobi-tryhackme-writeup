##  Overview
This project documents a full penetration testing walkthrough of the **TryHackMe Kenobi room**.  
The objective was to simulate a real-world attack by identifying vulnerabilities, gaining access, and escalating privileges.

##  Objectives
- Perform reconnaissance and enumeration
- Identify vulnerable services
- Gain initial access
- Escalate privileges to root
- Capture user and root flags
## Tools Used
- Nmap
- Netcat
- SMB tools
- FTP
- Exploit scripts
- Linux commands
## 1. Reconnaissance & Scanning

### Command Used:nmap -sC -sV -oN nmap_initial.txt <TARGET_IP>
![Nmap Scan](screenshots/kenobi%20nmap%20results.png)
### Key Findings:
- **FTP (21)** → ProFTPD
- **SMB (139, 445)**
- **NFS (2049)**
This tells us multiple attack surfaces exist.
## 2. SMB Enumeration

### Command:
smbclient -L  //<TARGET_IP> -N

![SMB Enumeration](screenshots/smbclient%20enumeration.png)

- Found **anonymous access enabled**
- Discovered shared directories

This is a misconfiguration → potential data exposure.
Used my machine to access the machine's  network share
command used: smbclient //target ip/anonymous
![SMB Anonymous Login](screenshots/smbclient%20anonymous%20login.png)
- Enumerated SMB shares 
- Connected to `anonymous` 
- Found a file: `log.txt` 
The log file was filtered using `grep` to extract relevant service information, revealing that the target was running ProFTPD 1.3.5 on port 21.
Command used: grep -Ei "ftp|proftpd|open" log.txt

![Filtered Search](screenshots/filtered%20search.png)

## 3. NFS Enumeration

Command: showmount -e <TARGET_IP>
Revealed exported directory
![NFS Showmount Output](screenshots/showmount%20output.png)

This means we can mount remote files locally
## 4. Mounting the Share
This is done to access files in /VAR remotely

![Mounting NFS Share](screenshots/mounting%20shared%20file.png)
## 5. Exploit ProFTPD (mod_copy)
-You **can’t access** `/home/kenobi/.ssh/id_rsa` directly 
 But FTP vulnerability lets you **copy it into `/var/tmp`** 
-/var is already mounted
Connect to ftp using netcat and copy the key
![ProFTPD Exploit](screenshots/proftpd%20exploit.png)
Checking the mounted folder the rsa key has been copied successfully
![SSH Private Key Retrieved](screenshots/rsa%20key.png)

## 6. Copying  the key
copied the key to current directory, changed permission and accessed a shell for initial access
![Copying SSH Key Locally](screenshots/grabbing%20the%20key.png)
After exploiting the ProFTPD `mod_copy` vulnerability, the SSH private key (`id_rsa`) was successfully copied into `/var/tmp`, a location accessible via the mounted NFS share.
###  Retrieving the key
The key was copied from the mounted directory to the local machine:

```
cp kenobi/mount/tmp/id_rsa .
```

This moves the private key into the current working directory for local use.

###  Setting correct permissions

```
chmod 600 id_rsa
```
- `600` means:
    - Owner → read & write
    - Others → no access

 SSH **refuses to use private keys** that are accessible by other users for security reasons.  
If permissions are too open, you’ll get an error
## Gaining Initial Access

```
ssh -i id_rsa kenobi@<TARGET_IP>
```
### Result:
- Logged in successfully
## 7. privilege escalation
After gaining access as `kenobi`, we search for binaries with the **SUID bit set**:
find / -perm -u=s -type f 2>/dev/null
![SUID Binary Enumeration](screenshots/suspicious%20binary.png)
After running the binary
![SUID Binary Options](screenshots/binary%20options.png)
After gaining access as the `kenobi` user, a SUID binary `/usr/bin/menu` was identified. Analysis revealed it executes system commands like `ifconfig` **without using absolute paths**, making it vulnerable to **PATH hijacking**.

![Privilege Escalation via PATH Hijacking](screenshots/privilege%20escalation.png)
Selecting the option that runs `ifconfig` caused the binary to execute the malicious script, spawning a root shell.

## Conclusion
> This assessment demonstrated a full attack chain beginning with service enumeration, followed by exploitation of a vulnerable ProFTPD service to obtain SSH credentials, and finally privilege escalation via a misconfigured SUID binary. The system was successfully compromised, resulting in root access.
> 
## Key Vulnerabilities Identified
- Anonymous SMB share exposing sensitive logs
- Outdated **ProFTPD 1.3.5** vulnerable to `mod_copy` exploit
- Sensitive SSH private key stored on the system
- NFS share exposing `/var` directory
- SUID binary (`/usr/bin/menu`) vulnerable to PATH hijacking

##  Recommendations 
- Disable anonymous SMB access or restrict permissions
- Update or patch ProFTPD to a secure version
- Restrict access to sensitive files like SSH keys
- Limit NFS exports and enforce proper access controls
- Avoid using relative paths in SUID binaries (use absolute paths)
- Implement proper logging and monitoring
