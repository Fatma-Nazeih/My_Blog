## 🛡️ ثغرات الويب (The Hunter’s Handbook)
### 1. SQL Injection (SQLi)
- **إيه هي؟** الهكر بيحط "تاتش" في جملة الاستعلام عشان يخلي قاعدة البيانات تطلع أسرارها (Usernames, Passwords) أو تمسح بيانات.
- **ألاقيها فين؟ (The Key):** أي مكان بياخد بيانات عشان يعمل Filter أو Search أو Login. دور على الـ Parameters اللي في الـ URL زي `id=1` أو `category=shoes`.
- **أصلحها إزاي؟** استخدم **Parameterized Queries** (أو Prepared Statements). يعني السيرفر بيفصل الكود عن البيانات، فمهما الهكر كتب كود، السيرفر هيعامله كـ "نص" ملوش قيمة تنفيذية.
---
### 2. Cross-Site Scripting (XSS)
- **إيه هي؟** حقن كود JavaScript خبيث في الصفحة. الكود ده بيشتغل عند المستخدم (Client-side) ويقدر يسرق الـ Cookies بتاعته.
- **ألاقيها فين؟ (The Key):** أي مكان بياخد Input ويعرضه تاني في الصفحة (Comments, Profile info, Search results).
- **أصلحها إزاي؟** طبق الـ **Context-aware Output Encoding**. يعني لو هتطبع كلمة "hello" جوه HTML، حول الـ `<` لـ `&lt;` عشان المتصفح ميفهمهاش كـ Tag برمجي. وفعل الـ **CSP (Content Security Policy)**.
---
### 3. Server-Side Request Forgery (SSRF)
- **إيه هي؟** السيرفر بيبقى "خادم مطيع زيادة"، الهكر بيديله رابط لمكان داخلي (زي Cloud Metadata) والسيرفر يروح يجيبهوله.
- **ألاقيها فين؟ (The Key):** أي وظيفة بتطلب ملفات من URLs تانية (Import from URL, Webhooks, PDF Generators).
- **أصلحها إزاي؟** طبق **Strict Allowlist** للـ Domains والـ IPs المسموح للسيرفر يكلمها، وامنع الوصول للـ Internal IPs (زي `127.0.0.1` أو `169.254.169.254`).
---
### 4. XML External Entity (XXE)
- **إيه هي؟** استغلال معالج الـ XML عشان يقرأ ملفات السيرفر أو يكلم سيرفرات تانية.
- **ألاقيها فين؟ (The Key):** الأبلكيشنز اللي بتبعت بيانات بصيغة XML أو بتسمح برفع ملفات Office (.docx) أو صور (.svg) لأنهم في الأصل ملفات XML.
- **أصلحها إزاي؟** عطل الـ **External Entities** والـ **DTD Processing** في مكتبة الـ XML اللي بتستخدمها في الكود (Disable DTDs). 
---
### 5. Server-Side Template Injection (SSTI)
- **إيه هي؟** حقن كود في محرك الـ Templates (زي Jinja2 أو Twig). لو نجحت، الهكر بيسيطر على السيرفر بالكامل (RCE).
- **ألاقيها فين؟ (The Key):** الأماكن اللي بتستخدم "قوالب" عشان تعرض رسايل مخصصة للمستخدم (زي: "Welcome {{user.name}}").
- **أصلحها إزاي؟** متسمحش للمستخدم إنه يكتب كود جوه الـ Template مباشرة. استخدم الـ **Sandboxing** أو دوال الـ Template الآمنة اللي بتفصل بين التصميم والبيانات.
---
### 6. Path Traversal
- **إيه هي؟** الهكر بيستخدم `../` عشان يخرج بره فولدر الموقع ويدخل في ملفات السيستم الحساسة.
- **ألاقيها فين؟ (The Key):** أي Parameter بياخد اسم ملف عشان يعرضه (زي `?file=report.pdf`).
- **أصلحها إزاي؟** استخدم **Hardcoded file paths** ومتحطش الـ Input بتاع المستخدم في مسار الملف مباشرة. واعمل **Validation** إن اسم الملف ميهتويش على `../`.
---
### 7. OS Command Injection
- **إيه هي؟** الهكر بيحقن أوامر "سيستم" (زي `ls`, `cat`, `dir`) جوه وظيفة في الموقع.
- **ألاقيها فين؟ (The Key):** أي أداة بتعمل وظيفة نظام (System calls) زي "Ping" أو "DNS Lookup" أو "Image Compression".
- **أصلحها إزاي؟** ابعد عن الدوال اللي بتنفذ أوامر مباشرة (زي `system()` أو `exec()`). استخدم الـ **Built-in APIs** بتاعة لغة البرمجة نفسها.
---
### 8. CORS Misconfiguration
- **إيه هي؟** ثقة مفرطة في مواقع تانية بتسمح لهم يسرقوا بيانات السيشن بتاعت المستخدمين.
- **ألاقيها فين؟ (The Key):** بص على الـ Headers في الـ Response، لو لقيت `Access-Control-Allow-Origin: *` أو بيعكس الـ `Origin` اللي أنت بتبعته.
- **أصلحها إزاي؟** زي ما قولنا، **Static Whitelist** وممنوع الـ `null` وممنوع الـ `Reflection`.
---
### 9. Information Disclosure
- **إيه هي؟** الموقع "بقّبق" بمعلومات زيادة عن اللزوم (الإصدارات، الـ API Keys، ملفات الـ Backup).
- **ألاقيها فين؟ (The Key):** الـ Error messages، الـ Comments في الـ Source code، ملفات زي `.env` أو `.git`.
- **أصلحها إزاي؟** اقفل الـ **Debug mode** في الـ Production، وامنع الوصول للملفات الحساسة عن طريق الـ `.htaccess` أو إعدادات السيرفر.
---
### 1. الـ Fundamentals (XSS, Path Traversal, OS Injection)
- **XSS (Cross-Site Scripting):**
    - **الفكرة:** الهكر بيحقن كود JavaScript في صفحة بيشوفها مستخدم تاني.
    - **المفتاح (Key):** أي مكان بياخد input منك ويعرضه في الصفحة (Search, Comments, Profile Name).
    - **التست:** جرب `"><script>alert(1)</script>` وشوف هتنطق ولا لأ.
- **Path Traversal:**
    
    - **الفكرة:** التلاعب بمسارات الملفات عشان تقرأ ملفات من السيرفر نفسه (زي `/etc/passwd`).
    - **المفتاح (Key):** أي Parameter بياخد اسم ملف أو مسار (image?name=test.jpg).
    - **التست:** جرب `../../../../etc/passwd`.
- **OS Command Injection:**
    - **الفكرة:** تنفيذ أوامر مباشرة على نظام التشغيل (Linux/Windows) بتاع السيرفر.
    - **المفتاح (Key):** أي وظيفة بتكلم السيستم (Ping tool, Image converter).
    - **التست:** جرب `| whoami` أو `; ls`.
---
### 2. ثغرات الـ Database والـ Backend (SQLi, SSRF)
- **SQL Injection:**
    - **الفكرة:** حقن أوامر SQL عشان تسرق بيانات من قاعدة البيانات أو تعدل فيها.
    - **المفتاح (Key):** الـ ID، الـ Filters، الـ Login forms.
    - **التست:** جرب `' OR 1=1--`.
- **SSRF (Server-Side Request Forgery):**
    - **الفكرة:** بتخلي السيرفر يبعت Request لمكان هو بس اللي يقدر يوصله (زي Cloud metadata أو Localhost).
    - **المفتاح (Key):** أي حاجة بتاخد URL وتعمل له Fetch (Import image from URL).
    - **التست:** جرب `http://169.254.169.254/` أو `http://localhost:80`.
---
### 3. ثغرات الـ Parsing والـ Templates (XXE, SSTI)
- **XXE (XML External Entity):**
    - **الفكرة:** التلاعب بطريقة معالجة السيرفر لملفات الـ XML عشان تسحب ملفات أو تعمل SSRF.
    - **المفتاح (Key):** أي طلب بيبعت بيانات بصيغة XML (تعديل بروفايل، رفع ملفات .docx أو .svg).
    - **التست:** جرب تحقن `<!ENTITY xxe SYSTEM "file:///etc/passwd">`.
- **SSTI (Server-Side Tamplate Injection):**
    - **الفكرة:** حقن كود في محركات الـ Templates (زي Jinja2, Twig) عشان تنفذ كود (RCE).
    - **المفتاح (Key):** أي مكان بيعرض اسمك أو رسالة ترحيب ديناميكية.
    - **التست:** جرب `{{7*7}}` لو طلعت 49 يبقى مبروك.
---
### 4. ثغرات الـ Policy والـ Data (CORS, Info Disclosure)
- **CORS Misconfiguration:**
    - **الفكرة:** السماح لمواقع غريبة إنها تسحب بيانات المستخدمين بسبب إعدادات غلط في الـ Headers.
    - **المفتاح (Key):** الـ `Origin` header في الـ Request.
    - **التست:** غير الـ Origin لموقعك وشوف السيرفر هيرد بـ `Access-Control-Allow-Origin: موقعك` ولا لأ.
- **Information Disclosure:**
    - **الفكرة:** السيرفر بيسرب معلومات مش مفروض تظهر (نسخة السيرفر، Error messages، ملفات .git).
    - **المفتاح (Key):** الـ Headers، الـ Source code، الـ Error pages.
    - **التست:** دور على `/robots.txt` أو `.env` أو جرب تبعت بيانات غلط وتشوف الـ Error.