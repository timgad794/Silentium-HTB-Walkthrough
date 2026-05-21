# 🎯 Silentium – Hack The Box Walkthrough

**Difficulty:** Easy  
**OS:** Linux  
**Player #:** 7,008  
**Date Completed:** May 2026

---

## 📋 Machine Summary

Silentium is an Easy Linux machine that requires exploiting multiple vulnerabilities:
- Flowise AI (CVE-2025-58434) – Password Reset Token Leak
- Flowise RCE via customMCP endpoint
- Gogs (CVE-2025-8110) – Arbitrary File Write via Symlink

The attack chain goes from **web application → Docker container → Host → Root**.

---

## 🔍 Enumeration

```bash
# Add domain to /etc/hosts
echo "10.129.1.255 silentium.htb" >> /etc/hosts

# Subdomain fuzzing
gobuster vhost -u http://silentium.htb -w /usr/share/wordlists/dirb/common.txt --append-domain

# Found: staging.silentium.htb (Flowise AI v3.0.5)
