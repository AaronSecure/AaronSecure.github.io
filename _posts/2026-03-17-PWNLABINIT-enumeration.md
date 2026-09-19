---
title: PWNLAB_INIT
date: 2026-03-17 10:00:00 +0300
categories: [Cybersecurity, Web Exploitation]
tags: [pwnlab, lfi, file-upload, privilege-escalation, pentest]
image:
path: /assets/img/network-scan/pwnlab.jpg
alt: PWNLAB COVER
---

## 1. Executive Summary

This report documents a complete end-to-end penetration test performed against the PwnLab: init 
vulnerable virtual machine a deliberately insecure Capture-The-Flag (CTF) target designed to 
simulate a real-world internal intranet image-sharing web application. The assessment was 
conducted from an attacking Kali Linux machine residing on the same host-only network segment. 
The engagement resulted in a full system compromise, progressing from unauthenticated web 
enumeration through Local File Inclusion (LFI) exploitation, PHP filter wrapper abuse, credential 
extraction from a MySQL database, web shell upload via MIME-type bypass, reverse shell 
establishment, and multi-stage lateral movement through three user accounts (www-data → kane 
→ mike) before achieving root-level access via a SUID binary exploitation technique. 
### Security Findings Summary

Total Findings: **7**

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | 3 |
| 🟠 HIGH | 3 |
| 🟡 MEDIUM | 1 |

#### Detailed Findings

| Category | Finding | Severity |
|----------|---------|----------|
| Web Application | Local File Inclusion (LFI) | 🔴 CRITICAL |
| Authentication | Hardcoded DB Credentials in PHP | 🟠 HIGH |
| File Upload | MIME-Type Bypass (PHP Shell) | 🔴 CRITICAL |
| Database | Weak MD5 Hashed Passwords | 🟠 HIGH |
| Privilege Escalation | SUID Binary (msg2root) | 🔴 CRITICAL |
| Privilege Escalation | Lateral Movement via su kane | 🟠 HIGH |
| OS Hardening | Outdated Apache 2.4.10 | 🟡 MEDIUM |
#### Lab Environment

- Attacker Machine: Kali Linux(192.168.56.104 )
- Target Machine: Debian GNU/Linux (192.168.56.107)
- Network: Oracle VirtualBox — Host-Only / NAT Network 
- Tools Used: Nmap, net discover, Nikto, Burp Suite, CyberChef, MySQL CLI, php-reverse-shell, Netcat (nc), Python pty, vim  
## 2. Scope and Environment
### 2.1 Network Topology

Both machines were hosted inside Oracle VirtualBox. The target machine was configured with a 
Host-Only Adapter (isolating it from the internet), while the attacking machine had two adapters 
one Host-Only for direct communication and one NAT for internet access. 
### Lab Network Configuration

| Machine | Role | IP Address | Network | Adapter Type |
|---------|------|------------|---------|--------------|
| Ethical-Hacker-Kali | 🟢 Attacker | 192.168.56.104 | 192.168.56.0/24 | Host-Only (eth1) + NAT (eth0) |
| PwnLab VM (vm) | 🔴 Target | 192.168.56.107 | 192.168.56.0/24 | Host-Only Adapter Only |

### 2.2 Tools Used 
| Tool | Purpose |
|------|---------|
| 🕵️ Nmap | Network discovery and port/service enumeration |
| 🔍 net discover | ARP-based host discovery to find target IP |
| 🛡️ Nikto | Web server vulnerability scanning |
| 🔐 Burp Suite | HTTP proxy — request interception and manipulation |
| 🧩 CyberChef | Base64 decoding of PHP source code and password hashes |
| 🗄️ MySQL CLI | Remote database access and credential harvesting |
| 💻 php-reverse-shell | PHP web shell to establish a reverse TCP connection |
| 📡 Netcat (nc) | TCP listener for catching the incoming reverse shell |
| 🐍 Python pty | TTY shell upgrade from dumb shell to interactive bash |
| ✏️ vim | Text editor to configure shell payload before upload |
# Procedure
## 3. Reconnaissance 
### 3.1 Host Discovery with net discover 
#### Step 1: Identify the Network (Reconnaissance) 
The attacker first confirmed their own IP address by running ifconfig on the Kali machine. The 
eth1 interface was assigned 192.168.56.104/24 on the Host-Only network. With the network range 
known, net discover was used to perform passive ARP scanning:
```bash
net discover
```
### Scanning for IP of pwnlab using net discover
![Scanning for IP of pwnlab using netdicover](/assets/img/network-scan/net_discover.png)
*Figure 1 Scanning for IP of pwnlab using netdicover*
The scan returned three live hosts. The target was identified by its MAC vendor (PCS Systemtechnik GmbH the VirtualBox vendor signature) at IP 192.168.56.107: 
Note: The method I used to find the IP is by opening my attacking machine scan with netdiscover then after it finished scanning the existing IP I now turned on the pwnlab machine and the IP pop up.

### 3.2 Connectivity Check (Ping)
Before launching a full port scan, a standard ICMP ping test was performed to confirm the target 
was online and responsive: 
![nmap scan](/assets/img/network-scan/Ping_target_ip.png)
*Figure 2 Ping the target ip*
All four packets were returned with 0% packet loss, confirming the machine is alive and responsive.

## 4. Enumeration
### 4.1 Nmap Full Port Scan
A comprehensive Nmap scan was launched using aggressive service detection, version scanning, OS detection, and script scanning. Results were saved to a file for reference:
#### Command
```bash
nmap -sS -A -T4 -oN nmap-scan.txt 192.168.56.107
```
The scan revealed the following open ports and services:
### Nmap Scan Results - Open Ports

### Service Enumeration

| Port | Protocol | Service | Version | Details |
|------|----------|---------|---------|---------|
| **80/tcp** | TCP | HTTP | Apache 2.4.10 | - Web server running on Debian<br>- PwnLab Intranet Image Hosting<br>- Potential LFI, RFI, file upload vulnerabilities |
| **111/tcp** | TCP | rpcbind | RPC #100000 | - RPC service versions 2,3,4<br>- May expose NFS or other RPC services |
| **3306/tcp** | TCP | MySQL | MySQL 5.5.47 | - Remote database server<br>- Protocol version 10<br>- Thread ID: 52<br>- Potential weak credentials or SQL injection |
| **100024/udp** | UDP | status | RPC #100024 | - RPC status service<br>- Information disclosure |

### Attack Surface Analysis

| Service | Attack Vectors | Exploitation Potential |
|---------|---------------|----------------------|
| **HTTP (80/tcp)** | - LFI/RFI<br>- File upload bypass<br>- Directory traversal<br>- Outdated Apache exploits | 🔴 CRITICAL |
| **rpcbind (111/tcp)** | - RPC enumeration<br>- NFS exposure<br>- Information disclosure | 🟡 MEDIUM |
| **MySQL (3306/tcp)** | - Weak credentials<br>- SQL injection<br>- Remote database access<br>- Privilege escalation | 🔴 HIGH |
| **status (100024/udp)** | - Service enumeration<br>- Information leakage | 🟢 LOW |

![nmap scan](/assets/img/network-scan/nmap_scan.png)
*Figure 3 Doing an nmap scan*
### 4.2 Web Application Discovery (Browser Enumeration)
Port 80 was open, so the application was opened directly in the Firefox browser on the Kali machine. The site presented a simple intranet image hosting portal branded as "PWNLAB" with three navigation links: Home, Login, and Upload.
| Page / URL | Observation | Security Issue | Priority |
|------------|-------------|----------------|----------|
| http://192.168.56.107/ | Home page — message: 'Use this server to upload and share image files inside the intranet' | Information disclosure | 🟢 LOW |
| http://192.168.56.107/?page=login | Login form — Username and Password fields with Login button | Potential brute force, SQL injection | 🔴 HIGH |
| http://192.168.56.107/?page=upload | Upload form — 'You must be logged in' (authentication required) | Authentication bypass possible | 🔴 CRITICAL |
| http://192.168.56.107/upload/ | Upload directory listing — Apache directory indexing is ENABLED | Directory listing exposure | 🟠 HIGH |

![nmap scan](/assets/img/network-scan/pwn_homepage.png "Pwnlab Home page")
*Figure 4 Pwnlab Home page*
![nmap scan](/assets/img/network-scan/pwn_loginpage.png "Pwn login page")
*Figure 5 Pwn login page*
![nmap scan](/assets/img/network-scan/pwn_uploadpage.png "pwn_uploadpage")
*Figure 6 Pwn Upload page*
![nmap scan](/assets/img/network-scan/upload_dir.png "upload directory")
*Figure 7 Upload directory*

### 4.3 Nikto Web Vulnerability Scan
Nikto was run against the web server to identify common misconfigurations and vulnerabilities automatically:
#### Command
```bash
nikto -h http://192.168.56.107
```
![Nikto Web Scan](/assets/img/network-scan/nikto_scan.jpg "Nikto Web Vulnerability Scan")
*Figure 8 — Nikto Web Vulnerability Scan*
#### *Key findings reported by Nikto included:*
•	Missing X-Frame-Options header — potential Clickjacking risk.
•	Missing X-Content-Type-Options header — MIME sniffing vulnerability.
•	Apache/2.4.10 is outdated — End-of-Life with known CVEs (minimum 2.4.54 recommended at time of scan).
•	No CGI directories found.
•	/login.php — PHPSESSID cookie created without the HttpOnly flag (session hijacking risk).
•	/config.php — identified as a PHP config file that may contain database IDs and passwords.
•	/#wp-config.php# — file found, noted as containing credentials.
•	8102 requests completed; 12 items reported.

## 5. Exploitation Phase 1 Local File Inclusion (LFI)
### 5.1 Identifying the LFI Vulnerability
The URL structure of the web application used a GET parameter called "page" to load PHP files:
#### URL Pattern
```bash
http://192.168.56.107/?page=login
```
This pattern is a classic indicator of a potential Local File Inclusion vulnerability. The application appeared to use `include()` or `require()` in PHP to dynamically load page files based on the `page` parameter value. Burp Suite was opened to intercept and inspect the HTTP requests to the login, home, and upload pages. The request headers were examined and the cookie structure was noted.

![Burp intercept of the login page](/assets/img/network-scan/burp-login.jpg)
*Figure 9 — Burp Suite intercept of the login request*

![Burp intercept of the upload page](/assets/img/network-scan/burp-upload.jpg)
*Figure 10 — Burp Suite intercept of the upload request*

![Burp intercept of the home page](/assets/img/network-scan/burp-home.jpg)
*Figure 11 — Burp Suite intercept of the home request*

### 5.2 PHP Filter Wrapper — Reading Server-Side Source

Because the application executed included PHP files, a PHP filter wrapper was used to read the raw source as base64 before it ran. This approach was referenced from the Deep Hacking write-up on PHP wrappers.

![PHP filter wrapper reference](/assets/img/network-scan/php-filter-reference.jpg)
*Figure 12 — PHP filter wrapper technique used to read source as base64*

The `page` parameter was pointed at `config` through the filter wrapper. The page returned a long base64 string instead of executing the config file.

![Base64 output of config.php](/assets/img/network-scan/lfi-config-b64.jpg)
*Figure 13 — Base64-encoded contents of config.php*

### 5.3 Decoding config.php — Database Credentials

The base64 string was decoded in CyberChef (`From Base64`). The decoded `config.php` contained hardcoded MySQL credentials:

- Server: `localhost`
- Username: `root`
- Password: `H4u%QJ_H99`
- Database: `Users`

![CyberChef decode of config.php](/assets/img/network-scan/cyberchef-config.jpg)
*Figure 14 — CyberChef decode of config.php*

The same filter technique was applied to `index.php` to understand application logic, including language-cookie handling.

![CyberChef decode of index.php](/assets/img/network-scan/cyberchef-index.jpg)
*Figure 15 — Decoded index.php showing the lang cookie include*

The decoded source showed that if a `lang` cookie is present, PHP includes a file under `lang/` using that cookie value. That include became the later execution path for an uploaded file.

## 6. Exploitation Phase 2 — Database Access and Credential Harvesting

### 6.1 Remote MySQL Access

Using the credentials from `config.php`, a remote MySQL session was opened from Kali:

```bash
mysql -u root -p -h 192.168.56.107
```

The connection succeeded. Databases were listed, `Users` was selected, and the `users` table was dumped.

![MySQL connection and database listing](/assets/img/network-scan/mysql-connect.jpg)
*Figure 16 — Remote MySQL login and schema enumeration*

![Users table dump](/assets/img/network-scan/mysql-users.jpg)
*Figure 17 — Usernames and encoded passwords in the users table*

### 6.2 Password Decoding

The stored passwords were base64-encoded rather than hashed. Each value was decoded in CyberChef.

| Username | Encoded password (from DB) | Decoded password |
|----------|----------------------------|------------------|
| kent | `Sld6WHVCSkp0eQ==` | `JWzXuBJJNy` |
| mike | base64 value from the dump | decoded in CyberChef |
| kane | base64 value from the dump | `iSv5Ym26Ro` (used successfully with `su`) |

![CyberChef decode of kent password](/assets/img/network-scan/cyberchef-kent.jpg)
*Figure 18 — kent password decoded to JWzXuBJJNy*

## 7. Exploitation Phase 3 — Authenticated Upload

### 7.1 Logging in as kent

The decoded kent credentials were used on `/?page=login`. After login, the upload form became available.

![Login as kent](/assets/img/network-scan/login-kent.jpg)
*Figure 19 — Logging in as kent*

![Upload page after login](/assets/img/network-scan/upload-after-login.jpg)
*Figure 20 — Authenticated upload page*

### 7.2 Preparing a PHP reverse-shell script

A public PHP reverse-shell script (PentestMonkey) was downloaded and extracted. The listener was set to the Kali host-only address and port **1234**.

![PentestMonkey php-reverse-shell page](/assets/img/network-scan/pentestmonkey-shell.jpg)
*Figure 21 — PHP reverse-shell download page*

![Extracting the reverse-shell archive](/assets/img/network-scan/shell-extract.jpg)
*Figure 22 — Archive extraction*

![Renaming the shell script](/assets/img/network-scan/shell-rename.jpg)
*Figure 23 — Script renamed for upload*

![Listener IP and port configuration](/assets/img/network-scan/shell-ip-port.jpg)
*Figure 24 — Attacker IP 192.168.56.104 and port 1234*

### 7.3 Upload filter bypass (GIF89a header)

A first upload was rejected because the form expected an image. A `GIF89a` magic-byte header was prepended and the file was renamed to `php-shell.png` so a simple type check would accept it.

![GIF89a header prepended](/assets/img/network-scan/gif89a-header.png)
*Figure 25 — GIF89a header added at the start of the file*

![php-shell.png selected for upload](/assets/img/network-scan/php-shell-png.jpg)
*Figure 26 — Disguised file ready in the file picker*

### 7.4 Uploading and confirming the file

The file was submitted through the upload page. The UI then showed "No file selected", but `/upload/` still listed a randomised PNG.

![Selecting php-shell.png on the upload form](/assets/img/network-scan/upload-php-shell.jpg)
*Figure 27 — Upload form with php-shell.png selected*

![Upload form after submit](/assets/img/network-scan/upload-no-file-selected.jpg)
*Figure 28 — Form reset after submit*

![Upload directory showing the stored file](/assets/img/network-scan/upload-dir-shell.jpg)
*Figure 29 — Stored file `356a194153f9b5e71b23dddd665d1dcb.png`*

## 8. Exploitation Phase 4 — Reverse Shell as www-data

### 8.1 Netcat listener

A listener was started on Kali before triggering execution:

```bash
nc -nvlp 1234
```

### 8.2 Triggering execution via the lang cookie

Directly opening the uploaded file did not give a shell. The `lang` cookie include in `index.php` was used instead: Burp intercepted a request to `/` and the cookie was set to the uploaded file under `/upload/`.

![Burp request before cookie change](/assets/img/network-scan/burp-before-cookie.jpg)
*Figure 30 — Intercepted home request before cookie change*

![lang cookie pointing at the uploaded file](/assets/img/network-scan/burp-lang-cookie.jpg)
*Figure 31 — lang cookie set to `../upload/356a194153f9b5e71b23dddd665d1dcb.png`*

### 8.3 Initial foothold

Forwarding that request caused the uploaded PHP to run. Netcat received a connection from `192.168.56.107` as **www-data**.

![Netcat catching the reverse connection](/assets/img/network-scan/nc-listener-www.jpg)
*Figure 32 — Reverse connection as www-data*

A TTY upgrade was then used so the session could run `su` and other interactive commands:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

![TTY upgrade and /etc/passwd](/assets/img/network-scan/tty-upgrade.jpg)
*Figure 33 — Interactive bash and local user enumeration*

## 9. Privilege Escalation — Root Compromise

### 9.1 Lateral movement: www-data → kane

From the www-data shell, `su kane` was used with the decoded kane password (`iSv5Ym26Ro`). After `cd ~`, `msgmike` was found in kane's home directory.

![Switching to kane](/assets/img/network-scan/su-kane.jpg)
*Figure 34 — su kane and msgmike in the home directory*

### 9.2 Lateral movement: kane → mike (`msgmike`)

`file` and `strings` showed `msgmike` is a 32-bit SUID binary owned by mike that calls `cat` with a relative path (`cat /home/mike/msg.txt`). That is a PATH-hijack issue: a fake `cat` in `/tmp`, then `/tmp` prepended to `PATH`, caused the SUID binary to run the attacker's `cat` and spawn a shell as **mike**.

![msgmike analysis and PATH setup](/assets/img/network-scan/msgmike-analysis.png)
*Figure 35 — SUID msgmike analysis and PATH hijack attempts*

![Shell as mike](/assets/img/network-scan/path-hijack-mike.png)
*Figure 36 — Successful PATH hijack: uid=1002(mike)*

### 9.3 Root escalation: mike → root (`msg2root`)

Mike's home directory contained another SUID binary, `msg2root`. It prompted for a message. `strings` showed it used `system()` with `/bin/echo %s >> /root/messages.txt`, so the message was passed to a shell.

![strings output for msg2root](/assets/img/network-scan/msg2root-strings.png)
*Figure 37 — msg2root calling system() on unsanitized input*

A shell metacharacter in the message field (`test;/bin/sh`) produced a shell with **euid=0 (root)**.

![Root shell via msg2root](/assets/img/network-scan/root-shell.png)
*Figure 38 — euid=0 after command injection in msg2root*

### 9.4 Capturing the root flag

`cat flag.txt` from `/root` dropped back to mike because `cat` was still the hijacked copy on `PATH`. Using the absolute path worked:

```bash
/bin/cat /root/flag.txt
```

![Root flag](/assets/img/network-scan/root-flag.png)
*Figure 39 — Root flag from /root/flag.txt*

## 10. Full Attack Chain Summary

| Step | Phase | Action | Outcome |
|------|-------|--------|---------|
| 1 | Reconnaissance | netdiscover + ping | Target 192.168.56.107 live |
| 2 | Enumeration | nmap -sS -A -T4 | Ports 80, 111, 3306; Apache 2.4.10 + MySQL 5.5 |
| 3 | Enumeration | Browser + Nikto | Intranet app, `/upload/` listing, config.php noted |
| 4 | LFI | PHP filter wrapper on config.php / index.php | DB credentials and lang-cookie include revealed |
| 5 | Database | mysql -u root -h 192.168.56.107 | Users table dumped; passwords decoded |
| 6 | Upload | Login as kent; GIF89a + .png upload | Shell stored as randomised PNG under /upload/ |
| 7 | Execution | lang cookie → uploaded file + nc -nvlp 1234 | Shell as www-data |
| 8 | Lateral | su kane | Access to msgmike |
| 9 | Lateral | PATH hijack on msgmike | Shell as mike |
| 10 | Root | Command injection in msg2root | euid=0; flag read with /bin/cat |

## 11. Vulnerabilities and Recommendations

### VULN-01: Local File Inclusion via page parameter and lang cookie

**Severity:** 🔴 CRITICAL  
**Location:** `index.php` (`page` parameter and `lang` cookie)

User-controlled values were passed into PHP `include()` without an allowlist. Combined with PHP filters, this disclosed source; combined with an upload, it executed code.

**Remediation:** Never pass user input into `include()`/`require()`. Allowlist language files. Keep `allow_url_include=Off`.

### VULN-02: Hardcoded database credentials in config.php

**Severity:** 🟠 HIGH  
**Location:** `config.php` — MySQL root account

Credentials were readable via LFI, and MySQL accepted a remote root login.

**Remediation:** Store secrets outside the web root. Use a least-privilege DB account. Bind MySQL to localhost.

### VULN-03: Weak password storage (Base64)

**Severity:** 🟠 HIGH  
**Location:** `Users.users` table

Passwords were encoded, not hashed, so they decoded instantly.

**Remediation:** Use bcrypt / Argon2 (`password_hash()`). Never treat encoding as encryption.

### VULN-04: File upload type bypass

**Severity:** 🔴 CRITICAL  
**Location:** Authenticated upload form

Validation trusted a GIF magic header / image extension. PHP in the uploaded file still executed later via LFI.

**Remediation:** Server-side MIME checks, random names, uploads outside the web root, PHP engine off in the upload directory.

### VULN-05: SUID PATH hijacking (msgmike)

**Severity:** 🟠 HIGH  
**Location:** `/home/kane/msgmike`

The SUID binary called `cat` without an absolute path.

**Remediation:** Use absolute paths in privileged binaries. Sanitize `PATH`. Avoid SUID where possible.

### VULN-06: Command injection in msg2root

**Severity:** 🔴 CRITICAL  
**Location:** `/home/mike/msg2root`

User input was passed to `system()`. Shell metacharacters ran as root.

**Remediation:** Do not use `system()` on unsanitized input. Prefer `execve()` with a fixed argument list. Avoid `gets()`.

### VULN-07: Apache directory listing

**Severity:** 🟡 MEDIUM  
**Location:** `/upload/`

Directory indexing revealed the randomised upload name.

**Remediation:** `Options -Indexes`. Keep uploads out of the web root.

### VULN-08: Outdated Apache / OS

**Severity:** 🟡 MEDIUM  
**Location:** Apache 2.4.10 on Debian 8 (Jessie)

Both components are end-of-life.

**Remediation:** Upgrade to a supported OS and current Apache 2.4.x. Maintain a patch process.

## 12. Conclusion

PwnLab: init was fully compromised through a chain of issues that each unlocked the next step: LFI and source disclosure, weak credential storage, a bypassable upload filter, cookie-based inclusion, then two SUID binaries. No single finding was the whole breach. Proper hashing, an include allowlist, PHP disabled in the upload directory, or safer SUID code would have broken the chain.

## References

- OWASP Foundation. (2021). *OWASP Top 10*. <https://owasp.org/www-project-top-ten/>
- Scarfone, K., Souppaya, M., & Cody, A. (2008). *NIST SP 800-115: Technical Guide to Information Security Testing and Assessment*
- ENISA. (2022). *ENISA Threat Landscape*
- ISO/IEC 27002:2022 — Information security controls
- SANS Institute — penetration testing methodology references
