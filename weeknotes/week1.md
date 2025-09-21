# Week 1 — Lab Setup & Evidence
_Last Updated: Sep 20, 2025_

---

### Quick journal
This week I focused on getting my lab environment set up and verified so I can start hands-on recon and WebApp labs next week. Tools installed, WSL/Kali configured, Burp + browsers proxied, and a repo structure created to collect evidence and writeups.

---

## 1) Overview
- **Host OS:** Windows 11 (version: 24H2)  
- **WSL Distro:** Kali Linux (default)  
- **Primary use:** web app recon/exploitation practice, safe lab work, HackerOne/Bugcrowd triage practice

---

## 2) Inventory (Tools installed)
- **Browsers:** Chrome (testing profile)  
- **Chrome extensions:** FoxyProxy, Wappalyzer  
- **Proxy:** Burp Suite Community (v2025.8.4)  
- **API / HTTP client:** Postman (v11.63.6)  
- **WSL:** Kali Linux (installed packages listed below)  
- **Recon & fuzzing:** ffuf, gobuster  
- **Port scanner:** nmap  
- **Utilities:** curl, jq, git, unzip, wget, python3-pip  
- **Optional / nice-to-have:** OWASP ZAP, sqlmap, nuclei, masscan, amass, certutil/certifi tools

---

## 3) Installation & config snippets

### On Windows (PowerShell - Admin)
```powershell
# Install WSL and Kali distro
wsl --install -d kali-linux

# Ensure WSL2 default
wsl --set-default-version 2

# List installed distro(s)
wsl -l -v
```

### In WSL (Kali)
```bash
# 1) Update + upgrade
sudo apt update && sudo apt upgrade -y

# 2) Essentials
sudo apt install -y build-essential git curl wget unzip jq python3-pip

# 3) Core recon & discovery
sudo apt install -y nmap masscan gobuster amass ffuf

# 4) Template-based scanners
sudo apt install -y nuclei

# 5) Optional extras
sudo apt install -y sqlmap hydra

# 6) Kali meta (optional; large)
sudo apt install -y kali-linux-default

# 7) Cleanup
sudo apt autoremove -y && sudo apt autoclean
```

### Burp / Browser / Proxy
- Configure browser proxy to `127.0.0.1:8080` or use FoxyProxy profile.  
- Export Burp CA certificate and import into OS/browser certificate store so browsers trust Burp (avoids SSL warnings).  
- In Postman, set proxy to system or explicit `127.0.0.1:8080` for testing.

---

## 4) Verification checklist
- [ ] Intercept HTTP(S) in Burp from Chrome with FoxyProxy enabled.  
- [ ] Burp CA certificate trusted (no SSL warnings).  
- [ ] `nmap` scan runs and returns open ports on a lab host.  
- [ ] `ffuf` / `gobuster` enumerate a test path on a lab site.  
- [ ] Postman request can be proxied and seen in Burp.

---

## 5) Quick test commands & examples

**nmap**
```bash
nmap -sS -sV -p- -T4 <target-ip>
```

**ffuf (dir fuzz)**
```bash
ffuf -u https://target/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

**curl via Burp**
```bash
curl -x http://127.0.0.1:8080 -k https://example.com
```

---

## 6) Lab evidence & repo layout
Raw lab notes go into `lab-evidence/`. Once cleaned/redacted, a polished summary goes into `/labs` for sharing.

Keep a `lab-evidence/` folder (or mirror in Drive) with this structure per lab:
```
lab-evidence/
  lab-name/
    lab-name.md         # short notes & summary
    burp-requests/      # exported .xml/.txt requests
    screens/            # screenshots (annotated when useful)
    report-drafts/      # bug report drafts / PoC
```

---

## 7) Bug report template
```
Title:
Summary:
Product / URL:
Impact:
Steps to reproduce:
Proof-of-concept:
Mitigation:
Notes:
```

---

## 8) Troubleshooting

### Quick checklist (start here)
- Confirm Burp is running and Proxy → Options shows an active listener on the expected port (default `8080`).
- Confirm browser proxy is set to `127.0.0.1:8080` or FoxyProxy profile is enabled.
- Confirm Burp CA certificate is imported/trusted in the browser or OS cert store.
- If using WSL2, remember localhost mapping can differ — test using Windows host IP if `127.0.0.1` from WSL fails.
- Check Windows Firewall / Defender rules — allow Burp or test with firewall briefly disabled (be careful).

---

### 8.1) Windows: is Burp listening?
From an elevated Windows Command Prompt (cmd) or PowerShell:

- Classic netstat to show PID:
```
netstat -ano | findstr :8080
Look for a line like:
TCP    0.0.0.0:8080    0.0.0.0:0    LISTENING    12345
The final column is the PID.
```

- Find process by PID (cmd):
```
tasklist /FI "PID eq 12345"
```

- PowerShell (clean view of connection + owning process):
```
Get-NetTCPConnection -LocalPort 8080 | Select-Object LocalAddress,LocalPort,State,OwningProcess
```
Then:
```
Get-Process -Id (Get-NetTCPConnection -LocalPort 8080).OwningProcess
```

- GUI alternative (nice): run Sysinternals TCPView to see live ports and owning processes.

If netstat shows nothing listening:
- Make sure Burp is open and Proxy → Options has an active listener on the port you expect.
- Confirm Burp isn’t bound only to a specific interface (e.g., a non-loopback IP) — check the listener settings.

---

### 8.2) Confirm Burp process / PID
- Use Task Manager to search for java or Burp (depending on start method).
- Or:
```
tasklist | findstr java
```
If multiple Java processes exist, compare PIDs to the netstat PID.

---

### 8.3) Quick Windows-side connection test
From Windows (cmd / PowerShell):
- Simple curl via the proxy:
```
curl -I -x http://127.0.0.1:8080 -k https://example.com
```
If this returns headers, Burp is accepting connections from Windows.

---

### 8.4) WSL → Windows connectivity (WSL2 notes)
WSL2 uses a lightweight VM and localhost inside WSL may not map to Windows localhost in the same way. If WSL curl -x http://127.0.0.1:8080 fails, do this:

- From WSL, discover the Windows host IP used for DNS resolution:
```
grep nameserver /etc/resolv.conf
```
Example output: nameserver 172.22.112.1 — that IP is usually the Windows host.

- Test from WSL using that IP:
```
curl -I -x http://172.22.112.1:8080 -k https://example.com
```
(Replace the IP with the one from /etc/resolv.conf and 8080 with your Burp port.)

If that succeeds, WSL → Windows TCP routing is fine and you should use the host IP in your proxy env vars inside WSL (or set NO_PROXY appropriately).

---

### 8.5) WSL-side checks (if you run Burp from Windows and test from WSL)
- If nc (netcat) is installed in WSL, test basic TCP connect:
```
nc -vz 172.22.112.1 8080
```
- Or curl proxy test (one-off):
```
curl -I -x http://172.22.112.1:8080 -k https://example.com
```
- If you prefer to leave environment variables pointing at 127.0.0.1, you can create a small helper in WSL that resolves and sets the proxy env to the Windows host IP automatically.

---

### 8.6) SSL / CA cert issues
If you see TLS errors or browser SSL warnings:
- Export Burp's CA from Proxy → Options → Import / Export CA and import into the OS/browser certificate store used by your testing profile.
- Restart the browser after importing.
- For WSL curl tests, you can use -k to skip cert verification for quick tests, but for full HTTPS inspection you want the CA trusted where the request originates. Some CLI tools use different cert bundles (e.g., curl in WSL uses WSL's CA bundle).

---

### 8.7) If requests aren’t showing in Burp's Proxy → HTTP history
- Ensure browser proxy is not being bypassed by an extension or OS setting.
- Confirm the FoxyProxy profile is enabled and set to use `127.0.0.1:8080`.
- Confirm Intercept mode (Burp) — if Intercept is ON and you are not forwarding, traffic may pause. Turn Intercept off to allow traffic into HTTP history.
- Check Postman: Postman has its own proxy settings; ensure it’s set to use system proxy or explicitly to `127.0.0.1:8080`.

---

### 8.8) Firewall or security product interference
- Windows Defender or another firewall could block loopback or WSL-originated connections. Try:
  - Add an inbound rule allowing Burp (java) or allow the specific port.
  - Temporarily disable the firewall for a quick test (remember to re-enable it).
- If using corporate VPN or other network tools, they may change routing or block local proxying.

---

### 8.9) Live inspection / captures (advanced)
- From Windows, use Wireshark or tcpdump-equivalent (or run tcpdump in WSL if you need to capture from the WSL VM) to inspect traffic.
- From WSL (root), you can run:
```
sudo tcpdump -i any tcp port 8080 -n -vv
```
Be careful — captures can be large. Stop after you have enough.

---

### 8.10) Process & port cleanup (if port already in use)
If netstat shows another service bound to 8080:
- Identify the PID and stop that process. `taskkill /PID 12345 /F` (use with caution).
- Alternatively, change Burp’s listener to a different free port in Proxy → Options.

---

### 8.11) Common gotchas & quick remedies
- Browser uses system proxy only when profile is configured; some apps ignore system proxy. Double-check in-app proxy settings.
- Multiple proxies / VPNs / firewall rules can cause subtle failures. Disable other proxies while testing.
- If Burp was updated recently and crashes on start, check the Burp UI console and the Burp temp/log folders for stack traces.
- If a tool caches DNS or proxy settings, open a fresh browser profile or new shell session after changing proxy env vars.

---

### Handy reference commands (copy-paste friendly)
- Windows netstat: 
```
netstat -ano | findstr :8080
```
- Windows PID → process: 
```
tasklist /FI "PID eq 12345"
```
- PowerShell connection view: 
```
Get-NetTCPConnection -LocalPort 8080 | Select LocalAddress,LocalPort,State,OwningProcess
```
- PowerShell get process for the port: 
```
Get-Process -Id (Get-NetTCPConnection -LocalPort 8080).OwningProcess
```
- Test from Windows with curl: 
```
curl -I -x http://127.0.0.1:8080 -k https://example.com
```
- Get Windows host IP from WSL: 
```
grep nameserver /etc/resolv.conf
```
- Test from WSL using host IP: 
```
curl -I -x http://<WINDOWS_HOST_IP>:8080 -k https://example.com
```
- WSL netcat test: 
```
nc -vz <WINDOWS_HOST_IP> 8080
```
- Check Burp listener in Burp Proxy → Options (UI)

---

## 9) Security & legal
- Only test explicitly allowed targets (e.g., PortSwigger labs, Hacker101, HTB labs, authorized bounty programs).  
- Keep a personal log: date, target, tests run, scope, proof of permission.

---

## 10) Next steps / Week 2 goals
- Pick 3 live programs to scope for recon.  
- Automate basic recon with a script: `nmap → ffuf → nuclei` and store outputs.  
- Start collecting common payload snippets and checks in `payloads.md`.  
- Complete 2–3 PortSwigger Academy labs & document short notes for each.

---

## 11) References / links
- PortSwigger Academy: https://portswigger.net/web-security  
- Hacker101: https://www.hacker101.com/  
- ffuf docs: https://github.com/ffuf/ffuf
