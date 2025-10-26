# تدريبات عملية في أمن الشبكات
## Network Security - Practical Labs

---

## 📑 جدول المحتويات

1. [NAT/PAT - ترجمة عناوين الشبكة](#1-natpat---ترجمة-عناوين-الشبكة)
2. [ACL - قوائم التحكم بالوصول](#2-acl---قوائم-التحكم-بالوصول)
3. [FTP وآلية Established](#3-ftp-وآلية-established)
4. [WAF و Load Balancer](#4-waf-و-load-balancer)
5. [Cisco Firepower - تحليل البرمجيات الخبيثة](#5-cisco-firepower---تحليل-البرمجيات-الخبيثة)
6. [ARP - بروتوكول تحليل العناوين](#6-arp---بروتوكول-تحليل-العناوين)
7. [Nmap - مسح الشبكات](#7-nmap---مسح-الشبكات)
8. [ICMP - اختبارات الشبكة](#8-icmp---اختبارات-الشبكة)
9. [SYN Flooding Attack](#9-syn-flooding-attack)
10. [UDP Flood Attack](#10-udp-flood-attack)
11. [Nmap Port Scanning](#11-nmap-port-scanning)
12. [John the Ripper - كسر كلمات المرور](#12-john-the-ripper---كسر-كلمات-المرور)
13. [DHCP Snooping](#13-dhcp-snooping)

---

## 1. NAT/PAT - ترجمة عناوين الشبكة

### 🎯 الهدف
فهم آلية عمل NAT/PAT في ترجمة العناوين الخاصة إلى عناوين عامة.

### 📋 السيناريو
جهاز داخلي بعنوان `10.0.0.5` يريد تصفح الإنترنت عبر جدار ناري يستخدم NAT.

### 🔄 خطوات العمل

1. **الطلب الصادر:**
   - الجهاز الداخلي `10.0.0.5` يرسل طلب HTTP
   - الجدار الناري يترجم العنوان إلى `203.0.113.2:10500`
   - العنوان `203.0.113.2` هو العنوان العام
   - المنفذ `10500` يُستخدم لتمييز الاتصال

2. **الرد الوارد:**
   - عند وصول الرد من الإنترنت إلى `203.0.113.2:10500`
   - يعيد NAT الترجمة العكسية إلى `10.0.0.5` بالمنفذ الأصلي
   - يتم تسليم البيانات للجهاز الداخلي

### ✅ الفائدة
- مئات الأجهزة يمكنها مشاركة عنوان عام واحد
- لا يوجد تضارب بفضل استخدام أرقام المنافذ (Port Numbers)
- توفير في عناوين IPv4 العامة

### 📝 ملاحظات
- PAT (Port Address Translation) هو نوع من NAT
- يُسمى أيضاً NAT Overload
- الجدول الداخلي يحتفظ بالربط بين العناوين الداخلية والمنافذ

---

## 2. ACL - قوائم التحكم بالوصول

### 🎯 الهدف
تطبيق سياسات أمنية باستخدام قوائم التحكم بالوصول (Access Control Lists).

### 📋 السيناريو
لديك:
- **Web Server** في DMZ (Demilitarized Zone)
- **Database Server** داخل الشبكة الداخلية

> 💡 **DMZ:** منطقة منزوعة السلاح، وهي منطقة بين الشبكة الداخلية والإنترنت لتقليل المخاطر الأمنية.

### 🔧 التكوين العملي

```cisco
access-list 100 permit tcp any 10.10.10.10 eq www
access-list 100 permit tcp any 10.10.10.10 eq 443
access-list 100 permit tcp any 10.10.10.12 eq ftp
access-list 100 permit tcp any 10.10.10.12 eq ftp-data
access-list 100 deny ip any any log

interface gi0/1
ip access-group 100 in
```

### 📊 شرح الأوامر

| الأمر | الوظيفة |
|------|---------|
| `permit tcp any 10.10.10.10 eq www` | السماح بالوصول لخادم الويب عبر HTTP (منفذ 80) |
| `permit tcp any 10.10.10.10 eq 443` | السماح بالوصول عبر HTTPS (منفذ 443) |
| `permit tcp any 10.10.10.12 eq ftp` | السماح بـ FTP Control (منفذ 21) |
| `permit tcp any 10.10.10.12 eq ftp-data` | السماح بـ FTP Data (منفذ 20) |
| `deny ip any any log` | حظر أي حركة أخرى مع التسجيل |
| `ip access-group 100 in` | تطبيق القائمة على الواجهة |

### 🎯 الهدف الأمني
- السماح بالوصول فقط للمنافذ المطلوبة
- منع الوصول المباشر لخادم قاعدة البيانات
- تسجيل المحاولات المشبوهة
- تقليل نطاق الهجوم (Attack Surface)

### ⚠️ ملاحظات مهمة
- الوصول لقاعدة البيانات يجب أن يتم فقط من خلال خادم الويب
- ترتيب القواعد مهم جداً (من الأعلى للأسفل)
- القاعدة الضمنية في النهاية: `deny any any`

---

## 3. FTP وآلية Established

### 🎯 الهدف
فهم الفرق بين FTP Passive و Active Mode وتأثيره على ACL.

### 📋 بنية FTP
بروتوكول FTP يستخدم قناتين منفصلتين:
- **Port 21:** قناة التحكم (Control Channel)
- **Port 20:** قناة البيانات (Data Channel)

### 🔄 FTP Passive Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Random High Port (Data)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يطلب وضع Passive
3. الخادم يرد بمنفذ عشوائي عالي
4. العميل يفتح اتصال البيانات نحو هذا المنفذ
5. كل الردود تحتوي على بت ACK ← **مسموح بها** ✅

### 🔄 FTP Active Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Port 20 (Data initiated by server)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يخبر الخادم بمنفذ معين لاستقبال البيانات
3. **الخادم يبدأ اتصال البيانات** من المنفذ 20
4. أول حزمة من الخادم تحتوي على SYN فقط ← **تُرفض** 🚫

### ⚠️ المشكلة مع Established

```cisco
access-list 110 permit tcp any any established
```

هذه القاعدة تسمح فقط بالحزم التي تحتوي على:
- بت ACK
- أو بت RST

**النتيجة:**
- في Active Mode، الخادم يرسل SYN أولاً
- هذه الحزمة لا تحقق شرط "established"
- الاتصال يفشل 🚫

### ✅ الحل الأمثل
استخدام جدار ناري حقيقي (Stateful Firewall) قادر على:
- تتبع الجلسات النشطة
- فهم البروتوكولات المعقدة مثل FTP
- السماح بالاتصالات الديناميكية تلقائياً

### 📝 الخلاصة
- `established` لا يكفي للبروتوكولات المعقدة
- FTP Passive Mode أكثر توافقاً مع الجدران النارية
- Stateful Firewalls ضرورية للبيئات الإنتاجية

---

## 4. WAF و Load Balancer

### 🎯 الهدف
بناء بنية تحتية آمنة وقابلة للتوسع لموقع ويب.

### 📋 السيناريو
موقع ويب يحتوي على عدة خوادم Web Servers خلف Load Balancer، مع حماية من هجمات XSS.

### 🏗️ البنية المعمارية

```
[Internet]
    |
    ↓
[WAF - Web Application Firewall]
    |
    ↓
[Load Balancer]
    |
    ↓------------------↓------------------↓------------------↓
[Web Server 1]   [Web Server 2]   [Web Server 3]   [Web Server 4]
```

### ⚖️ مهام Load Balancer

**خوارزمية التوزيع:**
- **Round Robin:** توزيع متساوٍ على جميع الخوادم
- دورة منتظمة: Server 1 → Server 2 → Server 3 → Server 4 → تكرار

**إدارة الأعطال:**
1. مراقبة صحة الخوادم (Health Checks)
2. عند فشل أحد الخوادم:
   - إزالته من قائمة التوزيع تلقائياً
   - تحويل طلباته إلى الخوادم السليمة
3. عند عودة الخادم: إعادة إدراجه تلقائياً

### 🛡️ مهام WAF (كـ Reverse Proxy)

**الحماية من XSS:**
```javascript
// مثال على محاولة هجوم XSS
<script>alert('XSS Attack')</script>
```

**آلية الحماية:**
1. فحص كل طلب HTTP قبل وصوله للخوادم
2. كشف الشيفرات الخبيثة:
   - JavaScript مشبوه
   - SQL Injection
   - Command Injection
3. منع الطلبات المشبوهة فوراً ⛔
4. منع تسريب البيانات الحساسة في الاستجابات

### 🔍 سياسات الأمان

| نوع الحماية | الوظيفة |
|-------------|---------|
| XSS Prevention | منع حقن JavaScript |
| SQL Injection | حماية قواعد البيانات |
| CSRF Protection | منع الطلبات المزورة |
| Rate Limiting | الحد من معدل الطلبات |
| IP Blacklisting | حظر IP المشبوهة |

### ✅ المزايا المحققة

**الأداء:**
- ⚡ توزيع الأحمال بشكل متوازن
- 🔄 تحمل الأعطال (Fault Tolerance)
- 📈 قابلية التوسع الأفقي

**الأمان:**
- 🛡️ حماية متقدمة من WAF
- 🔒 فحص جميع الطلبات
- 📝 تسجيل المحاولات المشبوهة
- 🚫 حظر الهجمات في الوقت الحقيقي

### 📊 مثال على سجلات WAF

```
2025-10-26 10:15:32 | BLOCKED | XSS Attempt | IP: 192.168.1.100
2025-10-26 10:16:45 | BLOCKED | SQL Injection | IP: 10.0.0.50
2025-10-26 10:17:12 | ALLOWED | Normal Request | IP: 172.16.0.25
```

---

## 5. Cisco Firepower - تحليل البرمجيات الخبيثة

### 🎯 الهدف
فهم كيفية عمل أنظمة كشف البرمجيات الخبيثة المتقدمة وتتبع انتشارها.

### 📋 السيناريو
ملف باسم `tool.exe` تم اكتشافه ضمن حركة المرور على الشبكة.

### 🔍 مراحل الكشف والتحليل

#### المرحلة 1: الفحص الأولي
1. **اعتراض الملف:** جهاز Cisco Firepower يفحص جميع الملفات المارة
2. **حساب البصمة:** استخراج بصمة SHA-256 للملف
   ```
   SHA-256: a3f7b8c9d2e1f4a6b5c8d7e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
   ```
3. **المقارنة:** مقارنة البصمة مع قاعدة بيانات التهديدات

#### المرحلة 2: التصنيف
```
Status: ⚠️ MALWARE DETECTED
Threat Name: Generic.Trojan.Downloader
Threat Score: 95/100 (High)
Category: Trojan
First Seen: 2025-10-25 14:30:22
```

#### المرحلة 3: تحديد الأجهزة المصابة
الأجهزة التالية تم تحديدها كمصابة:
- 🖥️ `192.168.10.90`
- 🖥️ `192.168.133.50`

### 📈 Trajectory Map (خريطة المسار)

```
Time →
Device 1: --------[*]==========[*]-------
                   ↓            ↓
Device 2: --------[*]----[*]----[*]------
                         ↓      ↓
Device 3: ---------------[*]=====[*]-----
```

**رموز الخريطة:**
- `[*]` نقطة انتقال الملف
- `=` فترة نشاط الملف على الجهاز
- `↓` اتجاه انتقال العدوى (من جهاز لآخر)
- `-` فترة خمول

### 📊 جدول الأحداث (Events Table)

| الوقت | الجهاز المصدر | الجهاز الهدف | الحدث | الحالة |
|------|---------------|--------------|--------|--------|
| 14:30:22 | External | 192.168.10.90 | File Download | Infected |
| 14:35:18 | 192.168.10.90 | 192.168.133.50 | File Transfer | Infected |
| 14:42:33 | 192.168.133.50 | Network Share | Propagation Attempt | Blocked |

### 🎯 استخدامات Trajectory Map

1. **تحديد المصدر الأولي:**
   - أول جهاز تلقى الملف
   - نقطة الدخول إلى الشبكة

2. **تتبع مسار الانتشار:**
   - الأجهزة الوسيطة
   - طرق الانتقال المستخدمة

3. **تقييم نطاق الإصابة:**
   - عدد الأجهزة المصابة
   - الأقسام المتأثرة

4. **التخطيط للاستجابة:**
   - أولوية المعالجة
   - خطة العزل والإزالة

### 🚨 خطة الاستجابة للحادث

**الخطوات الفورية:**
1. ✅ عزل الأجهزة المصابة عن الشبكة
2. ✅ حظر بصمة الملف على جميع الأجهزة
3. ✅ فحص شامل للأجهزة المجاورة
4. ✅ تحليل سجلات النشاط

**الإجراءات التصحيحية:**
1. 🔧 إزالة البرمجية الخبيثة
2. 🔧 تحديث تعريفات مكافح الفيروسات
3. 🔧 تصحيح الثغرات المستغلة
4. 🔧 استعادة النظام من نسخة احتياطية نظيفة

**المتابعة:**
1. 📝 توثيق الحادث
2. 📝 تحديث السياسات الأمنية
3. 📝 تدريب المستخدمين
4. 📝 مراقبة معززة لفترة

### 💡 الدروس المستفادة
- أهمية الفحص العميق لحركة المرور
- ضرورة التسجيل الشامل للأحداث
- قيمة أدوات التحليل البصري
- أهمية الاستجابة السريعة

---

## 6. ARP - بروتوكول تحليل العناوين

### 🎯 الهدف
فهم آلية عمل بروتوكول ARP ومراقبة الجدول الخاص به.

### 📋 نظرة عامة
ARP (Address Resolution Protocol) يربط بين:
- **عنوان IP** (Layer 3)
- **عنوان MAC** (Layer 2)

### 🔍 التطبيق العملي

#### الخطوة 1: عرض جدول ARP

**على Windows:**
```cmd
arp -a
```

**على Linux/macOS:**
```bash
arp -a
```

#### الخطوة 2: قراءة النتائج

```
Interface: 192.168.1.100 --- 0x4
  Internet Address      Physical Address      Type
  192.168.1.1          00-1A-2B-3C-4D-5E     dynamic
  192.168.1.10         00-AA-BB-CC-DD-EE     dynamic
  192.168.1.255        FF-FF-FF-FF-FF-FF     static
  224.0.0.22           01-00-5E-00-00-16     static
```

### 📊 تحليل الجدول

| المكون | الشرح |
|--------|--------|
| **Internet Address** | عنوان IP للجهاز |
| **Physical Address** | عنوان MAC للجهاز |
| **Type** | نوع الإدخال |

### 🏷️ أنواع الإدخالات

**Dynamic:**
- يتم تعلمه تلقائياً من حركة المرور
- يُحذف بعد فترة عدم النشاط (عادة 2-10 دقائق)
- الأكثر شيوعاً في الشبكات

**Static:**
- يتم إضافته يدوياً بواسطة المدير
- لا يُحذف تلقائياً
- يُستخدم للأجهزة الحساسة

### 🔄 عناوين خاصة

**Broadcast:**
```
192.168.1.255 → FF-FF-FF-FF-FF-FF
```
- يُرسل لجميع الأجهزة في الشبكة

**Multicast:**
```
224.0.0.22 → 01-00-5E-00-00-16
```
- يُرسل لمجموعة محددة من الأجهزة

### 🧹 مسح الكاش

**مسح جميع الإدخالات (Windows):**
```cmd
arp -d *
```

**مسح إدخال محدد:**
```cmd
arp -d 192.168.1.10
```

**على Linux:**
```bash
sudo ip -s -s neigh flush all
```

### 📝 إضافة إدخال ثابت

**Windows:**
```cmd
arp -s 192.168.1.50 00-11-22-33-44-55
```

**Linux:**
```bash
sudo arp -s 192.168.1.50 00:11:22:33:44:55
```

### 🔍 حالات الاستخدام

**1. تشخيص مشاكل الاتصال:**
```bash
# فحص وجود الجهاز في الجدول
arp -a | grep 192.168.1.10

# إذا لم يظهر، المشكلة قد تكون في Layer 2
```

**2. كشف تضارب IP:**
```bash
# إذا ظهر نفس IP بعنواني MAC مختلفين
192.168.1.50    00-AA-BB-CC-DD-EE
192.168.1.50    00-11-22-33-44-55  # ⚠️ تضارب!
```

**3. مراقبة الأجهزة النشطة:**
```bash
# عد الأجهزة المتصلة
arp -a | grep dynamic | wc -l
```

### ⚠️ ملاحظات أمنية

**ARP Spoofing/Poisoning:**
- مهاجم يرسل رسائل ARP مزيفة
- يخدع الأجهزة لإرسال البيانات له
- **الحماية:** ARP Inspection على السويتشات

**الوقاية:**
```cisco
# تفعيل Dynamic ARP Inspection
Switch(config)# ip arp inspection vlan 10
Switch(config)# interface gi0/1
Switch(config-if)# ip arp inspection trust
```

### 💡 نصائح عملية
- راقب جدول ARP بشكل دوري
- انتبه للتغييرات المفاجئة في عناوين MAC
- استخدم إدخالات ثابتة للأجهزة الحساسة
- فعّل ARP Inspection في البيئات الحساسة

---

## 7. Nmap - مسح الشبكات

### 🎯 الهدف
استخدام Nmap لاكتشاف الأجهزة والخدمات النشطة على الشبكة.

### 📋 المتطلبات
- نظام Linux (يفضل Kali Linux)
- صلاحيات root للمسح المتقدم
- **بيئة اختبارية فقط** ⚠️

### 🔍 أنواع المسح

#### 1. اكتشاف الأجهزة النشطة (Host Discovery)

```bash
nmap -sn 192.168.1.0/24
```

**الشرح:**
- `-sn`: Ping Scan (بدون فحص المنافذ)
- يستخدم ICMP Echo Request
- يُظهر الأجهزة المتصلة والنشطة

**مثال على النتائج:**
```
Starting Nmap 7.94
Nmap scan report for 192.168.1.1
Host is up (0.0012s latency).

Nmap scan report for 192.168.1.50
Host is up (0.0034s latency).

Nmap scan report for 192.168.1.100
Host is up (0.0028s latency).

Nmap done: 256 IP addresses (3 hosts up) scanned in 2.85 seconds
```

#### 2. فحص المنافذ (Port Scanning)

**Basic TCP Scan:**
```bash
nmap 192.168.1.50
```

**Comprehensive Scan:**
```bash
nmap -sS -sV -O 192.168.1.50
```

**الشرح:**
- `-sS`: SYN Stealth Scan (أقل قابلية للكشف)
- `-sV`: كشف إصدارات الخدمات
- `-O`: تخمين نظام التشغيل

**مثال على النتائج:**
```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 8.2p1
80/tcp   open  http       Apache httpd 2.4.41
443/tcp  open  ssl/http   Apache httpd 2.4.41
3306/tcp open  mysql      MySQL 5.7.33

OS details: Linux 3.2 - 4.9
```

#### 3. فحص جميع المنافذ

```bash
nmap -p- 192.168.1.50
```

- `-p-`: فحص جميع المنافذ (1-65535)
- يستغرق وقتاً أطول

#### 4. فحص منافذ محددة

```bash
nmap -p 21,22,80,443,3306 192.168.1.50
```

### 🎭 أنواع مسح المنافذ

#### TCP SYN Scan (الأكثر شيوعاً)
```bash
nmap -sS 192.168.1.50
```
- يرسل SYN فقط
- لا يُكمل المصافحة الثلاثية
- أقل قابلية للكشف

#### TCP Connect Scan
```bash
nmap -sT 192.168.1.50
```
- يُكمل المصافحة الثلاثية
- لا يحتاج صلاحيات root
- أكثر قابلية للكشف

#### UDP Scan
```bash
nmap -sU 192.168.1.50
```
- فحص منافذ UDP
- بطيء جداً
- مهم لخدمات مثل DNS, DHCP

### 🔬 فحص الثغرات

```bash
nmap --script vuln 192.168.1.50
```

يستخدم NSE (Nmap Scripting Engine) للبحث عن ثغرات معروفة.

### 📊 تقييم المخاطر

#### منافذ عالية الخطورة:

| المنفذ | الخدمة | الخطر |
|--------|---------|-------|
| 21 | FTP | نقل بيانات بدون تشفير |
| 23 | Telnet | إدارة عن بعد بدون تشفير |
| 445 | SMB | عرضة لهجمات WannaCry وغيرها |
| 3389 | RDP | هدف شائع للهجمات |
| 1433 | MSSQL | قاعدة بيانات قد تكون عرضة |

### 🛡️ الحماية من Nmap

**1. على مستوى الجدار الناري:**
```cisco
# رفض المسح من مصادر خارجية
access-list 100 deny tcp any any range 1 1024
```

**2. على مستوى الخادم:**
```bash
# استخدام iptables لمنع المسح
iptables -A INPUT -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT
```

**3. أدوات كشف المسح:**
```bash
# تثبيت portsentry
sudo apt install portsentry

# مراقبة محاولات المسح
tail -f /var/log/syslog | grep portsentry
```

### 💡 نصائح للاستخدام الآمن
- ⚠️ **لا تقم بمسح شبكات لا تملكها**
- 📝 احصل على إذن خطي قبل أي اختبار اختراق
- 🔒 استخدم بيئة مختبرية معزولة للتدريب
- 📊 وثّق جميع عمليات المسح ونتائجها

### 🎓 أمثلة تعليمية إضافية

**مسح سريع لأهم 100 منفذ:**
```bash
nmap --top-ports 100 192.168.1.50
```

**مسح مع تجنب الكشف:**
```bash
nmap -sS -T2 -f 192.168.1.50
```
- `-T2`: سرعة بطيئة (أقل إزعاجاً)
- `-f`: تجزئة الحزم

**حفظ النتائج:**
```bash
nmap -oN results.txt 192.168.1.50
nmap -oX results.xml 192.168.1.50
```

---

## 8. ICMP - اختبارات الشبكة

### 🎯 الهدف
استخدام أدوات ICMP لتشخيص واختبار الاتصال بالشبكة.

### 📋 بروتوكول ICMP
Internet Control Message Protocol - بروتوكول رسائل التحكم في الإنترنت
- Layer 3 Protocol
- يُستخدم للتشخيص وإبلاغ الأخطاء
- لا يحمل بيانات المستخدم

### 🔍 الأوامر الأساسية

#### 1. Ping - اختبار الاتصال

**الاستخدام البسيط:**
```bash
ping 8.8.8.8
```

**مع خيارات متقدمة:**
```bash
# إرسال 10 حزم فقط
ping -c 10 8.8.8.8

# تغيير حجم الحزمة
ping -s 1000 8.8.8.8

# تحديد الوقت بين الحزم (ثانية واحدة)
ping -i 1 8.8.8.8
```

**مثال على النتائج:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=11.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=12.1 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 11.8/12.1/12.4/0.3 ms
```

#### 2. Traceroute - تتبع المسار

**على Linux:**
```bash
traceroute 8.8.8.8
```

**على Windows:**
```cmd
tracert 8.8.8.8
```

**مثال على النتائج:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.245 ms  1.123 ms  1.089 ms
 2  10.0.0.1 (10.0.0.1)  8.456 ms  8.234 ms  8.123 ms
 3  172.16.0.1 (172.16.0.1)  15.678 ms  15.456 ms  15.234 ms
 4  * * *
 5  8.8.8.8 (8.8.8.8)  22.345 ms  22.123 ms  21.987 ms
```

**التحليل:**
- كل سطر يمثل موجّه (router) في المسار
- الأرقام الثلاثة: زمن الاستجابة للمحاولات الثلاث
- `* * *`: الموجّه لا يستجيب لطلبات ICMP

#### 3. MTR - تتبع متقدم

```bash
mtr 8.8.8.8
```

يجمع بين ping و traceroute مع إحصائيات مستمرة:
```
                             Packets               Pings
 Host                      Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.1.1             0.0%    10    1.2   1.3   1.1   1.5   0.1
 2. 10.0.0.1                0.0%    10    8.4   8.5   8.1   9.2   0.3
 3. 8.8.8.8                 0.0%    10   22.1  22.3  21.8  23.1   0.4
```

### 📊 أنواع رسائل ICMP

| النوع | الاسم | الاستخدام |
|-------|-------|-----------|
| Type 0 | Echo Reply | رد على Ping |
| Type 3 | Destination Unreachable | الوجهة غير متاحة |
| Type 5 | Redirect | إعادة توجيه المسار |
| Type 8 | Echo Request | طلب Ping |
| Type 11 | Time Exceeded | انتهاء TTL (يستخدمه traceroute) |

### ⚠️ اختبارات متقدمة (بيئة مختبرية فقط)

#### ICMP Flood Test

```bash
# ⚠️ للاختبار في بيئة معزولة فقط
sudo ping -f -s 65500 TARGET_IP
```

**الشرح:**
- `-f`: Flood mode (إرسال سريع جداً)
- `-s 65500`: حجم الحزمة (أقصى حد)
- **يُستخدم لاختبار تحمل الشبكة**

**النتائج المتوقعة:**
```
PING target (192.168.1.50) 65500(65528) bytes of data.
..................................
--- target ping statistics ---
10000 packets transmitted, 9856 received, 1.44% packet loss
```

#### Ping of Death (تاريخي)

```bash
# ⚠️ لا يعمل على الأنظمة الحديثة
ping -s 65510 TARGET_IP
```

**الخلفية:**
- كان يُسبب تعطل الأنظمة القديمة
- الأنظمة الحديثة محمية ضده

### 🛡️ الحماية من هجمات ICMP

#### 1. تحديد معدل ICMP

**على Linux:**
```bash
# تحديد عدد حزم ICMP في الثانية
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

#### 2. على Cisco Router

```cisco
# تحديد معدل ICMP
access-list 100 permit icmp any any echo-reply
access-list 100 deny icmp any any echo-request
interface gi0/0
ip access-group 100 in
rate-limit input access-group 100 1000000 250000 500000 conform-action transmit exceed-action drop
```

#### 3. حظر ICMP نهائياً (غير مستحسن)

```bash
# يُفقد القدرة على التشخيص
iptables -A INPUT -p icmp -j DROP
```

### 📝 حالات الاستخدام العملية

**1. تشخيص انقطاع الاتصال:**
```bash
# خطوة بخطوة
ping 192.168.1.1  # البوابة المحلية
ping 8.8.8.8      # خادم خارجي
traceroute 8.8.8.8  # تحديد نقطة الانقطاع
```

**2. قياس جودة الاتصال:**
```bash
# مراقبة لمدة 5 دقائق
ping -c 300 -i 1 8.8.8.8 | tee ping_results.txt

# تحليل النتائج
grep "time=" ping_results.txt | awk '{print $7}' | sed 's/time=//' | sort -n
```

**3. اختبار MTU (Maximum Transmission Unit):**
```bash
# ابدأ بحجم كبير وقلله تدريجياً
ping -M do -s 1472 8.8.8.8

# إذا فشل، قلل الحجم حتى ينجح
ping -M do -s 1400 8.8.8.8
```

### 💡 نصائح مهمة
- ICMP ليس آمناً 100% - يمكن استخدامه في الهجمات
- بعض الشبكات تحظر ICMP للأمان
- استخدم أدوات بديلة (TCP ping) عند الحاجة
- لا تعتمد على ICMP فقط للمراقبة

---

## 9. SYN Flooding Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم SYN Flood وتأثيره على الخوادم.

### ⚠️ تحذير مهم جداً
```
┌─────────────────────────────────────────────────┐
│  ⚠️  هذا المحتوى لأغراض تعليمية فقط  ⚠️       │
│                                                  │
│  تنفيذ هذا الهجوم على أنظمة حقيقية:           │
│  • جريمة يعاقب عليها القانون                   │
│  • يتطلب إذن خطي مسبق                          │
│  • يُسمح فقط في بيئة مختبرية معزولة           │
└─────────────────────────────────────────────────┘
```

### 📋 نظرة عامة على الهجوم

#### المصافحة الثلاثية الطبيعية (TCP Three-Way Handshake)

```
Client                    Server
  |                         |
  |-------- SYN ---------->|  [1]
  |<----- SYN-ACK ---------|  [2]
  |-------- ACK ---------->|  [3]
  |                         |
  |==== Connection Open ====|
```

#### هجوم SYN Flood

```
Attacker                  Server
  |                         |
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |      (thousands)        |
  |                         |
  |   ❌ No ACK Sent ❌    |
  |                         |
  |                    [Server]
  |                   SYN Queue
  |                   ┌─────────┐
  |                   │ SYN #1  │ ⏳
  |                   │ SYN #2  │ ⏳
  |                   │ SYN #3  │ ⏳
  |                   │ SYN #... │ ⏳
  |                   │ FULL!   │ 🔴
  |                   └─────────┘
```

### 🔬 التطبيق العملي (بيئة مختبرية فقط)

#### البيئة المطلوبة
- آلة افتراضية للمهاجم (Kali Linux)
- خادم اختبار معزول
- شبكة محلية منعزلة تماماً

#### الأداة: hping3

**التثبيت:**
```bash
sudo apt update
sudo apt install hping3
```

**الهجوم الأساسي:**
```bash
sudo hping3 -S -p 80 --flood TARGET_IP
```

**الشرح:**
- `-S`: إرسال حزم SYN فقط
- `-p 80`: استهداف المنفذ 80 (HTTP)
- `--flood`: إرسال أسرع ما يمكن (بدون انتظار)
- `TARGET_IP`: عنوان الخادم المستهدف

**هجوم متقدم مع عناوين مزيفة:**
```bash
sudo hping3 -S -p 80 --flood --rand-source TARGET_IP
```

- `--rand-source`: عناوين IP مصدر عشوائية
- يُصعّب تتبع المهاجم
- يتجاوز بعض آليات الحماية البسيطة

### 📊 المراقبة على الخادم المستهدف

#### عرض الاتصالات نصف المفتوحة

```bash
netstat -ant | grep SYN_RECV
```

**مثال على النتائج أثناء الهجوم:**
```
tcp    0    0 192.168.1.50:80    10.0.0.100:45123    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.101:45124    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.102:45125    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.103:45126    SYN_RECV
... (مئات أو آلاف السطور)
```

#### مراقبة استهلاك الموارد

```bash
# عرض قائمة الانتظار
ss -tan | grep SYN-RECV | wc -l

# مراقبة الذاكرة
free -h

# مراقبة CPU
top
```

**مؤشرات الهجوم:**
```
SYN-RECV connections: 5000+  🔴 (طبيعي: < 100)
Memory usage: 95%             🔴 (طبيعي: < 70%)
CPU load: 100%                🔴 (طبيعي: < 50%)
```

### 🛡️ آليات الحماية

#### 1. SYN Cookies

**تفعيل SYN Cookies على Linux:**
```bash
# التحقق من الحالة الحالية
cat /proc/sys/net/ipv4/tcp_syncookies

# التفعيل (1 = enabled)
sudo sysctl -w net.ipv4.tcp_syncookies=1

# جعله دائماً
echo "net.ipv4.tcp_syncookies = 1" | sudo tee -a /etc/sysctl.conf
```

**كيف تعمل:**
```
بدلاً من تخزين الاتصال في الذاكرة:
1. الخادم يحسب قيمة مشفرة (cookie)
2. يُرسلها في SYN-ACK
3. لا يحفظ شيء في الذاكرة
4. عند استلام ACK، يتحقق من الـ cookie
5. إن صح، يُكمل الاتصال

النتيجة: لا استنزاف للذاكرة ✅
```

#### 2. تحديد معدل الاتصالات الجديدة

**باستخدام iptables:**
```bash
# السماح بـ 5 اتصالات جديدة فقط في الثانية من نفس IP
iptables -A INPUT -p tcp --syn -m limit --limit 5/s --limit-burst 10 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP
```

**باستخدام nftables:**
```bash
nft add rule ip filter input tcp flags syn ct state new limit rate 5/second accept
```

#### 3. زيادة حجم قائمة SYN

```bash
# زيادة الحد الأقصى للاتصالات نصف المفتوحة
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# تقليل وقت الانتظار
sudo sysctl -w net.ipv4.tcp_synack_retries=2
```

#### 4. استخدام Firewall متقدم

**Cisco ASA:**
```cisco
! تحديد عدد الاتصالات المتزامنة
access-list OUTSIDE_IN extended permit tcp any host 203.0.113.50 eq www
class-map SYN-ATTACK
 match access-list OUTSIDE_IN
policy-map OUTSIDE_POLICY
 class SYN-ATTACK
  set connection conn-max 100 embryonic-conn-max 50
service-policy OUTSIDE_POLICY interface outside
```

**pfSense/OPNsense:**
- تفعيل "SYN Proxy"
- ضبط "Maximum States" و "Maximum Source Nodes"

#### 5. استخدام CDN/DDoS Protection

خدمات مثل:
- Cloudflare
- AWS Shield
- Akamai

تقوم بـ:
- امتصاص الهجمات الضخمة
- توزيع الحمل
- فلترة الحركة الخبيثة

### 📈 التأثير على الخادم

#### مراحل الهجوم

**المرحلة 1: البداية (0-30 ثانية)**
```
SYN Queue: 20% full
Response Time: 100ms → 500ms
```

**المرحلة 2: التصعيد (30-60 ثانية)**
```
SYN Queue: 80% full
Response Time: 500ms → 3000ms
New connections: بطيئة جداً
```

**المرحلة 3: الانهيار (60+ ثانية)**
```
SYN Queue: 100% full
Response Time: timeout
New connections: مرفوضة بالكامل
Service: غير متاح 🔴
```

### 🔍 كشف الهجوم

#### علامات التحذير

```bash
# ارتفاع مفاجئ في SYN_RECV
watch -n 1 'netstat -ant | grep SYN_RECV | wc -l'

# تنبيه عند تجاوز حد معين
while true; do
  COUNT=$(netstat -ant | grep SYN_RECV | wc -l)
  if [ $COUNT -gt 1000 ]; then
    echo "⚠️ Possible SYN Flood: $COUNT connections"
  fi
  sleep 5
done
```

#### تحليل السجلات

```bash
# عرض أكثر IP مصدرية نشاطاً
netstat -ant | grep SYN_RECV | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -10
```

### 💡 الدروس المستفادة

**للمدافعين:**
1. ✅ فعّل SYN Cookies دائماً
2. ✅ راقب معدلات الاتصال بشكل مستمر
3. ✅ استخدم Firewall مع إمكانيات DDoS protection
4. ✅ احتفظ بخطة استجابة جاهزة

**للمهاجمين الأخلاقيين:**
1. 📝 احصل على إذن خطي قبل الاختبار
2. 🔒 استخدم بيئة معزولة تماماً
3. ⏰ حدد نافذة زمنية محدودة للاختبار
4. 📊 وثّق النتائج بشكل احترافي

---

## 10. UDP Flood Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم UDP Flood والفرق بينه وبين SYN Flood.

### ⚠️ تحذير قانوني
```
┌──────────────────────────────────────────────────┐
│  🚨 هذا المحتوى للتعليم والأبحاث الأمنية فقط  │
│                                                   │
│  القيام بهذا الهجوم دون إذن هو:                │
│  • جريمة إلكترونية                              │
│  • يُعاقب عليها بالسجن والغرامة                │
│  • يُسمح فقط في بيئة مختبرية مملوكة لك         │
└──────────────────────────────────────────────────┘
```

### 📋 الفرق بين TCP و UDP

| الميزة | TCP | UDP |
|--------|-----|-----|
| الاتصال | Connection-oriented | Connectionless |
| المصافحة | نعم (3-way handshake) | لا |
| الموثوقية | مضمون الوصول | غير مضمون |
| السرعة | أبطأ | أسرع |
| الترتيب | مرتب | غير مرتب |

### 🔬 آلية هجوم UDP Flood

```
Attacker                    Target Server
  |                              |
  |------- UDP Packet ---------->| Port 53 (DNS)
  |------- UDP Packet ---------->| Port 161 (SNMP)
  |------- UDP Packet ---------->| Port 12345 (Random)
  |------- UDP Packet ---------->| Port 9999 (Closed)
  |       (thousands/sec)        |
  |                              |
  |                         [Server Process]
  |                         1. تلقي الحزمة
  |                         2. فحص المنفذ
  |                         3. المنفذ مغلق؟
  |                         4. إرسال ICMP Port Unreachable
  |                              |
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |                              |
  |                    [Result: Resource Exhaustion]
  |                    • CPU: 100%
  |                    • Bandwidth: Saturated
  |                    • Service: Unavailable
```

### 🔬 التطبيق العملي (بيئة مختبرية معزولة فقط)

#### البيئة المطلوبة
- شبكة مختبرية معزولة تماماً
- خادم اختبار مخصص
- إذن خطي موثق

#### استخدام hping3

**الهجوم الأساسي:**
```bash
sudo hping3 --udp -p 53 --flood TARGET_IP
```

**الشرح:**
- `--udp`: استخدام بروتوكول UDP
- `-p 53`: استهداف منفذ DNS
- `--flood`: إرسال بأقصى سرعة ممكنة

**هجوم مع منافذ عشوائية:**
```bash
sudo hping3 --udp --rand-dest --flood TARGET_IP
```

- `--rand-dest`: منافذ وجهة عشوائية
- يُضعف الخادم أكثر (يفحص كل منفذ)

**هجوم مع حجم حزمة كبير:**
```bash
sudo hping3 --udp -p 53 --flood -d 65500 TARGET_IP
```

- `-d 65500`: حجم البيانات (أقصى حد لـ UDP)
- يستهلك bandwidth أكثر

### 📊 المراقبة على الخادم المستهدف

#### عرض إحصائيات UDP

```bash
netstat -s | grep UDP
```

**مثال على النتائج أثناء الهجوم:**
```
UDP:
    125847 packets received
    98234 packets to unknown port received
    89123 packet receive errors
    45678 packets sent
    RcvbufErrors: 12345
    SndbufErrors: 0
    InErrors: 89123
```

#### مراقبة الحزم في الوقت الفعلي

```bash
# عرض معدل حزم UDP
watch -n 1 'netstat -su | grep "packets received"'

# مراقبة منافذ محددة
tcpdump -i eth0 udp port 53 -nn
```

**مؤشرات الهجوم:**
```
UDP packets/sec: 50,000+     🔴 (طبيعي: < 1,000)
Port Unreachable: 40,000+    🔴 (طبيعي: < 10)
Bandwidth usage: 90%+        🔴 (طبيعي: < 50%)
CPU usage: 100%              🔴 (طبيعي: < 30%)
```

#### تحديد أكثر المنافذ المستهدفة

```bash
tcpdump -i eth0 -nn udp -c 1000 | awk '{print $5}' | cut -d'.' -f5 | sort | uniq -c | sort -nr | head -10
```

### 🛡️ آليات الحماية

#### 1. Rate Limiting على Linux

```bash
# تحديد معدل حزم UDP من كل IP
iptables -A INPUT -p udp -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p udp -j DROP

# حماية منافذ محددة
iptables -A INPUT -p udp --dport 53 -m limit --limit 50/s -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j DROP
```

#### 2. تعطيل ICMP Port Unreachable

```bash
# منع إرسال رسائل Port Unreachable
iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP

# أو عبر sysctl
sysctl -w net.ipv4.icmp_ignore_bogus_error_responses=1
```

**الفائدة:**
- يوفر موارد CPU
- يمنع استنزاف bandwidth بالردود

#### 3. حماية على مستوى الخادم

```bash
# زيادة حجم buffer للـ UDP
sysctl -w net.core.rmem_max=26214400
sysctl -w net.core.rmem_default=26214400

# تحسين معالجة الحزم
sysctl -w net.core.netdev_max_backlog=5000
```

#### 4. Firewall Rules على Cisco

```cisco
! تحديد معدل UDP
access-list 110 permit udp any host 203.0.113.50 eq domain
access-list 110 deny udp any any

interface GigabitEthernet0/0
 ip access-group 110 in
 rate-limit input access-group 110 10000000 1000000 2000000 conform-action transmit exceed-action drop
```

#### 5. استخدام fail2ban

```bash
# تثبيت fail2ban
sudo apt install fail2ban

# إنشاء قاعدة لـ UDP flood
sudo nano /etc/fail2ban/filter.d/udp-flood.conf
```

**محتوى الملف:**
```ini
[Definition]
failregex = .*kernel.*UDP.*SRC=<HOST>.*
ignoreregex =
```

**التفعيل:**
```ini
# /etc/fail2ban/jail.local
[udp-flood]
enabled = true
filter = udp-flood
logpath = /var/log/syslog
maxretry = 100
findtime = 60
bantime = 3600
```

### 📈 المنافذ الشائعة المستهدفة

| المنفذ | الخدمة | سبب الاستهداف |
|--------|---------|---------------|
| 53 | DNS | حيوي للشبكة، يولد استجابات |
| 123 | NTP | يمكن تضخيمه (Amplification) |
| 161 | SNMP | إدارة، غالباً مفتوح |
| 1900 | SSDP | UPnP، شائع في الأجهزة المنزلية |
| 19 | Chargen | يولد بيانات كبيرة |
| 111 | RPC | يمكن استغلاله للتضخيم |

### 🎯 UDP Amplification Attack

**المفهوم:**
المهاجم يرسل طلبات صغيرة مع عنوان IP مزيف (IP Spoofing)، والخادم يرد بردود كبيرة للضحية.

```
Attacker                DNS Server              Victim
  |                         |                      |
  |-- Small Query (64B) -->|                      |
  |  (Spoofed Source IP)   |                      |
  |                         |                      |
  |                         |-- Large Reply ------>|
  |                         |    (3000B)           |
  |                         |                      |
  
Amplification Factor: 3000 / 64 = 47x
```

**مثال عملي على DNS:**
```bash
# طلب صغير
dig ANY example.com @8.8.8.8

# يُرجع رد كبير جداً يحتوي على:
# - A records
# - AAAA records
# - MX records
# - TXT records
# - NS records
# وغيرها...
```

### 🛡️ الحماية من UDP Amplification

#### 1. BCP38 - منع IP Spoofing

```cisco
! على Router الحدودي
interface GigabitEthernet0/0
 description To Internet
 ip verify unicast source reachable-via rx
```

#### 2. تقييد الاستعلامات الخارجية

**على DNS Server:**
```bash
# BIND Configuration
options {
    allow-query { localhost; 192.168.0.0/16; };
    recursion no;
};
```

#### 3. Rate Limiting على DNS

```bash
# BIND RRL (Response Rate Limiting)
rate-limit {
    responses-per-second 5;
    window 5;
};
```

### 📊 مقارنة بين SYN Flood و UDP Flood

| الميزة | SYN Flood | UDP Flood |
|--------|-----------|-----------|
| البروتوكول | TCP | UDP |
| الهدف | استنزاف Connection Queue | استنزاف CPU/Bandwidth |
| الكشف | سهل (SYN_RECV connections) | متوسط |
| التضخيم | لا يوجد | ممكن (Amplification) |
| الحماية | SYN Cookies | Rate Limiting |
| التأثير | Denial of Service | Bandwidth Saturation |

### 🔍 كشف UDP Flood

#### سكريبت مراقبة بسيط

```bash
#!/bin/bash
# udp_monitor.sh

THRESHOLD=1000
INTERVAL=5

while true; do
    UDP_COUNT=$(netstat -su | grep "packets received" | awk '{print $1}')
    sleep $INTERVAL
    UDP_COUNT_NEW=$(netstat -su | grep "packets received" | awk '{print $1}')
    
    RATE=$(( ($UDP_COUNT_NEW - $UDP_COUNT) / $INTERVAL ))
    
    if [ $RATE -gt $THRESHOLD ]; then
        echo "⚠️ [$(date)] High UDP traffic detected: $RATE packets/sec"
        # إرسال تنبيه
        echo "High UDP traffic: $RATE pps" | mail -s "UDP Flood Alert" admin@example.com
    fi
done
```

#### استخدام tshark للتحليل

```bash
# مراقبة وتسجيل حركة UDP
tshark -i eth0 -f "udp" -w udp_traffic.pcap

# تحليل الملف
tshark -r udp_traffic.pcap -q -z io,phs
```

### 💡 نصائح للحماية

**للمسؤولين:**
1. ✅ فعّل Rate Limiting على جميع الخدمات
2. ✅ راقب استخدام bandwidth بشكل مستمر
3. ✅ قيّد الوصول للخدمات (Whitelist)
4. ✅ استخدم CDN/DDoS Protection للمواقع المهمة
5. ✅ عطّل الخدمات غير المستخدمة

**للمختبرين:**
1. 📝 وثّق جميع الاختبارات
2. 🔒 استخدم شبكة معزولة 100%
3. ⏰ حدد فترة زمنية محدودة
4. 📊 راقب التأثير على الموارد
5. 🔄 أعد النظام لحالته الطبيعية بعد الاختبار

---

## 11. Nmap Port Scanning

### 🎯 الهدف
استخدام Nmap بشكل احترافي لفحص الشبكات واكتشاف الخدمات.

### 📋 السيناريو التطبيقي
فحص خادم في بيئة اختبار: `192.168.80.131`

### 🔍 الأمر الأساسي

```bash
nmap -v -sN 192.168.80.131
```

**الشرح:**
- `-v`: Verbose mode (إظهار تفاصيل أكثر)
- `-sN`: TCP Null Scan (فحص خفي)

### 📊 مثال على النتائج

```
Starting Nmap 7.94 ( https://nmap.org )
NSE: Loaded 156 scripts for scanning.
Initiating Ping Scan at 15:23:45
Scanning 192.168.80.131 [4 ports]
Completed Ping Scan at 15:23:45, 0.05s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host at 15:23:45
Completed Parallel DNS resolution of 1 host at 15:23:45, 0.01s elapsed
Initiating TCP Null Scan at 15:23:45
Scanning 192.168.80.131 [1000 ports]
Discovered open port 21/tcp on 192.168.80.131
Discovered open port 23/tcp on 192.168.80.131
Discovered open port 80/tcp on 192.168.80.131
Discovered open port 25/tcp on 192.168.80.131
Discovered open port 139/tcp on 192.168.80.131
Discovered open port 445/tcp on 192.168.80.131
Completed TCP Null Scan at 15:23:47, 2.15s elapsed (1000 total ports)
Nmap scan report for 192.168.80.131
Host is up (0.0012s latency).
Not shown: 994 closed ports
PORT    STATE         SERVICE
21/tcp  open|filtered ftp
23/tcp  open|filtered telnet
25/tcp  open|filtered smtp
80/tcp  open|filtered http
139/tcp open|filtered netbios-ssn
445/tcp open|filtered microsoft-ds

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 2.25 seconds
           Raw packets sent: 1004 (44.176KB) | Rcvd: 1001 (40.052KB)
```

### 🎯 تحليل المنافذ المكتشفة

| المنفذ | الخدمة | الخطر | التوصية |
|--------|---------|-------|---------|
| **21** | FTP | 🔴 عالي | نقل بيانات بدون تشفير، استخدم SFTP |
| **23** | Telnet | 🔴 عالي جداً | إدارة بدون تشفير، استخدم SSH |
| **25** | SMTP | 🟡 متوسط | بريد إلكتروني، تأكد من التشفير |
| **80** | HTTP | 🟡 متوسط | ويب بدون تشفير، استخدم HTTPS |
| **139** | NetBIOS | 🔴 عالي | عرضة لهجمات، أغلقه إن لم يكن ضرورياً |
| **445** | SMB | 🔴 عالي جداً | هدف شائع (WannaCry، EternalBlue) |

### 🔬 أنواع فحص Nmap

#### 1. TCP Connect Scan (-sT)
```bash
nmap -sT 192.168.80.131
```
- يكمل المصافحة الثلاثية
- لا يحتاج صلاحيات root
- سهل الكشف في السجلات

#### 2. SYN Stealth Scan (-sS)
```bash
sudo nmap -sS 192.168.80.131
```
- يرسل SYN فقط
- لا يكمل الاتصال (خفي)
- الأكثر شيوعاً

#### 3. NULL Scan (-sN)
```bash
sudo nmap -sN 192.168.80.131
```
- لا يُرسل أي flags
- يتجاوز بعض Firewalls
- لا يعمل مع Windows

#### 4. FIN Scan (-sF)
```bash
sudo nmap -sF 192.168.80.131
```
- يرسل FIN flag فقط
- مشابه لـ NULL scan

#### 5. Xmas Scan (-sX)
```bash
sudo nmap -sX 192.168.80.131
```
- يرسل FIN, PSH, URG flags
- سُمي "Xmas" لأن الـ flags مضاءة مثل شجرة الكريسماس

### 🎯 فحص متقدم مع كشف الإصدارات

```bash
sudo nmap -sV -O -A 192.168.80.131
```

**الشرح:**
- `-sV`: كشف إصدارات الخدمات
- `-O`: تخمين نظام التشغيل
- `-A`: فحص شامل (Script scanning + OS detection + Traceroute)

**مثال على النتائج:**
```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
23/tcp  open  telnet      Linux telnetd
25/tcp  open  smtp        Postfix smtpd
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian

OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop

Service Info: Host: metasploitable.localdomain; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### 🔍 كشف الثغرات

```bash
sudo nmap --script vuln 192.168.80.131
```

**مثال على اكتشاف ثغرة:**
```
PORT   STATE SERVICE
21/tcp open  ftp
| ftp-vsftpd-backdoor: 
|   VULNERABLE:
|   vsFTPd version 2.3.4 backdoor
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2011-2523
|     Risk factor: High  CVSSv2: 10.0 (HIGH) (AV:N/AC:L/AU:N/C:C/I:C/A:C)
|       vsFTPd version 2.3.4 backdoor, this was reported on 2011-07-04.
```

### 🎯 OS Fingerprinting

```bash
sudo nmap -O 192.168.80.131
```

**آلية العمل:**
1. إرسال حزم TCP/UDP/ICMP خاصة
2. تحليل الردود
3. مقارنة مع قاعدة بيانات الأنظمة
4. تخمين النظام بنسبة دقة

**مثال:**
```
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
```

### 📝 حفظ النتائج

```bash
# نص عادي
nmap -oN output.txt 192.168.80.131

# XML (للمعالجة البرمجية)
nmap -oX output.xml 192.168.80.131

# Grepable (للبحث السريع)
nmap -oG output.grep 192.168.80.131

# جميع الصيغ
nmap -oA output_all 192.168.80.131
```

### 🛡️ الحماية من Nmap

#### 1. استخدام IDS/IPS

**Snort Rule لكشف Nmap:**
```
alert tcp any any -> $HOME_NET any (msg:"SCAN nmap XMAS"; flags:FPU; reference:arachnids,30; classtype:attempted-recon; sid:1228; rev:7;)
```

#### 2. تقييد الوصول

```bash
# iptables - السماح فقط بـ IPs معروفة
iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -j DROP
```

#### 3. Port Knocking

```bash
# تثبيت knockd
sudo apt install knockd

# التكوين
sudo nano /etc/knockd.conf
```

**المفهوم:**
- المنافذ مغلقة بشكل افتراضي
- "طرق" على منافذ محددة بترتيب معين
- المنفذ المطلوب يُفتح مؤقتاً

#### 4. تغيير المنافذ الافتراضية

```bash
# SSH من 22 إلى 2222
sudo nano /etc/ssh/sshd_config
# Port 2222

# HTTP من 80 إلى 8080
# في Apache config
Listen 8080
```

### 💡 نصائح للاستخدام الأخلاقي

**قبل الفحص:**
1. ✅ احصل على إذن خطي
2. ✅ حدد نطاق الاختبار
3. ✅ اتفق على التوقيت
4. ✅ وثّق جميع الخطوات

**أثناء الفحص:**
1. 🔍 استخدم `-T2` أو `-T3` (سرعات معتدلة)
2. 📊 راقب تأثير الفحص على الشبكة
3. 💾 سجّل جميع النتائج
4. ⏸️ توقف إذا لاحظت مشاكل

**بعد الفحص:**
1. 📝 أعد تقرير مفصل
2. 🔒 قدّم توصيات أمنية
3. 🤝 ناقش النتائج مع المسؤولين
4. 🔐 احذف البيانات الحساسة بشكل آمن

---

## 12. John the Ripper - كسر كلمات المرور

### 🎯 الهدف التعليمي
فهم آليات كسر كلمات المرور وأهمية استخدام كلمات مرور قوية.

### ⚠️ تحذير أخلاقي وقانوني

```
┌──────────────────────────────────────────────────────┐
│  ⚠️  استخدام هذه الأداة على حسابات لا تملكها    │
│      يُعتبر جريمة إلكترونية                         │
│                                                        │
│  الاستخدامات المشروعة فقط:                          │
│  ✅ اختبار كلمات المرور الخاصة بك                  │
│  ✅ تدقيق أمني بإذن خطي                            │
│  ✅ استرجاع كلمات المرور المنسية (ملفاتك)          │
│  ✅ بيئة مختبرية تعليمية                           │
└──────────────────────────────────────────────────────┘
```

### 📋 نظرة عامة

**John the Ripper** هو أداة مفتوحة المصدر لاختبار قوة كلمات المرور عبر:
- **Dictionary Attack:** استخدام قوائم كلمات معروفة
- **Brute Force:** تجربة جميع التوافيق الممكنة
- **Rainbow Tables:** جداول hash محسوبة مسبقاً

### 🔧 التثبيت

```bash
# على Ubuntu/Debian
sudo apt update
sudo apt install john

# على Kali Linux (مُثبت مسبقاً)
john --version
```

### 📊 فهم التجزئة (Hashing)

```
Plain Text → Hash Function → Hash

مثال:
"password123" → MD5 → "482c811da5d5b4bc6d497ffa98491e38"
"password123" → SHA256 → "ef92b778bafe771e89245b89ecbc08a4..."
```

**خصائص Hash:**
- ✅ اتجاه واحد (لا يمكن عكسه)
- ✅ نفس المدخل = نفس المخرج دائماً
- ✅ تغيير صغير في المدخل = تغيير كبير في المخرج

### 🔍 السيناريو التطبيقي

#### الخطوة 1: استخراج كلمات المرور

**على Linux:**
```bash
# ملف /etc/shadow يحتوي على hashes
sudo cat /etc/shadow

# مثال على سطر:
user1:$6$saltsalt$hashhashhashhash...:18500:0:99999:7:::
```

**تنسيق الحقول:**
```
username:$algorithm$salt$hash:last_change:min:max:warn:inactive:expire
```

**أنواع الخوارزميات:**
- `$1# تدريبات عملية في أمن الشبكات
## Network Security - Practical Labs

---

## 📑 جدول المحتويات

1. [NAT/PAT - ترجمة عناوين الشبكة](#1-natpat---ترجمة-عناوين-الشبكة)
2. [ACL - قوائم التحكم بالوصول](#2-acl---قوائم-التحكم-بالوصول)
3. [FTP وآلية Established](#3-ftp-وآلية-established)
4. [WAF و Load Balancer](#4-waf-و-load-balancer)
5. [Cisco Firepower - تحليل البرمجيات الخبيثة](#5-cisco-firepower---تحليل-البرمجيات-الخبيثة)
6. [ARP - بروتوكول تحليل العناوين](#6-arp---بروتوكول-تحليل-العناوين)
7. [Nmap - مسح الشبكات](#7-nmap---مسح-الشبكات)
8. [ICMP - اختبارات الشبكة](#8-icmp---اختبارات-الشبكة)
9. [SYN Flooding Attack](#9-syn-flooding-attack)
10. [UDP Flood Attack](#10-udp-flood-attack)
11. [Nmap Port Scanning](#11-nmap-port-scanning)
12. [John the Ripper - كسر كلمات المرور](#12-john-the-ripper---كسر-كلمات-المرور)
13. [DHCP Snooping](#13-dhcp-snooping)

---

## 1. NAT/PAT - ترجمة عناوين الشبكة

### 🎯 الهدف
فهم آلية عمل NAT/PAT في ترجمة العناوين الخاصة إلى عناوين عامة.

### 📋 السيناريو
جهاز داخلي بعنوان `10.0.0.5` يريد تصفح الإنترنت عبر جدار ناري يستخدم NAT.

### 🔄 خطوات العمل

1. **الطلب الصادر:**
   - الجهاز الداخلي `10.0.0.5` يرسل طلب HTTP
   - الجدار الناري يترجم العنوان إلى `203.0.113.2:10500`
   - العنوان `203.0.113.2` هو العنوان العام
   - المنفذ `10500` يُستخدم لتمييز الاتصال

2. **الرد الوارد:**
   - عند وصول الرد من الإنترنت إلى `203.0.113.2:10500`
   - يعيد NAT الترجمة العكسية إلى `10.0.0.5` بالمنفذ الأصلي
   - يتم تسليم البيانات للجهاز الداخلي

### ✅ الفائدة
- مئات الأجهزة يمكنها مشاركة عنوان عام واحد
- لا يوجد تضارب بفضل استخدام أرقام المنافذ (Port Numbers)
- توفير في عناوين IPv4 العامة

### 📝 ملاحظات
- PAT (Port Address Translation) هو نوع من NAT
- يُسمى أيضاً NAT Overload
- الجدول الداخلي يحتفظ بالربط بين العناوين الداخلية والمنافذ

---

## 2. ACL - قوائم التحكم بالوصول

### 🎯 الهدف
تطبيق سياسات أمنية باستخدام قوائم التحكم بالوصول (Access Control Lists).

### 📋 السيناريو
لديك:
- **Web Server** في DMZ (Demilitarized Zone)
- **Database Server** داخل الشبكة الداخلية

> 💡 **DMZ:** منطقة منزوعة السلاح، وهي منطقة بين الشبكة الداخلية والإنترنت لتقليل المخاطر الأمنية.

### 🔧 التكوين العملي

```cisco
access-list 100 permit tcp any 10.10.10.10 eq www
access-list 100 permit tcp any 10.10.10.10 eq 443
access-list 100 permit tcp any 10.10.10.12 eq ftp
access-list 100 permit tcp any 10.10.10.12 eq ftp-data
access-list 100 deny ip any any log

interface gi0/1
ip access-group 100 in
```

### 📊 شرح الأوامر

| الأمر | الوظيفة |
|------|---------|
| `permit tcp any 10.10.10.10 eq www` | السماح بالوصول لخادم الويب عبر HTTP (منفذ 80) |
| `permit tcp any 10.10.10.10 eq 443` | السماح بالوصول عبر HTTPS (منفذ 443) |
| `permit tcp any 10.10.10.12 eq ftp` | السماح بـ FTP Control (منفذ 21) |
| `permit tcp any 10.10.10.12 eq ftp-data` | السماح بـ FTP Data (منفذ 20) |
| `deny ip any any log` | حظر أي حركة أخرى مع التسجيل |
| `ip access-group 100 in` | تطبيق القائمة على الواجهة |

### 🎯 الهدف الأمني
- السماح بالوصول فقط للمنافذ المطلوبة
- منع الوصول المباشر لخادم قاعدة البيانات
- تسجيل المحاولات المشبوهة
- تقليل نطاق الهجوم (Attack Surface)

### ⚠️ ملاحظات مهمة
- الوصول لقاعدة البيانات يجب أن يتم فقط من خلال خادم الويب
- ترتيب القواعد مهم جداً (من الأعلى للأسفل)
- القاعدة الضمنية في النهاية: `deny any any`

---

## 3. FTP وآلية Established

### 🎯 الهدف
فهم الفرق بين FTP Passive و Active Mode وتأثيره على ACL.

### 📋 بنية FTP
بروتوكول FTP يستخدم قناتين منفصلتين:
- **Port 21:** قناة التحكم (Control Channel)
- **Port 20:** قناة البيانات (Data Channel)

### 🔄 FTP Passive Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Random High Port (Data)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يطلب وضع Passive
3. الخادم يرد بمنفذ عشوائي عالي
4. العميل يفتح اتصال البيانات نحو هذا المنفذ
5. كل الردود تحتوي على بت ACK ← **مسموح بها** ✅

### 🔄 FTP Active Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Port 20 (Data initiated by server)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يخبر الخادم بمنفذ معين لاستقبال البيانات
3. **الخادم يبدأ اتصال البيانات** من المنفذ 20
4. أول حزمة من الخادم تحتوي على SYN فقط ← **تُرفض** 🚫

### ⚠️ المشكلة مع Established

```cisco
access-list 110 permit tcp any any established
```

هذه القاعدة تسمح فقط بالحزم التي تحتوي على:
- بت ACK
- أو بت RST

**النتيجة:**
- في Active Mode، الخادم يرسل SYN أولاً
- هذه الحزمة لا تحقق شرط "established"
- الاتصال يفشل 🚫

### ✅ الحل الأمثل
استخدام جدار ناري حقيقي (Stateful Firewall) قادر على:
- تتبع الجلسات النشطة
- فهم البروتوكولات المعقدة مثل FTP
- السماح بالاتصالات الديناميكية تلقائياً

### 📝 الخلاصة
- `established` لا يكفي للبروتوكولات المعقدة
- FTP Passive Mode أكثر توافقاً مع الجدران النارية
- Stateful Firewalls ضرورية للبيئات الإنتاجية

---

## 4. WAF و Load Balancer

### 🎯 الهدف
بناء بنية تحتية آمنة وقابلة للتوسع لموقع ويب.

### 📋 السيناريو
موقع ويب يحتوي على عدة خوادم Web Servers خلف Load Balancer، مع حماية من هجمات XSS.

### 🏗️ البنية المعمارية

```
[Internet]
    |
    ↓
[WAF - Web Application Firewall]
    |
    ↓
[Load Balancer]
    |
    ↓------------------↓------------------↓------------------↓
[Web Server 1]   [Web Server 2]   [Web Server 3]   [Web Server 4]
```

### ⚖️ مهام Load Balancer

**خوارزمية التوزيع:**
- **Round Robin:** توزيع متساوٍ على جميع الخوادم
- دورة منتظمة: Server 1 → Server 2 → Server 3 → Server 4 → تكرار

**إدارة الأعطال:**
1. مراقبة صحة الخوادم (Health Checks)
2. عند فشل أحد الخوادم:
   - إزالته من قائمة التوزيع تلقائياً
   - تحويل طلباته إلى الخوادم السليمة
3. عند عودة الخادم: إعادة إدراجه تلقائياً

### 🛡️ مهام WAF (كـ Reverse Proxy)

**الحماية من XSS:**
```javascript
// مثال على محاولة هجوم XSS
<script>alert('XSS Attack')</script>
```

**آلية الحماية:**
1. فحص كل طلب HTTP قبل وصوله للخوادم
2. كشف الشيفرات الخبيثة:
   - JavaScript مشبوه
   - SQL Injection
   - Command Injection
3. منع الطلبات المشبوهة فوراً ⛔
4. منع تسريب البيانات الحساسة في الاستجابات

### 🔍 سياسات الأمان

| نوع الحماية | الوظيفة |
|-------------|---------|
| XSS Prevention | منع حقن JavaScript |
| SQL Injection | حماية قواعد البيانات |
| CSRF Protection | منع الطلبات المزورة |
| Rate Limiting | الحد من معدل الطلبات |
| IP Blacklisting | حظر IP المشبوهة |

### ✅ المزايا المحققة

**الأداء:**
- ⚡ توزيع الأحمال بشكل متوازن
- 🔄 تحمل الأعطال (Fault Tolerance)
- 📈 قابلية التوسع الأفقي

**الأمان:**
- 🛡️ حماية متقدمة من WAF
- 🔒 فحص جميع الطلبات
- 📝 تسجيل المحاولات المشبوهة
- 🚫 حظر الهجمات في الوقت الحقيقي

### 📊 مثال على سجلات WAF

```
2025-10-26 10:15:32 | BLOCKED | XSS Attempt | IP: 192.168.1.100
2025-10-26 10:16:45 | BLOCKED | SQL Injection | IP: 10.0.0.50
2025-10-26 10:17:12 | ALLOWED | Normal Request | IP: 172.16.0.25
```

---

## 5. Cisco Firepower - تحليل البرمجيات الخبيثة

### 🎯 الهدف
فهم كيفية عمل أنظمة كشف البرمجيات الخبيثة المتقدمة وتتبع انتشارها.

### 📋 السيناريو
ملف باسم `tool.exe` تم اكتشافه ضمن حركة المرور على الشبكة.

### 🔍 مراحل الكشف والتحليل

#### المرحلة 1: الفحص الأولي
1. **اعتراض الملف:** جهاز Cisco Firepower يفحص جميع الملفات المارة
2. **حساب البصمة:** استخراج بصمة SHA-256 للملف
   ```
   SHA-256: a3f7b8c9d2e1f4a6b5c8d7e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
   ```
3. **المقارنة:** مقارنة البصمة مع قاعدة بيانات التهديدات

#### المرحلة 2: التصنيف
```
Status: ⚠️ MALWARE DETECTED
Threat Name: Generic.Trojan.Downloader
Threat Score: 95/100 (High)
Category: Trojan
First Seen: 2025-10-25 14:30:22
```

#### المرحلة 3: تحديد الأجهزة المصابة
الأجهزة التالية تم تحديدها كمصابة:
- 🖥️ `192.168.10.90`
- 🖥️ `192.168.133.50`

### 📈 Trajectory Map (خريطة المسار)

```
Time →
Device 1: --------[*]==========[*]-------
                   ↓            ↓
Device 2: --------[*]----[*]----[*]------
                         ↓      ↓
Device 3: ---------------[*]=====[*]-----
```

**رموز الخريطة:**
- `[*]` نقطة انتقال الملف
- `=` فترة نشاط الملف على الجهاز
- `↓` اتجاه انتقال العدوى (من جهاز لآخر)
- `-` فترة خمول

### 📊 جدول الأحداث (Events Table)

| الوقت | الجهاز المصدر | الجهاز الهدف | الحدث | الحالة |
|------|---------------|--------------|--------|--------|
| 14:30:22 | External | 192.168.10.90 | File Download | Infected |
| 14:35:18 | 192.168.10.90 | 192.168.133.50 | File Transfer | Infected |
| 14:42:33 | 192.168.133.50 | Network Share | Propagation Attempt | Blocked |

### 🎯 استخدامات Trajectory Map

1. **تحديد المصدر الأولي:**
   - أول جهاز تلقى الملف
   - نقطة الدخول إلى الشبكة

2. **تتبع مسار الانتشار:**
   - الأجهزة الوسيطة
   - طرق الانتقال المستخدمة

3. **تقييم نطاق الإصابة:**
   - عدد الأجهزة المصابة
   - الأقسام المتأثرة

4. **التخطيط للاستجابة:**
   - أولوية المعالجة
   - خطة العزل والإزالة

### 🚨 خطة الاستجابة للحادث

**الخطوات الفورية:**
1. ✅ عزل الأجهزة المصابة عن الشبكة
2. ✅ حظر بصمة الملف على جميع الأجهزة
3. ✅ فحص شامل للأجهزة المجاورة
4. ✅ تحليل سجلات النشاط

**الإجراءات التصحيحية:**
1. 🔧 إزالة البرمجية الخبيثة
2. 🔧 تحديث تعريفات مكافح الفيروسات
3. 🔧 تصحيح الثغرات المستغلة
4. 🔧 استعادة النظام من نسخة احتياطية نظيفة

**المتابعة:**
1. 📝 توثيق الحادث
2. 📝 تحديث السياسات الأمنية
3. 📝 تدريب المستخدمين
4. 📝 مراقبة معززة لفترة

### 💡 الدروس المستفادة
- أهمية الفحص العميق لحركة المرور
- ضرورة التسجيل الشامل للأحداث
- قيمة أدوات التحليل البصري
- أهمية الاستجابة السريعة

---

## 6. ARP - بروتوكول تحليل العناوين

### 🎯 الهدف
فهم آلية عمل بروتوكول ARP ومراقبة الجدول الخاص به.

### 📋 نظرة عامة
ARP (Address Resolution Protocol) يربط بين:
- **عنوان IP** (Layer 3)
- **عنوان MAC** (Layer 2)

### 🔍 التطبيق العملي

#### الخطوة 1: عرض جدول ARP

**على Windows:**
```cmd
arp -a
```

**على Linux/macOS:**
```bash
arp -a
```

#### الخطوة 2: قراءة النتائج

```
Interface: 192.168.1.100 --- 0x4
  Internet Address      Physical Address      Type
  192.168.1.1          00-1A-2B-3C-4D-5E     dynamic
  192.168.1.10         00-AA-BB-CC-DD-EE     dynamic
  192.168.1.255        FF-FF-FF-FF-FF-FF     static
  224.0.0.22           01-00-5E-00-00-16     static
```

### 📊 تحليل الجدول

| المكون | الشرح |
|--------|--------|
| **Internet Address** | عنوان IP للجهاز |
| **Physical Address** | عنوان MAC للجهاز |
| **Type** | نوع الإدخال |

### 🏷️ أنواع الإدخالات

**Dynamic:**
- يتم تعلمه تلقائياً من حركة المرور
- يُحذف بعد فترة عدم النشاط (عادة 2-10 دقائق)
- الأكثر شيوعاً في الشبكات

**Static:**
- يتم إضافته يدوياً بواسطة المدير
- لا يُحذف تلقائياً
- يُستخدم للأجهزة الحساسة

### 🔄 عناوين خاصة

**Broadcast:**
```
192.168.1.255 → FF-FF-FF-FF-FF-FF
```
- يُرسل لجميع الأجهزة في الشبكة

**Multicast:**
```
224.0.0.22 → 01-00-5E-00-00-16
```
- يُرسل لمجموعة محددة من الأجهزة

### 🧹 مسح الكاش

**مسح جميع الإدخالات (Windows):**
```cmd
arp -d *
```

**مسح إدخال محدد:**
```cmd
arp -d 192.168.1.10
```

**على Linux:**
```bash
sudo ip -s -s neigh flush all
```

### 📝 إضافة إدخال ثابت

**Windows:**
```cmd
arp -s 192.168.1.50 00-11-22-33-44-55
```

**Linux:**
```bash
sudo arp -s 192.168.1.50 00:11:22:33:44:55
```

### 🔍 حالات الاستخدام

**1. تشخيص مشاكل الاتصال:**
```bash
# فحص وجود الجهاز في الجدول
arp -a | grep 192.168.1.10

# إذا لم يظهر، المشكلة قد تكون في Layer 2
```

**2. كشف تضارب IP:**
```bash
# إذا ظهر نفس IP بعنواني MAC مختلفين
192.168.1.50    00-AA-BB-CC-DD-EE
192.168.1.50    00-11-22-33-44-55  # ⚠️ تضارب!
```

**3. مراقبة الأجهزة النشطة:**
```bash
# عد الأجهزة المتصلة
arp -a | grep dynamic | wc -l
```

### ⚠️ ملاحظات أمنية

**ARP Spoofing/Poisoning:**
- مهاجم يرسل رسائل ARP مزيفة
- يخدع الأجهزة لإرسال البيانات له
- **الحماية:** ARP Inspection على السويتشات

**الوقاية:**
```cisco
# تفعيل Dynamic ARP Inspection
Switch(config)# ip arp inspection vlan 10
Switch(config)# interface gi0/1
Switch(config-if)# ip arp inspection trust
```

### 💡 نصائح عملية
- راقب جدول ARP بشكل دوري
- انتبه للتغييرات المفاجئة في عناوين MAC
- استخدم إدخالات ثابتة للأجهزة الحساسة
- فعّل ARP Inspection في البيئات الحساسة

---

## 7. Nmap - مسح الشبكات

### 🎯 الهدف
استخدام Nmap لاكتشاف الأجهزة والخدمات النشطة على الشبكة.

### 📋 المتطلبات
- نظام Linux (يفضل Kali Linux)
- صلاحيات root للمسح المتقدم
- **بيئة اختبارية فقط** ⚠️

### 🔍 أنواع المسح

#### 1. اكتشاف الأجهزة النشطة (Host Discovery)

```bash
nmap -sn 192.168.1.0/24
```

**الشرح:**
- `-sn`: Ping Scan (بدون فحص المنافذ)
- يستخدم ICMP Echo Request
- يُظهر الأجهزة المتصلة والنشطة

**مثال على النتائج:**
```
Starting Nmap 7.94
Nmap scan report for 192.168.1.1
Host is up (0.0012s latency).

Nmap scan report for 192.168.1.50
Host is up (0.0034s latency).

Nmap scan report for 192.168.1.100
Host is up (0.0028s latency).

Nmap done: 256 IP addresses (3 hosts up) scanned in 2.85 seconds
```

#### 2. فحص المنافذ (Port Scanning)

**Basic TCP Scan:**
```bash
nmap 192.168.1.50
```

**Comprehensive Scan:**
```bash
nmap -sS -sV -O 192.168.1.50
```

**الشرح:**
- `-sS`: SYN Stealth Scan (أقل قابلية للكشف)
- `-sV`: كشف إصدارات الخدمات
- `-O`: تخمين نظام التشغيل

**مثال على النتائج:**
```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 8.2p1
80/tcp   open  http       Apache httpd 2.4.41
443/tcp  open  ssl/http   Apache httpd 2.4.41
3306/tcp open  mysql      MySQL 5.7.33

OS details: Linux 3.2 - 4.9
```

#### 3. فحص جميع المنافذ

```bash
nmap -p- 192.168.1.50
```

- `-p-`: فحص جميع المنافذ (1-65535)
- يستغرق وقتاً أطول

#### 4. فحص منافذ محددة

```bash
nmap -p 21,22,80,443,3306 192.168.1.50
```

### 🎭 أنواع مسح المنافذ

#### TCP SYN Scan (الأكثر شيوعاً)
```bash
nmap -sS 192.168.1.50
```
- يرسل SYN فقط
- لا يُكمل المصافحة الثلاثية
- أقل قابلية للكشف

#### TCP Connect Scan
```bash
nmap -sT 192.168.1.50
```
- يُكمل المصافحة الثلاثية
- لا يحتاج صلاحيات root
- أكثر قابلية للكشف

#### UDP Scan
```bash
nmap -sU 192.168.1.50
```
- فحص منافذ UDP
- بطيء جداً
- مهم لخدمات مثل DNS, DHCP

### 🔬 فحص الثغرات

```bash
nmap --script vuln 192.168.1.50
```

يستخدم NSE (Nmap Scripting Engine) للبحث عن ثغرات معروفة.

### 📊 تقييم المخاطر

#### منافذ عالية الخطورة:

| المنفذ | الخدمة | الخطر |
|--------|---------|-------|
| 21 | FTP | نقل بيانات بدون تشفير |
| 23 | Telnet | إدارة عن بعد بدون تشفير |
| 445 | SMB | عرضة لهجمات WannaCry وغيرها |
| 3389 | RDP | هدف شائع للهجمات |
| 1433 | MSSQL | قاعدة بيانات قد تكون عرضة |

### 🛡️ الحماية من Nmap

**1. على مستوى الجدار الناري:**
```cisco
# رفض المسح من مصادر خارجية
access-list 100 deny tcp any any range 1 1024
```

**2. على مستوى الخادم:**
```bash
# استخدام iptables لمنع المسح
iptables -A INPUT -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT
```

**3. أدوات كشف المسح:**
```bash
# تثبيت portsentry
sudo apt install portsentry

# مراقبة محاولات المسح
tail -f /var/log/syslog | grep portsentry
```

### 💡 نصائح للاستخدام الآمن
- ⚠️ **لا تقم بمسح شبكات لا تملكها**
- 📝 احصل على إذن خطي قبل أي اختبار اختراق
- 🔒 استخدم بيئة مختبرية معزولة للتدريب
- 📊 وثّق جميع عمليات المسح ونتائجها

### 🎓 أمثلة تعليمية إضافية

**مسح سريع لأهم 100 منفذ:**
```bash
nmap --top-ports 100 192.168.1.50
```

**مسح مع تجنب الكشف:**
```bash
nmap -sS -T2 -f 192.168.1.50
```
- `-T2`: سرعة بطيئة (أقل إزعاجاً)
- `-f`: تجزئة الحزم

**حفظ النتائج:**
```bash
nmap -oN results.txt 192.168.1.50
nmap -oX results.xml 192.168.1.50
```

---

## 8. ICMP - اختبارات الشبكة

### 🎯 الهدف
استخدام أدوات ICMP لتشخيص واختبار الاتصال بالشبكة.

### 📋 بروتوكول ICMP
Internet Control Message Protocol - بروتوكول رسائل التحكم في الإنترنت
- Layer 3 Protocol
- يُستخدم للتشخيص وإبلاغ الأخطاء
- لا يحمل بيانات المستخدم

### 🔍 الأوامر الأساسية

#### 1. Ping - اختبار الاتصال

**الاستخدام البسيط:**
```bash
ping 8.8.8.8
```

**مع خيارات متقدمة:**
```bash
# إرسال 10 حزم فقط
ping -c 10 8.8.8.8

# تغيير حجم الحزمة
ping -s 1000 8.8.8.8

# تحديد الوقت بين الحزم (ثانية واحدة)
ping -i 1 8.8.8.8
```

**مثال على النتائج:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=11.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=12.1 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 11.8/12.1/12.4/0.3 ms
```

#### 2. Traceroute - تتبع المسار

**على Linux:**
```bash
traceroute 8.8.8.8
```

**على Windows:**
```cmd
tracert 8.8.8.8
```

**مثال على النتائج:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.245 ms  1.123 ms  1.089 ms
 2  10.0.0.1 (10.0.0.1)  8.456 ms  8.234 ms  8.123 ms
 3  172.16.0.1 (172.16.0.1)  15.678 ms  15.456 ms  15.234 ms
 4  * * *
 5  8.8.8.8 (8.8.8.8)  22.345 ms  22.123 ms  21.987 ms
```

**التحليل:**
- كل سطر يمثل موجّه (router) في المسار
- الأرقام الثلاثة: زمن الاستجابة للمحاولات الثلاث
- `* * *`: الموجّه لا يستجيب لطلبات ICMP

#### 3. MTR - تتبع متقدم

```bash
mtr 8.8.8.8
```

يجمع بين ping و traceroute مع إحصائيات مستمرة:
```
                             Packets               Pings
 Host                      Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.1.1             0.0%    10    1.2   1.3   1.1   1.5   0.1
 2. 10.0.0.1                0.0%    10    8.4   8.5   8.1   9.2   0.3
 3. 8.8.8.8                 0.0%    10   22.1  22.3  21.8  23.1   0.4
```

### 📊 أنواع رسائل ICMP

| النوع | الاسم | الاستخدام |
|-------|-------|-----------|
| Type 0 | Echo Reply | رد على Ping |
| Type 3 | Destination Unreachable | الوجهة غير متاحة |
| Type 5 | Redirect | إعادة توجيه المسار |
| Type 8 | Echo Request | طلب Ping |
| Type 11 | Time Exceeded | انتهاء TTL (يستخدمه traceroute) |

### ⚠️ اختبارات متقدمة (بيئة مختبرية فقط)

#### ICMP Flood Test

```bash
# ⚠️ للاختبار في بيئة معزولة فقط
sudo ping -f -s 65500 TARGET_IP
```

**الشرح:**
- `-f`: Flood mode (إرسال سريع جداً)
- `-s 65500`: حجم الحزمة (أقصى حد)
- **يُستخدم لاختبار تحمل الشبكة**

**النتائج المتوقعة:**
```
PING target (192.168.1.50) 65500(65528) bytes of data.
..................................
--- target ping statistics ---
10000 packets transmitted, 9856 received, 1.44% packet loss
```

#### Ping of Death (تاريخي)

```bash
# ⚠️ لا يعمل على الأنظمة الحديثة
ping -s 65510 TARGET_IP
```

**الخلفية:**
- كان يُسبب تعطل الأنظمة القديمة
- الأنظمة الحديثة محمية ضده

### 🛡️ الحماية من هجمات ICMP

#### 1. تحديد معدل ICMP

**على Linux:**
```bash
# تحديد عدد حزم ICMP في الثانية
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

#### 2. على Cisco Router

```cisco
# تحديد معدل ICMP
access-list 100 permit icmp any any echo-reply
access-list 100 deny icmp any any echo-request
interface gi0/0
ip access-group 100 in
rate-limit input access-group 100 1000000 250000 500000 conform-action transmit exceed-action drop
```

#### 3. حظر ICMP نهائياً (غير مستحسن)

```bash
# يُفقد القدرة على التشخيص
iptables -A INPUT -p icmp -j DROP
```

### 📝 حالات الاستخدام العملية

**1. تشخيص انقطاع الاتصال:**
```bash
# خطوة بخطوة
ping 192.168.1.1  # البوابة المحلية
ping 8.8.8.8      # خادم خارجي
traceroute 8.8.8.8  # تحديد نقطة الانقطاع
```

**2. قياس جودة الاتصال:**
```bash
# مراقبة لمدة 5 دقائق
ping -c 300 -i 1 8.8.8.8 | tee ping_results.txt

# تحليل النتائج
grep "time=" ping_results.txt | awk '{print $7}' | sed 's/time=//' | sort -n
```

**3. اختبار MTU (Maximum Transmission Unit):**
```bash
# ابدأ بحجم كبير وقلله تدريجياً
ping -M do -s 1472 8.8.8.8

# إذا فشل، قلل الحجم حتى ينجح
ping -M do -s 1400 8.8.8.8
```

### 💡 نصائح مهمة
- ICMP ليس آمناً 100% - يمكن استخدامه في الهجمات
- بعض الشبكات تحظر ICMP للأمان
- استخدم أدوات بديلة (TCP ping) عند الحاجة
- لا تعتمد على ICMP فقط للمراقبة

---

## 9. SYN Flooding Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم SYN Flood وتأثيره على الخوادم.

### ⚠️ تحذير مهم جداً
```
┌─────────────────────────────────────────────────┐
│  ⚠️  هذا المحتوى لأغراض تعليمية فقط  ⚠️       │
│                                                  │
│  تنفيذ هذا الهجوم على أنظمة حقيقية:           │
│  • جريمة يعاقب عليها القانون                   │
│  • يتطلب إذن خطي مسبق                          │
│  • يُسمح فقط في بيئة مختبرية معزولة           │
└─────────────────────────────────────────────────┘
```

### 📋 نظرة عامة على الهجوم

#### المصافحة الثلاثية الطبيعية (TCP Three-Way Handshake)

```
Client                    Server
  |                         |
  |-------- SYN ---------->|  [1]
  |<----- SYN-ACK ---------|  [2]
  |-------- ACK ---------->|  [3]
  |                         |
  |==== Connection Open ====|
```

#### هجوم SYN Flood

```
Attacker                  Server
  |                         |
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |      (thousands)        |
  |                         |
  |   ❌ No ACK Sent ❌    |
  |                         |
  |                    [Server]
  |                   SYN Queue
  |                   ┌─────────┐
  |                   │ SYN #1  │ ⏳
  |                   │ SYN #2  │ ⏳
  |                   │ SYN #3  │ ⏳
  |                   │ SYN #... │ ⏳
  |                   │ FULL!   │ 🔴
  |                   └─────────┘
```

### 🔬 التطبيق العملي (بيئة مختبرية فقط)

#### البيئة المطلوبة
- آلة افتراضية للمهاجم (Kali Linux)
- خادم اختبار معزول
- شبكة محلية منعزلة تماماً

#### الأداة: hping3

**التثبيت:**
```bash
sudo apt update
sudo apt install hping3
```

**الهجوم الأساسي:**
```bash
sudo hping3 -S -p 80 --flood TARGET_IP
```

**الشرح:**
- `-S`: إرسال حزم SYN فقط
- `-p 80`: استهداف المنفذ 80 (HTTP)
- `--flood`: إرسال أسرع ما يمكن (بدون انتظار)
- `TARGET_IP`: عنوان الخادم المستهدف

**هجوم متقدم مع عناوين مزيفة:**
```bash
sudo hping3 -S -p 80 --flood --rand-source TARGET_IP
```

- `--rand-source`: عناوين IP مصدر عشوائية
- يُصعّب تتبع المهاجم
- يتجاوز بعض آليات الحماية البسيطة

### 📊 المراقبة على الخادم المستهدف

#### عرض الاتصالات نصف المفتوحة

```bash
netstat -ant | grep SYN_RECV
```

**مثال على النتائج أثناء الهجوم:**
```
tcp    0    0 192.168.1.50:80    10.0.0.100:45123    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.101:45124    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.102:45125    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.103:45126    SYN_RECV
... (مئات أو آلاف السطور)
```

#### مراقبة استهلاك الموارد

```bash
# عرض قائمة الانتظار
ss -tan | grep SYN-RECV | wc -l

# مراقبة الذاكرة
free -h

# مراقبة CPU
top
```

**مؤشرات الهجوم:**
```
SYN-RECV connections: 5000+  🔴 (طبيعي: < 100)
Memory usage: 95%             🔴 (طبيعي: < 70%)
CPU load: 100%                🔴 (طبيعي: < 50%)
```

### 🛡️ آليات الحماية

#### 1. SYN Cookies

**تفعيل SYN Cookies على Linux:**
```bash
# التحقق من الحالة الحالية
cat /proc/sys/net/ipv4/tcp_syncookies

# التفعيل (1 = enabled)
sudo sysctl -w net.ipv4.tcp_syncookies=1

# جعله دائماً
echo "net.ipv4.tcp_syncookies = 1" | sudo tee -a /etc/sysctl.conf
```

**كيف تعمل:**
```
بدلاً من تخزين الاتصال في الذاكرة:
1. الخادم يحسب قيمة مشفرة (cookie)
2. يُرسلها في SYN-ACK
3. لا يحفظ شيء في الذاكرة
4. عند استلام ACK، يتحقق من الـ cookie
5. إن صح، يُكمل الاتصال

النتيجة: لا استنزاف للذاكرة ✅
```

#### 2. تحديد معدل الاتصالات الجديدة

**باستخدام iptables:**
```bash
# السماح بـ 5 اتصالات جديدة فقط في الثانية من نفس IP
iptables -A INPUT -p tcp --syn -m limit --limit 5/s --limit-burst 10 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP
```

**باستخدام nftables:**
```bash
nft add rule ip filter input tcp flags syn ct state new limit rate 5/second accept
```

#### 3. زيادة حجم قائمة SYN

```bash
# زيادة الحد الأقصى للاتصالات نصف المفتوحة
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# تقليل وقت الانتظار
sudo sysctl -w net.ipv4.tcp_synack_retries=2
```

#### 4. استخدام Firewall متقدم

**Cisco ASA:**
```cisco
! تحديد عدد الاتصالات المتزامنة
access-list OUTSIDE_IN extended permit tcp any host 203.0.113.50 eq www
class-map SYN-ATTACK
 match access-list OUTSIDE_IN
policy-map OUTSIDE_POLICY
 class SYN-ATTACK
  set connection conn-max 100 embryonic-conn-max 50
service-policy OUTSIDE_POLICY interface outside
```

**pfSense/OPNsense:**
- تفعيل "SYN Proxy"
- ضبط "Maximum States" و "Maximum Source Nodes"

#### 5. استخدام CDN/DDoS Protection

خدمات مثل:
- Cloudflare
- AWS Shield
- Akamai

تقوم بـ:
- امتصاص الهجمات الضخمة
- توزيع الحمل
- فلترة الحركة الخبيثة

### 📈 التأثير على الخادم

#### مراحل الهجوم

**المرحلة 1: البداية (0-30 ثانية)**
```
SYN Queue: 20% full
Response Time: 100ms → 500ms
```

**المرحلة 2: التصعيد (30-60 ثانية)**
```
SYN Queue: 80% full
Response Time: 500ms → 3000ms
New connections: بطيئة جداً
```

**المرحلة 3: الانهيار (60+ ثانية)**
```
SYN Queue: 100% full
Response Time: timeout
New connections: مرفوضة بالكامل
Service: غير متاح 🔴
```

### 🔍 كشف الهجوم

#### علامات التحذير

```bash
# ارتفاع مفاجئ في SYN_RECV
watch -n 1 'netstat -ant | grep SYN_RECV | wc -l'

# تنبيه عند تجاوز حد معين
while true; do
  COUNT=$(netstat -ant | grep SYN_RECV | wc -l)
  if [ $COUNT -gt 1000 ]; then
    echo "⚠️ Possible SYN Flood: $COUNT connections"
  fi
  sleep 5
done
```

#### تحليل السجلات

```bash
# عرض أكثر IP مصدرية نشاطاً
netstat -ant | grep SYN_RECV | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -10
```

### 💡 الدروس المستفادة

**للمدافعين:**
1. ✅ فعّل SYN Cookies دائماً
2. ✅ راقب معدلات الاتصال بشكل مستمر
3. ✅ استخدم Firewall مع إمكانيات DDoS protection
4. ✅ احتفظ بخطة استجابة جاهزة

**للمهاجمين الأخلاقيين:**
1. 📝 احصل على إذن خطي قبل الاختبار
2. 🔒 استخدم بيئة معزولة تماماً
3. ⏰ حدد نافذة زمنية محدودة للاختبار
4. 📊 وثّق النتائج بشكل احترافي

---

## 10. UDP Flood Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم UDP Flood والفرق بينه وبين SYN Flood.

### ⚠️ تحذير قانوني
```
┌──────────────────────────────────────────────────┐
│  🚨 هذا المحتوى للتعليم والأبحاث الأمنية فقط  │
│                                                   │
│  القيام بهذا الهجوم دون إذن هو:                │
│  • جريمة إلكترونية                              │
│  • يُعاقب عليها بالسجن والغرامة                │
│  • يُسمح فقط في بيئة مختبرية مملوكة لك         │
└──────────────────────────────────────────────────┘
```

### 📋 الفرق بين TCP و UDP

| الميزة | TCP | UDP |
|--------|-----|-----|
| الاتصال | Connection-oriented | Connectionless |
| المصافحة | نعم (3-way handshake) | لا |
| الموثوقية | مضمون الوصول | غير مضمون |
| السرعة | أبطأ | أسرع |
| الترتيب | مرتب | غير مرتب |

### 🔬 آلية هجوم UDP Flood

```
Attacker                    Target Server
  |                              |
  |------- UDP Packet ---------->| Port 53 (DNS)
  |------- UDP Packet ---------->| Port 161 (SNMP)
  |------- UDP Packet ---------->| Port 12345 (Random)
  |------- UDP Packet ---------->| Port 9999 (Closed)
  |       (thousands/sec)        |
  |                              |
  |                         [Server Process]
  |                         1. تلقي الحزمة
  |                         2. فحص المنفذ
  |                         3. المنفذ مغلق؟
  |                         4. إرسال ICMP Port Unreachable
  |                              |
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |                              |
  |                    [Result: Resource Exhaustion]
  |                    • CPU: 100%
  |                    • Bandwidth: Saturated
  |                    • Service: Unavailable
```

### 🔬 التطبيق العملي (بيئة مختبرية معزولة فقط)

#### البيئة المطلوبة
- شبكة مختبرية معزولة تماماً
- خادم اختبار مخصص
- إذن خطي موثق

#### استخدام hping3

**الهجوم الأساسي:**
```bash
sudo hping3 --udp -p 53 --flood TARGET_IP
```

**الشرح:**
- `--udp`: استخدام بروتوكول UDP
- `-p 53`: استهداف منفذ DNS
- `--flood`: إرسال بأقصى سرعة ممكنة

**هجوم مع منافذ عشوائية:**
```bash
sudo hping3 --udp --rand-dest --flood TARGET_IP
```

- `--rand-dest`: منافذ وجهة عشوائية
- يُضعف الخادم أكثر (يفحص كل منفذ)

**هجوم مع حجم حزمة كبير:**
```bash
sudo hping3 --udp -p 53 --flood -d 65500 TARGET_IP
```

- `-d 65500`: حجم البيانات (أقصى حد لـ UDP)
- يستهلك bandwidth أكثر

### 📊 المراقبة على الخادم المستهدف

#### عرض إحصائيات UDP

```bash
netstat -s | grep UDP
```

**مثال على النتائج أثناء الهجوم:**
```
UDP:
    125847 packets received
    98234 packets to unknown port received
    89123 packet receive errors
    45678 packets sent
    RcvbufErrors: 12345
    SndbufErrors: 0
    InErrors: 89123
```

#### مراقبة الحزم في الوقت الفعلي

```bash
# عرض معدل حزم UDP
watch -n 1 'netstat -su | grep "packets received"'

# مراقبة منافذ محددة
tcpdump -i eth0 udp port 53 -nn
```

**مؤشرات الهجوم:**
```
UDP packets/sec: 50,000+     🔴 (طبيعي: < 1,000)
Port Unreachable: 40,000+    🔴 (طبيعي: < 10)
Bandwidth usage: 90%+        🔴 (طبيعي: < 50%)
CPU usage: 100%              🔴 (طبيعي: < 30%)
```

#### تحديد أكثر المنافذ المستهدفة

```bash
tcpdump -i eth0 -nn udp -c 1000 | awk '{print $5}' | cut -d'.' -f5 | sort | uniq -c | sort -nr | head -10
```

### 🛡️ آليات الحماية

#### 1. Rate Limiting على Linux

```bash
# تحديد معدل حزم UDP من كل IP
iptables -A INPUT -p udp -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p udp -j DROP

# حماية منافذ محددة
iptables -A INPUT -p udp --dport 53 -m limit --limit 50/s -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j DROP
```

#### 2. تعطيل ICMP Port Unreachable

```bash
# منع إرسال رسائل Port Unreachable
iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP

# أو عبر sysctl
sysctl -w net.ipv4.icmp_ignore_bogus_error_responses=1
```

**الفائدة:**
- يوفر موارد CPU
- يمنع استنزاف bandwidth بالردود

#### 3. حماية على مستوى الخادم

```bash
# زيادة حجم buffer للـ UDP
sysctl -w net.core.rmem_max=26214400
sysctl -w net.core.rmem_default=26214400

# تحسين معالجة الحزم
sysctl -w net.core.netdev_max_backlog=5000
```

#### 4. Firewall Rules على Cisco

```cisco
! تحديد معدل UDP
access-list 110 permit udp any host 203.0.113.50 eq domain
access-list 110 deny udp any any

interface GigabitEthernet0/0
 ip access-group 110 in
 rate-limit input access-group 110 10000000 1000000 2000000 conform-action transmit exceed-action drop
```

#### 5. استخدام fail2ban

```bash
# تثبيت fail2ban
sudo apt install fail2ban

# إنشاء قاعدة لـ UDP flood
sudo nano /etc/fail2ban/filter.d/udp-flood.conf
```

**محتوى الملف:**
```ini
[Definition]
failregex = .*kernel.*UDP.*SRC=<HOST>.*
ignoreregex =
```

**التفعيل:**
```ini
# /etc/fail2ban/jail.local
[udp-flood]
enabled = true
filter = udp-flood
logpath = /var/log/syslog
maxretry = 100
findtime = 60
bantime = 3600
```

### 📈 المنافذ الشائعة المستهدفة

| المنفذ | الخدمة | سبب الاستهداف |
|--------|---------|---------------|
| 53 | DNS | حيوي للشبكة، يولد استجابات |
| 123 | NTP | يمكن تضخيمه (Amplification) |
| 161 | SNMP | إدارة، غالباً مفتوح |
| 1900 | SSDP | UPnP، شائع في الأجهزة المنزلية |
: MD5
- `$5# تدريبات عملية في أمن الشبكات
## Network Security - Practical Labs

---

## 📑 جدول المحتويات

1. [NAT/PAT - ترجمة عناوين الشبكة](#1-natpat---ترجمة-عناوين-الشبكة)
2. [ACL - قوائم التحكم بالوصول](#2-acl---قوائم-التحكم-بالوصول)
3. [FTP وآلية Established](#3-ftp-وآلية-established)
4. [WAF و Load Balancer](#4-waf-و-load-balancer)
5. [Cisco Firepower - تحليل البرمجيات الخبيثة](#5-cisco-firepower---تحليل-البرمجيات-الخبيثة)
6. [ARP - بروتوكول تحليل العناوين](#6-arp---بروتوكول-تحليل-العناوين)
7. [Nmap - مسح الشبكات](#7-nmap---مسح-الشبكات)
8. [ICMP - اختبارات الشبكة](#8-icmp---اختبارات-الشبكة)
9. [SYN Flooding Attack](#9-syn-flooding-attack)
10. [UDP Flood Attack](#10-udp-flood-attack)
11. [Nmap Port Scanning](#11-nmap-port-scanning)
12. [John the Ripper - كسر كلمات المرور](#12-john-the-ripper---كسر-كلمات-المرور)
13. [DHCP Snooping](#13-dhcp-snooping)

---

## 1. NAT/PAT - ترجمة عناوين الشبكة

### 🎯 الهدف
فهم آلية عمل NAT/PAT في ترجمة العناوين الخاصة إلى عناوين عامة.

### 📋 السيناريو
جهاز داخلي بعنوان `10.0.0.5` يريد تصفح الإنترنت عبر جدار ناري يستخدم NAT.

### 🔄 خطوات العمل

1. **الطلب الصادر:**
   - الجهاز الداخلي `10.0.0.5` يرسل طلب HTTP
   - الجدار الناري يترجم العنوان إلى `203.0.113.2:10500`
   - العنوان `203.0.113.2` هو العنوان العام
   - المنفذ `10500` يُستخدم لتمييز الاتصال

2. **الرد الوارد:**
   - عند وصول الرد من الإنترنت إلى `203.0.113.2:10500`
   - يعيد NAT الترجمة العكسية إلى `10.0.0.5` بالمنفذ الأصلي
   - يتم تسليم البيانات للجهاز الداخلي

### ✅ الفائدة
- مئات الأجهزة يمكنها مشاركة عنوان عام واحد
- لا يوجد تضارب بفضل استخدام أرقام المنافذ (Port Numbers)
- توفير في عناوين IPv4 العامة

### 📝 ملاحظات
- PAT (Port Address Translation) هو نوع من NAT
- يُسمى أيضاً NAT Overload
- الجدول الداخلي يحتفظ بالربط بين العناوين الداخلية والمنافذ

---

## 2. ACL - قوائم التحكم بالوصول

### 🎯 الهدف
تطبيق سياسات أمنية باستخدام قوائم التحكم بالوصول (Access Control Lists).

### 📋 السيناريو
لديك:
- **Web Server** في DMZ (Demilitarized Zone)
- **Database Server** داخل الشبكة الداخلية

> 💡 **DMZ:** منطقة منزوعة السلاح، وهي منطقة بين الشبكة الداخلية والإنترنت لتقليل المخاطر الأمنية.

### 🔧 التكوين العملي

```cisco
access-list 100 permit tcp any 10.10.10.10 eq www
access-list 100 permit tcp any 10.10.10.10 eq 443
access-list 100 permit tcp any 10.10.10.12 eq ftp
access-list 100 permit tcp any 10.10.10.12 eq ftp-data
access-list 100 deny ip any any log

interface gi0/1
ip access-group 100 in
```

### 📊 شرح الأوامر

| الأمر | الوظيفة |
|------|---------|
| `permit tcp any 10.10.10.10 eq www` | السماح بالوصول لخادم الويب عبر HTTP (منفذ 80) |
| `permit tcp any 10.10.10.10 eq 443` | السماح بالوصول عبر HTTPS (منفذ 443) |
| `permit tcp any 10.10.10.12 eq ftp` | السماح بـ FTP Control (منفذ 21) |
| `permit tcp any 10.10.10.12 eq ftp-data` | السماح بـ FTP Data (منفذ 20) |
| `deny ip any any log` | حظر أي حركة أخرى مع التسجيل |
| `ip access-group 100 in` | تطبيق القائمة على الواجهة |

### 🎯 الهدف الأمني
- السماح بالوصول فقط للمنافذ المطلوبة
- منع الوصول المباشر لخادم قاعدة البيانات
- تسجيل المحاولات المشبوهة
- تقليل نطاق الهجوم (Attack Surface)

### ⚠️ ملاحظات مهمة
- الوصول لقاعدة البيانات يجب أن يتم فقط من خلال خادم الويب
- ترتيب القواعد مهم جداً (من الأعلى للأسفل)
- القاعدة الضمنية في النهاية: `deny any any`

---

## 3. FTP وآلية Established

### 🎯 الهدف
فهم الفرق بين FTP Passive و Active Mode وتأثيره على ACL.

### 📋 بنية FTP
بروتوكول FTP يستخدم قناتين منفصلتين:
- **Port 21:** قناة التحكم (Control Channel)
- **Port 20:** قناة البيانات (Data Channel)

### 🔄 FTP Passive Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Random High Port (Data)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يطلب وضع Passive
3. الخادم يرد بمنفذ عشوائي عالي
4. العميل يفتح اتصال البيانات نحو هذا المنفذ
5. كل الردود تحتوي على بت ACK ← **مسموح بها** ✅

### 🔄 FTP Active Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Port 20 (Data initiated by server)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يخبر الخادم بمنفذ معين لاستقبال البيانات
3. **الخادم يبدأ اتصال البيانات** من المنفذ 20
4. أول حزمة من الخادم تحتوي على SYN فقط ← **تُرفض** 🚫

### ⚠️ المشكلة مع Established

```cisco
access-list 110 permit tcp any any established
```

هذه القاعدة تسمح فقط بالحزم التي تحتوي على:
- بت ACK
- أو بت RST

**النتيجة:**
- في Active Mode، الخادم يرسل SYN أولاً
- هذه الحزمة لا تحقق شرط "established"
- الاتصال يفشل 🚫

### ✅ الحل الأمثل
استخدام جدار ناري حقيقي (Stateful Firewall) قادر على:
- تتبع الجلسات النشطة
- فهم البروتوكولات المعقدة مثل FTP
- السماح بالاتصالات الديناميكية تلقائياً

### 📝 الخلاصة
- `established` لا يكفي للبروتوكولات المعقدة
- FTP Passive Mode أكثر توافقاً مع الجدران النارية
- Stateful Firewalls ضرورية للبيئات الإنتاجية

---

## 4. WAF و Load Balancer

### 🎯 الهدف
بناء بنية تحتية آمنة وقابلة للتوسع لموقع ويب.

### 📋 السيناريو
موقع ويب يحتوي على عدة خوادم Web Servers خلف Load Balancer، مع حماية من هجمات XSS.

### 🏗️ البنية المعمارية

```
[Internet]
    |
    ↓
[WAF - Web Application Firewall]
    |
    ↓
[Load Balancer]
    |
    ↓------------------↓------------------↓------------------↓
[Web Server 1]   [Web Server 2]   [Web Server 3]   [Web Server 4]
```

### ⚖️ مهام Load Balancer

**خوارزمية التوزيع:**
- **Round Robin:** توزيع متساوٍ على جميع الخوادم
- دورة منتظمة: Server 1 → Server 2 → Server 3 → Server 4 → تكرار

**إدارة الأعطال:**
1. مراقبة صحة الخوادم (Health Checks)
2. عند فشل أحد الخوادم:
   - إزالته من قائمة التوزيع تلقائياً
   - تحويل طلباته إلى الخوادم السليمة
3. عند عودة الخادم: إعادة إدراجه تلقائياً

### 🛡️ مهام WAF (كـ Reverse Proxy)

**الحماية من XSS:**
```javascript
// مثال على محاولة هجوم XSS
<script>alert('XSS Attack')</script>
```

**آلية الحماية:**
1. فحص كل طلب HTTP قبل وصوله للخوادم
2. كشف الشيفرات الخبيثة:
   - JavaScript مشبوه
   - SQL Injection
   - Command Injection
3. منع الطلبات المشبوهة فوراً ⛔
4. منع تسريب البيانات الحساسة في الاستجابات

### 🔍 سياسات الأمان

| نوع الحماية | الوظيفة |
|-------------|---------|
| XSS Prevention | منع حقن JavaScript |
| SQL Injection | حماية قواعد البيانات |
| CSRF Protection | منع الطلبات المزورة |
| Rate Limiting | الحد من معدل الطلبات |
| IP Blacklisting | حظر IP المشبوهة |

### ✅ المزايا المحققة

**الأداء:**
- ⚡ توزيع الأحمال بشكل متوازن
- 🔄 تحمل الأعطال (Fault Tolerance)
- 📈 قابلية التوسع الأفقي

**الأمان:**
- 🛡️ حماية متقدمة من WAF
- 🔒 فحص جميع الطلبات
- 📝 تسجيل المحاولات المشبوهة
- 🚫 حظر الهجمات في الوقت الحقيقي

### 📊 مثال على سجلات WAF

```
2025-10-26 10:15:32 | BLOCKED | XSS Attempt | IP: 192.168.1.100
2025-10-26 10:16:45 | BLOCKED | SQL Injection | IP: 10.0.0.50
2025-10-26 10:17:12 | ALLOWED | Normal Request | IP: 172.16.0.25
```

---

## 5. Cisco Firepower - تحليل البرمجيات الخبيثة

### 🎯 الهدف
فهم كيفية عمل أنظمة كشف البرمجيات الخبيثة المتقدمة وتتبع انتشارها.

### 📋 السيناريو
ملف باسم `tool.exe` تم اكتشافه ضمن حركة المرور على الشبكة.

### 🔍 مراحل الكشف والتحليل

#### المرحلة 1: الفحص الأولي
1. **اعتراض الملف:** جهاز Cisco Firepower يفحص جميع الملفات المارة
2. **حساب البصمة:** استخراج بصمة SHA-256 للملف
   ```
   SHA-256: a3f7b8c9d2e1f4a6b5c8d7e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
   ```
3. **المقارنة:** مقارنة البصمة مع قاعدة بيانات التهديدات

#### المرحلة 2: التصنيف
```
Status: ⚠️ MALWARE DETECTED
Threat Name: Generic.Trojan.Downloader
Threat Score: 95/100 (High)
Category: Trojan
First Seen: 2025-10-25 14:30:22
```

#### المرحلة 3: تحديد الأجهزة المصابة
الأجهزة التالية تم تحديدها كمصابة:
- 🖥️ `192.168.10.90`
- 🖥️ `192.168.133.50`

### 📈 Trajectory Map (خريطة المسار)

```
Time →
Device 1: --------[*]==========[*]-------
                   ↓            ↓
Device 2: --------[*]----[*]----[*]------
                         ↓      ↓
Device 3: ---------------[*]=====[*]-----
```

**رموز الخريطة:**
- `[*]` نقطة انتقال الملف
- `=` فترة نشاط الملف على الجهاز
- `↓` اتجاه انتقال العدوى (من جهاز لآخر)
- `-` فترة خمول

### 📊 جدول الأحداث (Events Table)

| الوقت | الجهاز المصدر | الجهاز الهدف | الحدث | الحالة |
|------|---------------|--------------|--------|--------|
| 14:30:22 | External | 192.168.10.90 | File Download | Infected |
| 14:35:18 | 192.168.10.90 | 192.168.133.50 | File Transfer | Infected |
| 14:42:33 | 192.168.133.50 | Network Share | Propagation Attempt | Blocked |

### 🎯 استخدامات Trajectory Map

1. **تحديد المصدر الأولي:**
   - أول جهاز تلقى الملف
   - نقطة الدخول إلى الشبكة

2. **تتبع مسار الانتشار:**
   - الأجهزة الوسيطة
   - طرق الانتقال المستخدمة

3. **تقييم نطاق الإصابة:**
   - عدد الأجهزة المصابة
   - الأقسام المتأثرة

4. **التخطيط للاستجابة:**
   - أولوية المعالجة
   - خطة العزل والإزالة

### 🚨 خطة الاستجابة للحادث

**الخطوات الفورية:**
1. ✅ عزل الأجهزة المصابة عن الشبكة
2. ✅ حظر بصمة الملف على جميع الأجهزة
3. ✅ فحص شامل للأجهزة المجاورة
4. ✅ تحليل سجلات النشاط

**الإجراءات التصحيحية:**
1. 🔧 إزالة البرمجية الخبيثة
2. 🔧 تحديث تعريفات مكافح الفيروسات
3. 🔧 تصحيح الثغرات المستغلة
4. 🔧 استعادة النظام من نسخة احتياطية نظيفة

**المتابعة:**
1. 📝 توثيق الحادث
2. 📝 تحديث السياسات الأمنية
3. 📝 تدريب المستخدمين
4. 📝 مراقبة معززة لفترة

### 💡 الدروس المستفادة
- أهمية الفحص العميق لحركة المرور
- ضرورة التسجيل الشامل للأحداث
- قيمة أدوات التحليل البصري
- أهمية الاستجابة السريعة

---

## 6. ARP - بروتوكول تحليل العناوين

### 🎯 الهدف
فهم آلية عمل بروتوكول ARP ومراقبة الجدول الخاص به.

### 📋 نظرة عامة
ARP (Address Resolution Protocol) يربط بين:
- **عنوان IP** (Layer 3)
- **عنوان MAC** (Layer 2)

### 🔍 التطبيق العملي

#### الخطوة 1: عرض جدول ARP

**على Windows:**
```cmd
arp -a
```

**على Linux/macOS:**
```bash
arp -a
```

#### الخطوة 2: قراءة النتائج

```
Interface: 192.168.1.100 --- 0x4
  Internet Address      Physical Address      Type
  192.168.1.1          00-1A-2B-3C-4D-5E     dynamic
  192.168.1.10         00-AA-BB-CC-DD-EE     dynamic
  192.168.1.255        FF-FF-FF-FF-FF-FF     static
  224.0.0.22           01-00-5E-00-00-16     static
```

### 📊 تحليل الجدول

| المكون | الشرح |
|--------|--------|
| **Internet Address** | عنوان IP للجهاز |
| **Physical Address** | عنوان MAC للجهاز |
| **Type** | نوع الإدخال |

### 🏷️ أنواع الإدخالات

**Dynamic:**
- يتم تعلمه تلقائياً من حركة المرور
- يُحذف بعد فترة عدم النشاط (عادة 2-10 دقائق)
- الأكثر شيوعاً في الشبكات

**Static:**
- يتم إضافته يدوياً بواسطة المدير
- لا يُحذف تلقائياً
- يُستخدم للأجهزة الحساسة

### 🔄 عناوين خاصة

**Broadcast:**
```
192.168.1.255 → FF-FF-FF-FF-FF-FF
```
- يُرسل لجميع الأجهزة في الشبكة

**Multicast:**
```
224.0.0.22 → 01-00-5E-00-00-16
```
- يُرسل لمجموعة محددة من الأجهزة

### 🧹 مسح الكاش

**مسح جميع الإدخالات (Windows):**
```cmd
arp -d *
```

**مسح إدخال محدد:**
```cmd
arp -d 192.168.1.10
```

**على Linux:**
```bash
sudo ip -s -s neigh flush all
```

### 📝 إضافة إدخال ثابت

**Windows:**
```cmd
arp -s 192.168.1.50 00-11-22-33-44-55
```

**Linux:**
```bash
sudo arp -s 192.168.1.50 00:11:22:33:44:55
```

### 🔍 حالات الاستخدام

**1. تشخيص مشاكل الاتصال:**
```bash
# فحص وجود الجهاز في الجدول
arp -a | grep 192.168.1.10

# إذا لم يظهر، المشكلة قد تكون في Layer 2
```

**2. كشف تضارب IP:**
```bash
# إذا ظهر نفس IP بعنواني MAC مختلفين
192.168.1.50    00-AA-BB-CC-DD-EE
192.168.1.50    00-11-22-33-44-55  # ⚠️ تضارب!
```

**3. مراقبة الأجهزة النشطة:**
```bash
# عد الأجهزة المتصلة
arp -a | grep dynamic | wc -l
```

### ⚠️ ملاحظات أمنية

**ARP Spoofing/Poisoning:**
- مهاجم يرسل رسائل ARP مزيفة
- يخدع الأجهزة لإرسال البيانات له
- **الحماية:** ARP Inspection على السويتشات

**الوقاية:**
```cisco
# تفعيل Dynamic ARP Inspection
Switch(config)# ip arp inspection vlan 10
Switch(config)# interface gi0/1
Switch(config-if)# ip arp inspection trust
```

### 💡 نصائح عملية
- راقب جدول ARP بشكل دوري
- انتبه للتغييرات المفاجئة في عناوين MAC
- استخدم إدخالات ثابتة للأجهزة الحساسة
- فعّل ARP Inspection في البيئات الحساسة

---

## 7. Nmap - مسح الشبكات

### 🎯 الهدف
استخدام Nmap لاكتشاف الأجهزة والخدمات النشطة على الشبكة.

### 📋 المتطلبات
- نظام Linux (يفضل Kali Linux)
- صلاحيات root للمسح المتقدم
- **بيئة اختبارية فقط** ⚠️

### 🔍 أنواع المسح

#### 1. اكتشاف الأجهزة النشطة (Host Discovery)

```bash
nmap -sn 192.168.1.0/24
```

**الشرح:**
- `-sn`: Ping Scan (بدون فحص المنافذ)
- يستخدم ICMP Echo Request
- يُظهر الأجهزة المتصلة والنشطة

**مثال على النتائج:**
```
Starting Nmap 7.94
Nmap scan report for 192.168.1.1
Host is up (0.0012s latency).

Nmap scan report for 192.168.1.50
Host is up (0.0034s latency).

Nmap scan report for 192.168.1.100
Host is up (0.0028s latency).

Nmap done: 256 IP addresses (3 hosts up) scanned in 2.85 seconds
```

#### 2. فحص المنافذ (Port Scanning)

**Basic TCP Scan:**
```bash
nmap 192.168.1.50
```

**Comprehensive Scan:**
```bash
nmap -sS -sV -O 192.168.1.50
```

**الشرح:**
- `-sS`: SYN Stealth Scan (أقل قابلية للكشف)
- `-sV`: كشف إصدارات الخدمات
- `-O`: تخمين نظام التشغيل

**مثال على النتائج:**
```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 8.2p1
80/tcp   open  http       Apache httpd 2.4.41
443/tcp  open  ssl/http   Apache httpd 2.4.41
3306/tcp open  mysql      MySQL 5.7.33

OS details: Linux 3.2 - 4.9
```

#### 3. فحص جميع المنافذ

```bash
nmap -p- 192.168.1.50
```

- `-p-`: فحص جميع المنافذ (1-65535)
- يستغرق وقتاً أطول

#### 4. فحص منافذ محددة

```bash
nmap -p 21,22,80,443,3306 192.168.1.50
```

### 🎭 أنواع مسح المنافذ

#### TCP SYN Scan (الأكثر شيوعاً)
```bash
nmap -sS 192.168.1.50
```
- يرسل SYN فقط
- لا يُكمل المصافحة الثلاثية
- أقل قابلية للكشف

#### TCP Connect Scan
```bash
nmap -sT 192.168.1.50
```
- يُكمل المصافحة الثلاثية
- لا يحتاج صلاحيات root
- أكثر قابلية للكشف

#### UDP Scan
```bash
nmap -sU 192.168.1.50
```
- فحص منافذ UDP
- بطيء جداً
- مهم لخدمات مثل DNS, DHCP

### 🔬 فحص الثغرات

```bash
nmap --script vuln 192.168.1.50
```

يستخدم NSE (Nmap Scripting Engine) للبحث عن ثغرات معروفة.

### 📊 تقييم المخاطر

#### منافذ عالية الخطورة:

| المنفذ | الخدمة | الخطر |
|--------|---------|-------|
| 21 | FTP | نقل بيانات بدون تشفير |
| 23 | Telnet | إدارة عن بعد بدون تشفير |
| 445 | SMB | عرضة لهجمات WannaCry وغيرها |
| 3389 | RDP | هدف شائع للهجمات |
| 1433 | MSSQL | قاعدة بيانات قد تكون عرضة |

### 🛡️ الحماية من Nmap

**1. على مستوى الجدار الناري:**
```cisco
# رفض المسح من مصادر خارجية
access-list 100 deny tcp any any range 1 1024
```

**2. على مستوى الخادم:**
```bash
# استخدام iptables لمنع المسح
iptables -A INPUT -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT
```

**3. أدوات كشف المسح:**
```bash
# تثبيت portsentry
sudo apt install portsentry

# مراقبة محاولات المسح
tail -f /var/log/syslog | grep portsentry
```

### 💡 نصائح للاستخدام الآمن
- ⚠️ **لا تقم بمسح شبكات لا تملكها**
- 📝 احصل على إذن خطي قبل أي اختبار اختراق
- 🔒 استخدم بيئة مختبرية معزولة للتدريب
- 📊 وثّق جميع عمليات المسح ونتائجها

### 🎓 أمثلة تعليمية إضافية

**مسح سريع لأهم 100 منفذ:**
```bash
nmap --top-ports 100 192.168.1.50
```

**مسح مع تجنب الكشف:**
```bash
nmap -sS -T2 -f 192.168.1.50
```
- `-T2`: سرعة بطيئة (أقل إزعاجاً)
- `-f`: تجزئة الحزم

**حفظ النتائج:**
```bash
nmap -oN results.txt 192.168.1.50
nmap -oX results.xml 192.168.1.50
```

---

## 8. ICMP - اختبارات الشبكة

### 🎯 الهدف
استخدام أدوات ICMP لتشخيص واختبار الاتصال بالشبكة.

### 📋 بروتوكول ICMP
Internet Control Message Protocol - بروتوكول رسائل التحكم في الإنترنت
- Layer 3 Protocol
- يُستخدم للتشخيص وإبلاغ الأخطاء
- لا يحمل بيانات المستخدم

### 🔍 الأوامر الأساسية

#### 1. Ping - اختبار الاتصال

**الاستخدام البسيط:**
```bash
ping 8.8.8.8
```

**مع خيارات متقدمة:**
```bash
# إرسال 10 حزم فقط
ping -c 10 8.8.8.8

# تغيير حجم الحزمة
ping -s 1000 8.8.8.8

# تحديد الوقت بين الحزم (ثانية واحدة)
ping -i 1 8.8.8.8
```

**مثال على النتائج:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=11.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=12.1 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 11.8/12.1/12.4/0.3 ms
```

#### 2. Traceroute - تتبع المسار

**على Linux:**
```bash
traceroute 8.8.8.8
```

**على Windows:**
```cmd
tracert 8.8.8.8
```

**مثال على النتائج:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.245 ms  1.123 ms  1.089 ms
 2  10.0.0.1 (10.0.0.1)  8.456 ms  8.234 ms  8.123 ms
 3  172.16.0.1 (172.16.0.1)  15.678 ms  15.456 ms  15.234 ms
 4  * * *
 5  8.8.8.8 (8.8.8.8)  22.345 ms  22.123 ms  21.987 ms
```

**التحليل:**
- كل سطر يمثل موجّه (router) في المسار
- الأرقام الثلاثة: زمن الاستجابة للمحاولات الثلاث
- `* * *`: الموجّه لا يستجيب لطلبات ICMP

#### 3. MTR - تتبع متقدم

```bash
mtr 8.8.8.8
```

يجمع بين ping و traceroute مع إحصائيات مستمرة:
```
                             Packets               Pings
 Host                      Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.1.1             0.0%    10    1.2   1.3   1.1   1.5   0.1
 2. 10.0.0.1                0.0%    10    8.4   8.5   8.1   9.2   0.3
 3. 8.8.8.8                 0.0%    10   22.1  22.3  21.8  23.1   0.4
```

### 📊 أنواع رسائل ICMP

| النوع | الاسم | الاستخدام |
|-------|-------|-----------|
| Type 0 | Echo Reply | رد على Ping |
| Type 3 | Destination Unreachable | الوجهة غير متاحة |
| Type 5 | Redirect | إعادة توجيه المسار |
| Type 8 | Echo Request | طلب Ping |
| Type 11 | Time Exceeded | انتهاء TTL (يستخدمه traceroute) |

### ⚠️ اختبارات متقدمة (بيئة مختبرية فقط)

#### ICMP Flood Test

```bash
# ⚠️ للاختبار في بيئة معزولة فقط
sudo ping -f -s 65500 TARGET_IP
```

**الشرح:**
- `-f`: Flood mode (إرسال سريع جداً)
- `-s 65500`: حجم الحزمة (أقصى حد)
- **يُستخدم لاختبار تحمل الشبكة**

**النتائج المتوقعة:**
```
PING target (192.168.1.50) 65500(65528) bytes of data.
..................................
--- target ping statistics ---
10000 packets transmitted, 9856 received, 1.44% packet loss
```

#### Ping of Death (تاريخي)

```bash
# ⚠️ لا يعمل على الأنظمة الحديثة
ping -s 65510 TARGET_IP
```

**الخلفية:**
- كان يُسبب تعطل الأنظمة القديمة
- الأنظمة الحديثة محمية ضده

### 🛡️ الحماية من هجمات ICMP

#### 1. تحديد معدل ICMP

**على Linux:**
```bash
# تحديد عدد حزم ICMP في الثانية
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

#### 2. على Cisco Router

```cisco
# تحديد معدل ICMP
access-list 100 permit icmp any any echo-reply
access-list 100 deny icmp any any echo-request
interface gi0/0
ip access-group 100 in
rate-limit input access-group 100 1000000 250000 500000 conform-action transmit exceed-action drop
```

#### 3. حظر ICMP نهائياً (غير مستحسن)

```bash
# يُفقد القدرة على التشخيص
iptables -A INPUT -p icmp -j DROP
```

### 📝 حالات الاستخدام العملية

**1. تشخيص انقطاع الاتصال:**
```bash
# خطوة بخطوة
ping 192.168.1.1  # البوابة المحلية
ping 8.8.8.8      # خادم خارجي
traceroute 8.8.8.8  # تحديد نقطة الانقطاع
```

**2. قياس جودة الاتصال:**
```bash
# مراقبة لمدة 5 دقائق
ping -c 300 -i 1 8.8.8.8 | tee ping_results.txt

# تحليل النتائج
grep "time=" ping_results.txt | awk '{print $7}' | sed 's/time=//' | sort -n
```

**3. اختبار MTU (Maximum Transmission Unit):**
```bash
# ابدأ بحجم كبير وقلله تدريجياً
ping -M do -s 1472 8.8.8.8

# إذا فشل، قلل الحجم حتى ينجح
ping -M do -s 1400 8.8.8.8
```

### 💡 نصائح مهمة
- ICMP ليس آمناً 100% - يمكن استخدامه في الهجمات
- بعض الشبكات تحظر ICMP للأمان
- استخدم أدوات بديلة (TCP ping) عند الحاجة
- لا تعتمد على ICMP فقط للمراقبة

---

## 9. SYN Flooding Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم SYN Flood وتأثيره على الخوادم.

### ⚠️ تحذير مهم جداً
```
┌─────────────────────────────────────────────────┐
│  ⚠️  هذا المحتوى لأغراض تعليمية فقط  ⚠️       │
│                                                  │
│  تنفيذ هذا الهجوم على أنظمة حقيقية:           │
│  • جريمة يعاقب عليها القانون                   │
│  • يتطلب إذن خطي مسبق                          │
│  • يُسمح فقط في بيئة مختبرية معزولة           │
└─────────────────────────────────────────────────┘
```

### 📋 نظرة عامة على الهجوم

#### المصافحة الثلاثية الطبيعية (TCP Three-Way Handshake)

```
Client                    Server
  |                         |
  |-------- SYN ---------->|  [1]
  |<----- SYN-ACK ---------|  [2]
  |-------- ACK ---------->|  [3]
  |                         |
  |==== Connection Open ====|
```

#### هجوم SYN Flood

```
Attacker                  Server
  |                         |
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |      (thousands)        |
  |                         |
  |   ❌ No ACK Sent ❌    |
  |                         |
  |                    [Server]
  |                   SYN Queue
  |                   ┌─────────┐
  |                   │ SYN #1  │ ⏳
  |                   │ SYN #2  │ ⏳
  |                   │ SYN #3  │ ⏳
  |                   │ SYN #... │ ⏳
  |                   │ FULL!   │ 🔴
  |                   └─────────┘
```

### 🔬 التطبيق العملي (بيئة مختبرية فقط)

#### البيئة المطلوبة
- آلة افتراضية للمهاجم (Kali Linux)
- خادم اختبار معزول
- شبكة محلية منعزلة تماماً

#### الأداة: hping3

**التثبيت:**
```bash
sudo apt update
sudo apt install hping3
```

**الهجوم الأساسي:**
```bash
sudo hping3 -S -p 80 --flood TARGET_IP
```

**الشرح:**
- `-S`: إرسال حزم SYN فقط
- `-p 80`: استهداف المنفذ 80 (HTTP)
- `--flood`: إرسال أسرع ما يمكن (بدون انتظار)
- `TARGET_IP`: عنوان الخادم المستهدف

**هجوم متقدم مع عناوين مزيفة:**
```bash
sudo hping3 -S -p 80 --flood --rand-source TARGET_IP
```

- `--rand-source`: عناوين IP مصدر عشوائية
- يُصعّب تتبع المهاجم
- يتجاوز بعض آليات الحماية البسيطة

### 📊 المراقبة على الخادم المستهدف

#### عرض الاتصالات نصف المفتوحة

```bash
netstat -ant | grep SYN_RECV
```

**مثال على النتائج أثناء الهجوم:**
```
tcp    0    0 192.168.1.50:80    10.0.0.100:45123    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.101:45124    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.102:45125    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.103:45126    SYN_RECV
... (مئات أو آلاف السطور)
```

#### مراقبة استهلاك الموارد

```bash
# عرض قائمة الانتظار
ss -tan | grep SYN-RECV | wc -l

# مراقبة الذاكرة
free -h

# مراقبة CPU
top
```

**مؤشرات الهجوم:**
```
SYN-RECV connections: 5000+  🔴 (طبيعي: < 100)
Memory usage: 95%             🔴 (طبيعي: < 70%)
CPU load: 100%                🔴 (طبيعي: < 50%)
```

### 🛡️ آليات الحماية

#### 1. SYN Cookies

**تفعيل SYN Cookies على Linux:**
```bash
# التحقق من الحالة الحالية
cat /proc/sys/net/ipv4/tcp_syncookies

# التفعيل (1 = enabled)
sudo sysctl -w net.ipv4.tcp_syncookies=1

# جعله دائماً
echo "net.ipv4.tcp_syncookies = 1" | sudo tee -a /etc/sysctl.conf
```

**كيف تعمل:**
```
بدلاً من تخزين الاتصال في الذاكرة:
1. الخادم يحسب قيمة مشفرة (cookie)
2. يُرسلها في SYN-ACK
3. لا يحفظ شيء في الذاكرة
4. عند استلام ACK، يتحقق من الـ cookie
5. إن صح، يُكمل الاتصال

النتيجة: لا استنزاف للذاكرة ✅
```

#### 2. تحديد معدل الاتصالات الجديدة

**باستخدام iptables:**
```bash
# السماح بـ 5 اتصالات جديدة فقط في الثانية من نفس IP
iptables -A INPUT -p tcp --syn -m limit --limit 5/s --limit-burst 10 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP
```

**باستخدام nftables:**
```bash
nft add rule ip filter input tcp flags syn ct state new limit rate 5/second accept
```

#### 3. زيادة حجم قائمة SYN

```bash
# زيادة الحد الأقصى للاتصالات نصف المفتوحة
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# تقليل وقت الانتظار
sudo sysctl -w net.ipv4.tcp_synack_retries=2
```

#### 4. استخدام Firewall متقدم

**Cisco ASA:**
```cisco
! تحديد عدد الاتصالات المتزامنة
access-list OUTSIDE_IN extended permit tcp any host 203.0.113.50 eq www
class-map SYN-ATTACK
 match access-list OUTSIDE_IN
policy-map OUTSIDE_POLICY
 class SYN-ATTACK
  set connection conn-max 100 embryonic-conn-max 50
service-policy OUTSIDE_POLICY interface outside
```

**pfSense/OPNsense:**
- تفعيل "SYN Proxy"
- ضبط "Maximum States" و "Maximum Source Nodes"

#### 5. استخدام CDN/DDoS Protection

خدمات مثل:
- Cloudflare
- AWS Shield
- Akamai

تقوم بـ:
- امتصاص الهجمات الضخمة
- توزيع الحمل
- فلترة الحركة الخبيثة

### 📈 التأثير على الخادم

#### مراحل الهجوم

**المرحلة 1: البداية (0-30 ثانية)**
```
SYN Queue: 20% full
Response Time: 100ms → 500ms
```

**المرحلة 2: التصعيد (30-60 ثانية)**
```
SYN Queue: 80% full
Response Time: 500ms → 3000ms
New connections: بطيئة جداً
```

**المرحلة 3: الانهيار (60+ ثانية)**
```
SYN Queue: 100% full
Response Time: timeout
New connections: مرفوضة بالكامل
Service: غير متاح 🔴
```

### 🔍 كشف الهجوم

#### علامات التحذير

```bash
# ارتفاع مفاجئ في SYN_RECV
watch -n 1 'netstat -ant | grep SYN_RECV | wc -l'

# تنبيه عند تجاوز حد معين
while true; do
  COUNT=$(netstat -ant | grep SYN_RECV | wc -l)
  if [ $COUNT -gt 1000 ]; then
    echo "⚠️ Possible SYN Flood: $COUNT connections"
  fi
  sleep 5
done
```

#### تحليل السجلات

```bash
# عرض أكثر IP مصدرية نشاطاً
netstat -ant | grep SYN_RECV | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -10
```

### 💡 الدروس المستفادة

**للمدافعين:**
1. ✅ فعّل SYN Cookies دائماً
2. ✅ راقب معدلات الاتصال بشكل مستمر
3. ✅ استخدم Firewall مع إمكانيات DDoS protection
4. ✅ احتفظ بخطة استجابة جاهزة

**للمهاجمين الأخلاقيين:**
1. 📝 احصل على إذن خطي قبل الاختبار
2. 🔒 استخدم بيئة معزولة تماماً
3. ⏰ حدد نافذة زمنية محدودة للاختبار
4. 📊 وثّق النتائج بشكل احترافي

---

## 10. UDP Flood Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم UDP Flood والفرق بينه وبين SYN Flood.

### ⚠️ تحذير قانوني
```
┌──────────────────────────────────────────────────┐
│  🚨 هذا المحتوى للتعليم والأبحاث الأمنية فقط  │
│                                                   │
│  القيام بهذا الهجوم دون إذن هو:                │
│  • جريمة إلكترونية                              │
│  • يُعاقب عليها بالسجن والغرامة                │
│  • يُسمح فقط في بيئة مختبرية مملوكة لك         │
└──────────────────────────────────────────────────┘
```

### 📋 الفرق بين TCP و UDP

| الميزة | TCP | UDP |
|--------|-----|-----|
| الاتصال | Connection-oriented | Connectionless |
| المصافحة | نعم (3-way handshake) | لا |
| الموثوقية | مضمون الوصول | غير مضمون |
| السرعة | أبطأ | أسرع |
| الترتيب | مرتب | غير مرتب |

### 🔬 آلية هجوم UDP Flood

```
Attacker                    Target Server
  |                              |
  |------- UDP Packet ---------->| Port 53 (DNS)
  |------- UDP Packet ---------->| Port 161 (SNMP)
  |------- UDP Packet ---------->| Port 12345 (Random)
  |------- UDP Packet ---------->| Port 9999 (Closed)
  |       (thousands/sec)        |
  |                              |
  |                         [Server Process]
  |                         1. تلقي الحزمة
  |                         2. فحص المنفذ
  |                         3. المنفذ مغلق؟
  |                         4. إرسال ICMP Port Unreachable
  |                              |
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |                              |
  |                    [Result: Resource Exhaustion]
  |                    • CPU: 100%
  |                    • Bandwidth: Saturated
  |                    • Service: Unavailable
```

### 🔬 التطبيق العملي (بيئة مختبرية معزولة فقط)

#### البيئة المطلوبة
- شبكة مختبرية معزولة تماماً
- خادم اختبار مخصص
- إذن خطي موثق

#### استخدام hping3

**الهجوم الأساسي:**
```bash
sudo hping3 --udp -p 53 --flood TARGET_IP
```

**الشرح:**
- `--udp`: استخدام بروتوكول UDP
- `-p 53`: استهداف منفذ DNS
- `--flood`: إرسال بأقصى سرعة ممكنة

**هجوم مع منافذ عشوائية:**
```bash
sudo hping3 --udp --rand-dest --flood TARGET_IP
```

- `--rand-dest`: منافذ وجهة عشوائية
- يُضعف الخادم أكثر (يفحص كل منفذ)

**هجوم مع حجم حزمة كبير:**
```bash
sudo hping3 --udp -p 53 --flood -d 65500 TARGET_IP
```

- `-d 65500`: حجم البيانات (أقصى حد لـ UDP)
- يستهلك bandwidth أكثر

### 📊 المراقبة على الخادم المستهدف

#### عرض إحصائيات UDP

```bash
netstat -s | grep UDP
```

**مثال على النتائج أثناء الهجوم:**
```
UDP:
    125847 packets received
    98234 packets to unknown port received
    89123 packet receive errors
    45678 packets sent
    RcvbufErrors: 12345
    SndbufErrors: 0
    InErrors: 89123
```

#### مراقبة الحزم في الوقت الفعلي

```bash
# عرض معدل حزم UDP
watch -n 1 'netstat -su | grep "packets received"'

# مراقبة منافذ محددة
tcpdump -i eth0 udp port 53 -nn
```

**مؤشرات الهجوم:**
```
UDP packets/sec: 50,000+     🔴 (طبيعي: < 1,000)
Port Unreachable: 40,000+    🔴 (طبيعي: < 10)
Bandwidth usage: 90%+        🔴 (طبيعي: < 50%)
CPU usage: 100%              🔴 (طبيعي: < 30%)
```

#### تحديد أكثر المنافذ المستهدفة

```bash
tcpdump -i eth0 -nn udp -c 1000 | awk '{print $5}' | cut -d'.' -f5 | sort | uniq -c | sort -nr | head -10
```

### 🛡️ آليات الحماية

#### 1. Rate Limiting على Linux

```bash
# تحديد معدل حزم UDP من كل IP
iptables -A INPUT -p udp -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p udp -j DROP

# حماية منافذ محددة
iptables -A INPUT -p udp --dport 53 -m limit --limit 50/s -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j DROP
```

#### 2. تعطيل ICMP Port Unreachable

```bash
# منع إرسال رسائل Port Unreachable
iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP

# أو عبر sysctl
sysctl -w net.ipv4.icmp_ignore_bogus_error_responses=1
```

**الفائدة:**
- يوفر موارد CPU
- يمنع استنزاف bandwidth بالردود

#### 3. حماية على مستوى الخادم

```bash
# زيادة حجم buffer للـ UDP
sysctl -w net.core.rmem_max=26214400
sysctl -w net.core.rmem_default=26214400

# تحسين معالجة الحزم
sysctl -w net.core.netdev_max_backlog=5000
```

#### 4. Firewall Rules على Cisco

```cisco
! تحديد معدل UDP
access-list 110 permit udp any host 203.0.113.50 eq domain
access-list 110 deny udp any any

interface GigabitEthernet0/0
 ip access-group 110 in
 rate-limit input access-group 110 10000000 1000000 2000000 conform-action transmit exceed-action drop
```

#### 5. استخدام fail2ban

```bash
# تثبيت fail2ban
sudo apt install fail2ban

# إنشاء قاعدة لـ UDP flood
sudo nano /etc/fail2ban/filter.d/udp-flood.conf
```

**محتوى الملف:**
```ini
[Definition]
failregex = .*kernel.*UDP.*SRC=<HOST>.*
ignoreregex =
```

**التفعيل:**
```ini
# /etc/fail2ban/jail.local
[udp-flood]
enabled = true
filter = udp-flood
logpath = /var/log/syslog
maxretry = 100
findtime = 60
bantime = 3600
```

### 📈 المنافذ الشائعة المستهدفة

| المنفذ | الخدمة | سبب الاستهداف |
|--------|---------|---------------|
| 53 | DNS | حيوي للشبكة، يولد استجابات |
| 123 | NTP | يمكن تضخيمه (Amplification) |
| 161 | SNMP | إدارة، غالباً مفتوح |
| 1900 | SSDP | UPnP، شائع في الأجهزة المنزلية |
: SHA-256
- `$6# تدريبات عملية في أمن الشبكات
## Network Security - Practical Labs

---

## 📑 جدول المحتويات

1. [NAT/PAT - ترجمة عناوين الشبكة](#1-natpat---ترجمة-عناوين-الشبكة)
2. [ACL - قوائم التحكم بالوصول](#2-acl---قوائم-التحكم-بالوصول)
3. [FTP وآلية Established](#3-ftp-وآلية-established)
4. [WAF و Load Balancer](#4-waf-و-load-balancer)
5. [Cisco Firepower - تحليل البرمجيات الخبيثة](#5-cisco-firepower---تحليل-البرمجيات-الخبيثة)
6. [ARP - بروتوكول تحليل العناوين](#6-arp---بروتوكول-تحليل-العناوين)
7. [Nmap - مسح الشبكات](#7-nmap---مسح-الشبكات)
8. [ICMP - اختبارات الشبكة](#8-icmp---اختبارات-الشبكة)
9. [SYN Flooding Attack](#9-syn-flooding-attack)
10. [UDP Flood Attack](#10-udp-flood-attack)
11. [Nmap Port Scanning](#11-nmap-port-scanning)
12. [John the Ripper - كسر كلمات المرور](#12-john-the-ripper---كسر-كلمات-المرور)
13. [DHCP Snooping](#13-dhcp-snooping)

---

## 1. NAT/PAT - ترجمة عناوين الشبكة

### 🎯 الهدف
فهم آلية عمل NAT/PAT في ترجمة العناوين الخاصة إلى عناوين عامة.

### 📋 السيناريو
جهاز داخلي بعنوان `10.0.0.5` يريد تصفح الإنترنت عبر جدار ناري يستخدم NAT.

### 🔄 خطوات العمل

1. **الطلب الصادر:**
   - الجهاز الداخلي `10.0.0.5` يرسل طلب HTTP
   - الجدار الناري يترجم العنوان إلى `203.0.113.2:10500`
   - العنوان `203.0.113.2` هو العنوان العام
   - المنفذ `10500` يُستخدم لتمييز الاتصال

2. **الرد الوارد:**
   - عند وصول الرد من الإنترنت إلى `203.0.113.2:10500`
   - يعيد NAT الترجمة العكسية إلى `10.0.0.5` بالمنفذ الأصلي
   - يتم تسليم البيانات للجهاز الداخلي

### ✅ الفائدة
- مئات الأجهزة يمكنها مشاركة عنوان عام واحد
- لا يوجد تضارب بفضل استخدام أرقام المنافذ (Port Numbers)
- توفير في عناوين IPv4 العامة

### 📝 ملاحظات
- PAT (Port Address Translation) هو نوع من NAT
- يُسمى أيضاً NAT Overload
- الجدول الداخلي يحتفظ بالربط بين العناوين الداخلية والمنافذ

---

## 2. ACL - قوائم التحكم بالوصول

### 🎯 الهدف
تطبيق سياسات أمنية باستخدام قوائم التحكم بالوصول (Access Control Lists).

### 📋 السيناريو
لديك:
- **Web Server** في DMZ (Demilitarized Zone)
- **Database Server** داخل الشبكة الداخلية

> 💡 **DMZ:** منطقة منزوعة السلاح، وهي منطقة بين الشبكة الداخلية والإنترنت لتقليل المخاطر الأمنية.

### 🔧 التكوين العملي

```cisco
access-list 100 permit tcp any 10.10.10.10 eq www
access-list 100 permit tcp any 10.10.10.10 eq 443
access-list 100 permit tcp any 10.10.10.12 eq ftp
access-list 100 permit tcp any 10.10.10.12 eq ftp-data
access-list 100 deny ip any any log

interface gi0/1
ip access-group 100 in
```

### 📊 شرح الأوامر

| الأمر | الوظيفة |
|------|---------|
| `permit tcp any 10.10.10.10 eq www` | السماح بالوصول لخادم الويب عبر HTTP (منفذ 80) |
| `permit tcp any 10.10.10.10 eq 443` | السماح بالوصول عبر HTTPS (منفذ 443) |
| `permit tcp any 10.10.10.12 eq ftp` | السماح بـ FTP Control (منفذ 21) |
| `permit tcp any 10.10.10.12 eq ftp-data` | السماح بـ FTP Data (منفذ 20) |
| `deny ip any any log` | حظر أي حركة أخرى مع التسجيل |
| `ip access-group 100 in` | تطبيق القائمة على الواجهة |

### 🎯 الهدف الأمني
- السماح بالوصول فقط للمنافذ المطلوبة
- منع الوصول المباشر لخادم قاعدة البيانات
- تسجيل المحاولات المشبوهة
- تقليل نطاق الهجوم (Attack Surface)

### ⚠️ ملاحظات مهمة
- الوصول لقاعدة البيانات يجب أن يتم فقط من خلال خادم الويب
- ترتيب القواعد مهم جداً (من الأعلى للأسفل)
- القاعدة الضمنية في النهاية: `deny any any`

---

## 3. FTP وآلية Established

### 🎯 الهدف
فهم الفرق بين FTP Passive و Active Mode وتأثيره على ACL.

### 📋 بنية FTP
بروتوكول FTP يستخدم قناتين منفصلتين:
- **Port 21:** قناة التحكم (Control Channel)
- **Port 20:** قناة البيانات (Data Channel)

### 🔄 FTP Passive Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Random High Port (Data)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يطلب وضع Passive
3. الخادم يرد بمنفذ عشوائي عالي
4. العميل يفتح اتصال البيانات نحو هذا المنفذ
5. كل الردود تحتوي على بت ACK ← **مسموح بها** ✅

### 🔄 FTP Active Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Port 20 (Data initiated by server)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يخبر الخادم بمنفذ معين لاستقبال البيانات
3. **الخادم يبدأ اتصال البيانات** من المنفذ 20
4. أول حزمة من الخادم تحتوي على SYN فقط ← **تُرفض** 🚫

### ⚠️ المشكلة مع Established

```cisco
access-list 110 permit tcp any any established
```

هذه القاعدة تسمح فقط بالحزم التي تحتوي على:
- بت ACK
- أو بت RST

**النتيجة:**
- في Active Mode، الخادم يرسل SYN أولاً
- هذه الحزمة لا تحقق شرط "established"
- الاتصال يفشل 🚫

### ✅ الحل الأمثل
استخدام جدار ناري حقيقي (Stateful Firewall) قادر على:
- تتبع الجلسات النشطة
- فهم البروتوكولات المعقدة مثل FTP
- السماح بالاتصالات الديناميكية تلقائياً

### 📝 الخلاصة
- `established` لا يكفي للبروتوكولات المعقدة
- FTP Passive Mode أكثر توافقاً مع الجدران النارية
- Stateful Firewalls ضرورية للبيئات الإنتاجية

---

## 4. WAF و Load Balancer

### 🎯 الهدف
بناء بنية تحتية آمنة وقابلة للتوسع لموقع ويب.

### 📋 السيناريو
موقع ويب يحتوي على عدة خوادم Web Servers خلف Load Balancer، مع حماية من هجمات XSS.

### 🏗️ البنية المعمارية

```
[Internet]
    |
    ↓
[WAF - Web Application Firewall]
    |
    ↓
[Load Balancer]
    |
    ↓------------------↓------------------↓------------------↓
[Web Server 1]   [Web Server 2]   [Web Server 3]   [Web Server 4]
```

### ⚖️ مهام Load Balancer

**خوارزمية التوزيع:**
- **Round Robin:** توزيع متساوٍ على جميع الخوادم
- دورة منتظمة: Server 1 → Server 2 → Server 3 → Server 4 → تكرار

**إدارة الأعطال:**
1. مراقبة صحة الخوادم (Health Checks)
2. عند فشل أحد الخوادم:
   - إزالته من قائمة التوزيع تلقائياً
   - تحويل طلباته إلى الخوادم السليمة
3. عند عودة الخادم: إعادة إدراجه تلقائياً

### 🛡️ مهام WAF (كـ Reverse Proxy)

**الحماية من XSS:**
```javascript
// مثال على محاولة هجوم XSS
<script>alert('XSS Attack')</script>
```

**آلية الحماية:**
1. فحص كل طلب HTTP قبل وصوله للخوادم
2. كشف الشيفرات الخبيثة:
   - JavaScript مشبوه
   - SQL Injection
   - Command Injection
3. منع الطلبات المشبوهة فوراً ⛔
4. منع تسريب البيانات الحساسة في الاستجابات

### 🔍 سياسات الأمان

| نوع الحماية | الوظيفة |
|-------------|---------|
| XSS Prevention | منع حقن JavaScript |
| SQL Injection | حماية قواعد البيانات |
| CSRF Protection | منع الطلبات المزورة |
| Rate Limiting | الحد من معدل الطلبات |
| IP Blacklisting | حظر IP المشبوهة |

### ✅ المزايا المحققة

**الأداء:**
- ⚡ توزيع الأحمال بشكل متوازن
- 🔄 تحمل الأعطال (Fault Tolerance)
- 📈 قابلية التوسع الأفقي

**الأمان:**
- 🛡️ حماية متقدمة من WAF
- 🔒 فحص جميع الطلبات
- 📝 تسجيل المحاولات المشبوهة
- 🚫 حظر الهجمات في الوقت الحقيقي

### 📊 مثال على سجلات WAF

```
2025-10-26 10:15:32 | BLOCKED | XSS Attempt | IP: 192.168.1.100
2025-10-26 10:16:45 | BLOCKED | SQL Injection | IP: 10.0.0.50
2025-10-26 10:17:12 | ALLOWED | Normal Request | IP: 172.16.0.25
```

---

## 5. Cisco Firepower - تحليل البرمجيات الخبيثة

### 🎯 الهدف
فهم كيفية عمل أنظمة كشف البرمجيات الخبيثة المتقدمة وتتبع انتشارها.

### 📋 السيناريو
ملف باسم `tool.exe` تم اكتشافه ضمن حركة المرور على الشبكة.

### 🔍 مراحل الكشف والتحليل

#### المرحلة 1: الفحص الأولي
1. **اعتراض الملف:** جهاز Cisco Firepower يفحص جميع الملفات المارة
2. **حساب البصمة:** استخراج بصمة SHA-256 للملف
   ```
   SHA-256: a3f7b8c9d2e1f4a6b5c8d7e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
   ```
3. **المقارنة:** مقارنة البصمة مع قاعدة بيانات التهديدات

#### المرحلة 2: التصنيف
```
Status: ⚠️ MALWARE DETECTED
Threat Name: Generic.Trojan.Downloader
Threat Score: 95/100 (High)
Category: Trojan
First Seen: 2025-10-25 14:30:22
```

#### المرحلة 3: تحديد الأجهزة المصابة
الأجهزة التالية تم تحديدها كمصابة:
- 🖥️ `192.168.10.90`
- 🖥️ `192.168.133.50`

### 📈 Trajectory Map (خريطة المسار)

```
Time →
Device 1: --------[*]==========[*]-------
                   ↓            ↓
Device 2: --------[*]----[*]----[*]------
                         ↓      ↓
Device 3: ---------------[*]=====[*]-----
```

**رموز الخريطة:**
- `[*]` نقطة انتقال الملف
- `=` فترة نشاط الملف على الجهاز
- `↓` اتجاه انتقال العدوى (من جهاز لآخر)
- `-` فترة خمول

### 📊 جدول الأحداث (Events Table)

| الوقت | الجهاز المصدر | الجهاز الهدف | الحدث | الحالة |
|------|---------------|--------------|--------|--------|
| 14:30:22 | External | 192.168.10.90 | File Download | Infected |
| 14:35:18 | 192.168.10.90 | 192.168.133.50 | File Transfer | Infected |
| 14:42:33 | 192.168.133.50 | Network Share | Propagation Attempt | Blocked |

### 🎯 استخدامات Trajectory Map

1. **تحديد المصدر الأولي:**
   - أول جهاز تلقى الملف
   - نقطة الدخول إلى الشبكة

2. **تتبع مسار الانتشار:**
   - الأجهزة الوسيطة
   - طرق الانتقال المستخدمة

3. **تقييم نطاق الإصابة:**
   - عدد الأجهزة المصابة
   - الأقسام المتأثرة

4. **التخطيط للاستجابة:**
   - أولوية المعالجة
   - خطة العزل والإزالة

### 🚨 خطة الاستجابة للحادث

**الخطوات الفورية:**
1. ✅ عزل الأجهزة المصابة عن الشبكة
2. ✅ حظر بصمة الملف على جميع الأجهزة
3. ✅ فحص شامل للأجهزة المجاورة
4. ✅ تحليل سجلات النشاط

**الإجراءات التصحيحية:**
1. 🔧 إزالة البرمجية الخبيثة
2. 🔧 تحديث تعريفات مكافح الفيروسات
3. 🔧 تصحيح الثغرات المستغلة
4. 🔧 استعادة النظام من نسخة احتياطية نظيفة

**المتابعة:**
1. 📝 توثيق الحادث
2. 📝 تحديث السياسات الأمنية
3. 📝 تدريب المستخدمين
4. 📝 مراقبة معززة لفترة

### 💡 الدروس المستفادة
- أهمية الفحص العميق لحركة المرور
- ضرورة التسجيل الشامل للأحداث
- قيمة أدوات التحليل البصري
- أهمية الاستجابة السريعة

---

## 6. ARP - بروتوكول تحليل العناوين

### 🎯 الهدف
فهم آلية عمل بروتوكول ARP ومراقبة الجدول الخاص به.

### 📋 نظرة عامة
ARP (Address Resolution Protocol) يربط بين:
- **عنوان IP** (Layer 3)
- **عنوان MAC** (Layer 2)

### 🔍 التطبيق العملي

#### الخطوة 1: عرض جدول ARP

**على Windows:**
```cmd
arp -a
```

**على Linux/macOS:**
```bash
arp -a
```

#### الخطوة 2: قراءة النتائج

```
Interface: 192.168.1.100 --- 0x4
  Internet Address      Physical Address      Type
  192.168.1.1          00-1A-2B-3C-4D-5E     dynamic
  192.168.1.10         00-AA-BB-CC-DD-EE     dynamic
  192.168.1.255        FF-FF-FF-FF-FF-FF     static
  224.0.0.22           01-00-5E-00-00-16     static
```

### 📊 تحليل الجدول

| المكون | الشرح |
|--------|--------|
| **Internet Address** | عنوان IP للجهاز |
| **Physical Address** | عنوان MAC للجهاز |
| **Type** | نوع الإدخال |

### 🏷️ أنواع الإدخالات

**Dynamic:**
- يتم تعلمه تلقائياً من حركة المرور
- يُحذف بعد فترة عدم النشاط (عادة 2-10 دقائق)
- الأكثر شيوعاً في الشبكات

**Static:**
- يتم إضافته يدوياً بواسطة المدير
- لا يُحذف تلقائياً
- يُستخدم للأجهزة الحساسة

### 🔄 عناوين خاصة

**Broadcast:**
```
192.168.1.255 → FF-FF-FF-FF-FF-FF
```
- يُرسل لجميع الأجهزة في الشبكة

**Multicast:**
```
224.0.0.22 → 01-00-5E-00-00-16
```
- يُرسل لمجموعة محددة من الأجهزة

### 🧹 مسح الكاش

**مسح جميع الإدخالات (Windows):**
```cmd
arp -d *
```

**مسح إدخال محدد:**
```cmd
arp -d 192.168.1.10
```

**على Linux:**
```bash
sudo ip -s -s neigh flush all
```

### 📝 إضافة إدخال ثابت

**Windows:**
```cmd
arp -s 192.168.1.50 00-11-22-33-44-55
```

**Linux:**
```bash
sudo arp -s 192.168.1.50 00:11:22:33:44:55
```

### 🔍 حالات الاستخدام

**1. تشخيص مشاكل الاتصال:**
```bash
# فحص وجود الجهاز في الجدول
arp -a | grep 192.168.1.10

# إذا لم يظهر، المشكلة قد تكون في Layer 2
```

**2. كشف تضارب IP:**
```bash
# إذا ظهر نفس IP بعنواني MAC مختلفين
192.168.1.50    00-AA-BB-CC-DD-EE
192.168.1.50    00-11-22-33-44-55  # ⚠️ تضارب!
```

**3. مراقبة الأجهزة النشطة:**
```bash
# عد الأجهزة المتصلة
arp -a | grep dynamic | wc -l
```

### ⚠️ ملاحظات أمنية

**ARP Spoofing/Poisoning:**
- مهاجم يرسل رسائل ARP مزيفة
- يخدع الأجهزة لإرسال البيانات له
- **الحماية:** ARP Inspection على السويتشات

**الوقاية:**
```cisco
# تفعيل Dynamic ARP Inspection
Switch(config)# ip arp inspection vlan 10
Switch(config)# interface gi0/1
Switch(config-if)# ip arp inspection trust
```

### 💡 نصائح عملية
- راقب جدول ARP بشكل دوري
- انتبه للتغييرات المفاجئة في عناوين MAC
- استخدم إدخالات ثابتة للأجهزة الحساسة
- فعّل ARP Inspection في البيئات الحساسة

---

## 7. Nmap - مسح الشبكات

### 🎯 الهدف
استخدام Nmap لاكتشاف الأجهزة والخدمات النشطة على الشبكة.

### 📋 المتطلبات
- نظام Linux (يفضل Kali Linux)
- صلاحيات root للمسح المتقدم
- **بيئة اختبارية فقط** ⚠️

### 🔍 أنواع المسح

#### 1. اكتشاف الأجهزة النشطة (Host Discovery)

```bash
nmap -sn 192.168.1.0/24
```

**الشرح:**
- `-sn`: Ping Scan (بدون فحص المنافذ)
- يستخدم ICMP Echo Request
- يُظهر الأجهزة المتصلة والنشطة

**مثال على النتائج:**
```
Starting Nmap 7.94
Nmap scan report for 192.168.1.1
Host is up (0.0012s latency).

Nmap scan report for 192.168.1.50
Host is up (0.0034s latency).

Nmap scan report for 192.168.1.100
Host is up (0.0028s latency).

Nmap done: 256 IP addresses (3 hosts up) scanned in 2.85 seconds
```

#### 2. فحص المنافذ (Port Scanning)

**Basic TCP Scan:**
```bash
nmap 192.168.1.50
```

**Comprehensive Scan:**
```bash
nmap -sS -sV -O 192.168.1.50
```

**الشرح:**
- `-sS`: SYN Stealth Scan (أقل قابلية للكشف)
- `-sV`: كشف إصدارات الخدمات
- `-O`: تخمين نظام التشغيل

**مثال على النتائج:**
```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 8.2p1
80/tcp   open  http       Apache httpd 2.4.41
443/tcp  open  ssl/http   Apache httpd 2.4.41
3306/tcp open  mysql      MySQL 5.7.33

OS details: Linux 3.2 - 4.9
```

#### 3. فحص جميع المنافذ

```bash
nmap -p- 192.168.1.50
```

- `-p-`: فحص جميع المنافذ (1-65535)
- يستغرق وقتاً أطول

#### 4. فحص منافذ محددة

```bash
nmap -p 21,22,80,443,3306 192.168.1.50
```

### 🎭 أنواع مسح المنافذ

#### TCP SYN Scan (الأكثر شيوعاً)
```bash
nmap -sS 192.168.1.50
```
- يرسل SYN فقط
- لا يُكمل المصافحة الثلاثية
- أقل قابلية للكشف

#### TCP Connect Scan
```bash
nmap -sT 192.168.1.50
```
- يُكمل المصافحة الثلاثية
- لا يحتاج صلاحيات root
- أكثر قابلية للكشف

#### UDP Scan
```bash
nmap -sU 192.168.1.50
```
- فحص منافذ UDP
- بطيء جداً
- مهم لخدمات مثل DNS, DHCP

### 🔬 فحص الثغرات

```bash
nmap --script vuln 192.168.1.50
```

يستخدم NSE (Nmap Scripting Engine) للبحث عن ثغرات معروفة.

### 📊 تقييم المخاطر

#### منافذ عالية الخطورة:

| المنفذ | الخدمة | الخطر |
|--------|---------|-------|
| 21 | FTP | نقل بيانات بدون تشفير |
| 23 | Telnet | إدارة عن بعد بدون تشفير |
| 445 | SMB | عرضة لهجمات WannaCry وغيرها |
| 3389 | RDP | هدف شائع للهجمات |
| 1433 | MSSQL | قاعدة بيانات قد تكون عرضة |

### 🛡️ الحماية من Nmap

**1. على مستوى الجدار الناري:**
```cisco
# رفض المسح من مصادر خارجية
access-list 100 deny tcp any any range 1 1024
```

**2. على مستوى الخادم:**
```bash
# استخدام iptables لمنع المسح
iptables -A INPUT -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT
```

**3. أدوات كشف المسح:**
```bash
# تثبيت portsentry
sudo apt install portsentry

# مراقبة محاولات المسح
tail -f /var/log/syslog | grep portsentry
```

### 💡 نصائح للاستخدام الآمن
- ⚠️ **لا تقم بمسح شبكات لا تملكها**
- 📝 احصل على إذن خطي قبل أي اختبار اختراق
- 🔒 استخدم بيئة مختبرية معزولة للتدريب
- 📊 وثّق جميع عمليات المسح ونتائجها

### 🎓 أمثلة تعليمية إضافية

**مسح سريع لأهم 100 منفذ:**
```bash
nmap --top-ports 100 192.168.1.50
```

**مسح مع تجنب الكشف:**
```bash
nmap -sS -T2 -f 192.168.1.50
```
- `-T2`: سرعة بطيئة (أقل إزعاجاً)
- `-f`: تجزئة الحزم

**حفظ النتائج:**
```bash
nmap -oN results.txt 192.168.1.50
nmap -oX results.xml 192.168.1.50
```

---

## 8. ICMP - اختبارات الشبكة

### 🎯 الهدف
استخدام أدوات ICMP لتشخيص واختبار الاتصال بالشبكة.

### 📋 بروتوكول ICMP
Internet Control Message Protocol - بروتوكول رسائل التحكم في الإنترنت
- Layer 3 Protocol
- يُستخدم للتشخيص وإبلاغ الأخطاء
- لا يحمل بيانات المستخدم

### 🔍 الأوامر الأساسية

#### 1. Ping - اختبار الاتصال

**الاستخدام البسيط:**
```bash
ping 8.8.8.8
```

**مع خيارات متقدمة:**
```bash
# إرسال 10 حزم فقط
ping -c 10 8.8.8.8

# تغيير حجم الحزمة
ping -s 1000 8.8.8.8

# تحديد الوقت بين الحزم (ثانية واحدة)
ping -i 1 8.8.8.8
```

**مثال على النتائج:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=11.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=12.1 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 11.8/12.1/12.4/0.3 ms
```

#### 2. Traceroute - تتبع المسار

**على Linux:**
```bash
traceroute 8.8.8.8
```

**على Windows:**
```cmd
tracert 8.8.8.8
```

**مثال على النتائج:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.245 ms  1.123 ms  1.089 ms
 2  10.0.0.1 (10.0.0.1)  8.456 ms  8.234 ms  8.123 ms
 3  172.16.0.1 (172.16.0.1)  15.678 ms  15.456 ms  15.234 ms
 4  * * *
 5  8.8.8.8 (8.8.8.8)  22.345 ms  22.123 ms  21.987 ms
```

**التحليل:**
- كل سطر يمثل موجّه (router) في المسار
- الأرقام الثلاثة: زمن الاستجابة للمحاولات الثلاث
- `* * *`: الموجّه لا يستجيب لطلبات ICMP

#### 3. MTR - تتبع متقدم

```bash
mtr 8.8.8.8
```

يجمع بين ping و traceroute مع إحصائيات مستمرة:
```
                             Packets               Pings
 Host                      Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.1.1             0.0%    10    1.2   1.3   1.1   1.5   0.1
 2. 10.0.0.1                0.0%    10    8.4   8.5   8.1   9.2   0.3
 3. 8.8.8.8                 0.0%    10   22.1  22.3  21.8  23.1   0.4
```

### 📊 أنواع رسائل ICMP

| النوع | الاسم | الاستخدام |
|-------|-------|-----------|
| Type 0 | Echo Reply | رد على Ping |
| Type 3 | Destination Unreachable | الوجهة غير متاحة |
| Type 5 | Redirect | إعادة توجيه المسار |
| Type 8 | Echo Request | طلب Ping |
| Type 11 | Time Exceeded | انتهاء TTL (يستخدمه traceroute) |

### ⚠️ اختبارات متقدمة (بيئة مختبرية فقط)

#### ICMP Flood Test

```bash
# ⚠️ للاختبار في بيئة معزولة فقط
sudo ping -f -s 65500 TARGET_IP
```

**الشرح:**
- `-f`: Flood mode (إرسال سريع جداً)
- `-s 65500`: حجم الحزمة (أقصى حد)
- **يُستخدم لاختبار تحمل الشبكة**

**النتائج المتوقعة:**
```
PING target (192.168.1.50) 65500(65528) bytes of data.
..................................
--- target ping statistics ---
10000 packets transmitted, 9856 received, 1.44% packet loss
```

#### Ping of Death (تاريخي)

```bash
# ⚠️ لا يعمل على الأنظمة الحديثة
ping -s 65510 TARGET_IP
```

**الخلفية:**
- كان يُسبب تعطل الأنظمة القديمة
- الأنظمة الحديثة محمية ضده

### 🛡️ الحماية من هجمات ICMP

#### 1. تحديد معدل ICMP

**على Linux:**
```bash
# تحديد عدد حزم ICMP في الثانية
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

#### 2. على Cisco Router

```cisco
# تحديد معدل ICMP
access-list 100 permit icmp any any echo-reply
access-list 100 deny icmp any any echo-request
interface gi0/0
ip access-group 100 in
rate-limit input access-group 100 1000000 250000 500000 conform-action transmit exceed-action drop
```

#### 3. حظر ICMP نهائياً (غير مستحسن)

```bash
# يُفقد القدرة على التشخيص
iptables -A INPUT -p icmp -j DROP
```

### 📝 حالات الاستخدام العملية

**1. تشخيص انقطاع الاتصال:**
```bash
# خطوة بخطوة
ping 192.168.1.1  # البوابة المحلية
ping 8.8.8.8      # خادم خارجي
traceroute 8.8.8.8  # تحديد نقطة الانقطاع
```

**2. قياس جودة الاتصال:**
```bash
# مراقبة لمدة 5 دقائق
ping -c 300 -i 1 8.8.8.8 | tee ping_results.txt

# تحليل النتائج
grep "time=" ping_results.txt | awk '{print $7}' | sed 's/time=//' | sort -n
```

**3. اختبار MTU (Maximum Transmission Unit):**
```bash
# ابدأ بحجم كبير وقلله تدريجياً
ping -M do -s 1472 8.8.8.8

# إذا فشل، قلل الحجم حتى ينجح
ping -M do -s 1400 8.8.8.8
```

### 💡 نصائح مهمة
- ICMP ليس آمناً 100% - يمكن استخدامه في الهجمات
- بعض الشبكات تحظر ICMP للأمان
- استخدم أدوات بديلة (TCP ping) عند الحاجة
- لا تعتمد على ICMP فقط للمراقبة

---

## 9. SYN Flooding Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم SYN Flood وتأثيره على الخوادم.

### ⚠️ تحذير مهم جداً
```
┌─────────────────────────────────────────────────┐
│  ⚠️  هذا المحتوى لأغراض تعليمية فقط  ⚠️       │
│                                                  │
│  تنفيذ هذا الهجوم على أنظمة حقيقية:           │
│  • جريمة يعاقب عليها القانون                   │
│  • يتطلب إذن خطي مسبق                          │
│  • يُسمح فقط في بيئة مختبرية معزولة           │
└─────────────────────────────────────────────────┘
```

### 📋 نظرة عامة على الهجوم

#### المصافحة الثلاثية الطبيعية (TCP Three-Way Handshake)

```
Client                    Server
  |                         |
  |-------- SYN ---------->|  [1]
  |<----- SYN-ACK ---------|  [2]
  |-------- ACK ---------->|  [3]
  |                         |
  |==== Connection Open ====|
```

#### هجوم SYN Flood

```
Attacker                  Server
  |                         |
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |      (thousands)        |
  |                         |
  |   ❌ No ACK Sent ❌    |
  |                         |
  |                    [Server]
  |                   SYN Queue
  |                   ┌─────────┐
  |                   │ SYN #1  │ ⏳
  |                   │ SYN #2  │ ⏳
  |                   │ SYN #3  │ ⏳
  |                   │ SYN #... │ ⏳
  |                   │ FULL!   │ 🔴
  |                   └─────────┘
```

### 🔬 التطبيق العملي (بيئة مختبرية فقط)

#### البيئة المطلوبة
- آلة افتراضية للمهاجم (Kali Linux)
- خادم اختبار معزول
- شبكة محلية منعزلة تماماً

#### الأداة: hping3

**التثبيت:**
```bash
sudo apt update
sudo apt install hping3
```

**الهجوم الأساسي:**
```bash
sudo hping3 -S -p 80 --flood TARGET_IP
```

**الشرح:**
- `-S`: إرسال حزم SYN فقط
- `-p 80`: استهداف المنفذ 80 (HTTP)
- `--flood`: إرسال أسرع ما يمكن (بدون انتظار)
- `TARGET_IP`: عنوان الخادم المستهدف

**هجوم متقدم مع عناوين مزيفة:**
```bash
sudo hping3 -S -p 80 --flood --rand-source TARGET_IP
```

- `--rand-source`: عناوين IP مصدر عشوائية
- يُصعّب تتبع المهاجم
- يتجاوز بعض آليات الحماية البسيطة

### 📊 المراقبة على الخادم المستهدف

#### عرض الاتصالات نصف المفتوحة

```bash
netstat -ant | grep SYN_RECV
```

**مثال على النتائج أثناء الهجوم:**
```
tcp    0    0 192.168.1.50:80    10.0.0.100:45123    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.101:45124    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.102:45125    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.103:45126    SYN_RECV
... (مئات أو آلاف السطور)
```

#### مراقبة استهلاك الموارد

```bash
# عرض قائمة الانتظار
ss -tan | grep SYN-RECV | wc -l

# مراقبة الذاكرة
free -h

# مراقبة CPU
top
```

**مؤشرات الهجوم:**
```
SYN-RECV connections: 5000+  🔴 (طبيعي: < 100)
Memory usage: 95%             🔴 (طبيعي: < 70%)
CPU load: 100%                🔴 (طبيعي: < 50%)
```

### 🛡️ آليات الحماية

#### 1. SYN Cookies

**تفعيل SYN Cookies على Linux:**
```bash
# التحقق من الحالة الحالية
cat /proc/sys/net/ipv4/tcp_syncookies

# التفعيل (1 = enabled)
sudo sysctl -w net.ipv4.tcp_syncookies=1

# جعله دائماً
echo "net.ipv4.tcp_syncookies = 1" | sudo tee -a /etc/sysctl.conf
```

**كيف تعمل:**
```
بدلاً من تخزين الاتصال في الذاكرة:
1. الخادم يحسب قيمة مشفرة (cookie)
2. يُرسلها في SYN-ACK
3. لا يحفظ شيء في الذاكرة
4. عند استلام ACK، يتحقق من الـ cookie
5. إن صح، يُكمل الاتصال

النتيجة: لا استنزاف للذاكرة ✅
```

#### 2. تحديد معدل الاتصالات الجديدة

**باستخدام iptables:**
```bash
# السماح بـ 5 اتصالات جديدة فقط في الثانية من نفس IP
iptables -A INPUT -p tcp --syn -m limit --limit 5/s --limit-burst 10 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP
```

**باستخدام nftables:**
```bash
nft add rule ip filter input tcp flags syn ct state new limit rate 5/second accept
```

#### 3. زيادة حجم قائمة SYN

```bash
# زيادة الحد الأقصى للاتصالات نصف المفتوحة
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# تقليل وقت الانتظار
sudo sysctl -w net.ipv4.tcp_synack_retries=2
```

#### 4. استخدام Firewall متقدم

**Cisco ASA:**
```cisco
! تحديد عدد الاتصالات المتزامنة
access-list OUTSIDE_IN extended permit tcp any host 203.0.113.50 eq www
class-map SYN-ATTACK
 match access-list OUTSIDE_IN
policy-map OUTSIDE_POLICY
 class SYN-ATTACK
  set connection conn-max 100 embryonic-conn-max 50
service-policy OUTSIDE_POLICY interface outside
```

**pfSense/OPNsense:**
- تفعيل "SYN Proxy"
- ضبط "Maximum States" و "Maximum Source Nodes"

#### 5. استخدام CDN/DDoS Protection

خدمات مثل:
- Cloudflare
- AWS Shield
- Akamai

تقوم بـ:
- امتصاص الهجمات الضخمة
- توزيع الحمل
- فلترة الحركة الخبيثة

### 📈 التأثير على الخادم

#### مراحل الهجوم

**المرحلة 1: البداية (0-30 ثانية)**
```
SYN Queue: 20% full
Response Time: 100ms → 500ms
```

**المرحلة 2: التصعيد (30-60 ثانية)**
```
SYN Queue: 80% full
Response Time: 500ms → 3000ms
New connections: بطيئة جداً
```

**المرحلة 3: الانهيار (60+ ثانية)**
```
SYN Queue: 100% full
Response Time: timeout
New connections: مرفوضة بالكامل
Service: غير متاح 🔴
```

### 🔍 كشف الهجوم

#### علامات التحذير

```bash
# ارتفاع مفاجئ في SYN_RECV
watch -n 1 'netstat -ant | grep SYN_RECV | wc -l'

# تنبيه عند تجاوز حد معين
while true; do
  COUNT=$(netstat -ant | grep SYN_RECV | wc -l)
  if [ $COUNT -gt 1000 ]; then
    echo "⚠️ Possible SYN Flood: $COUNT connections"
  fi
  sleep 5
done
```

#### تحليل السجلات

```bash
# عرض أكثر IP مصدرية نشاطاً
netstat -ant | grep SYN_RECV | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -10
```

### 💡 الدروس المستفادة

**للمدافعين:**
1. ✅ فعّل SYN Cookies دائماً
2. ✅ راقب معدلات الاتصال بشكل مستمر
3. ✅ استخدم Firewall مع إمكانيات DDoS protection
4. ✅ احتفظ بخطة استجابة جاهزة

**للمهاجمين الأخلاقيين:**
1. 📝 احصل على إذن خطي قبل الاختبار
2. 🔒 استخدم بيئة معزولة تماماً
3. ⏰ حدد نافذة زمنية محدودة للاختبار
4. 📊 وثّق النتائج بشكل احترافي

---

## 10. UDP Flood Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم UDP Flood والفرق بينه وبين SYN Flood.

### ⚠️ تحذير قانوني
```
┌──────────────────────────────────────────────────┐
│  🚨 هذا المحتوى للتعليم والأبحاث الأمنية فقط  │
│                                                   │
│  القيام بهذا الهجوم دون إذن هو:                │
│  • جريمة إلكترونية                              │
│  • يُعاقب عليها بالسجن والغرامة                │
│  • يُسمح فقط في بيئة مختبرية مملوكة لك         │
└──────────────────────────────────────────────────┘
```

### 📋 الفرق بين TCP و UDP

| الميزة | TCP | UDP |
|--------|-----|-----|
| الاتصال | Connection-oriented | Connectionless |
| المصافحة | نعم (3-way handshake) | لا |
| الموثوقية | مضمون الوصول | غير مضمون |
| السرعة | أبطأ | أسرع |
| الترتيب | مرتب | غير مرتب |

### 🔬 آلية هجوم UDP Flood

```
Attacker                    Target Server
  |                              |
  |------- UDP Packet ---------->| Port 53 (DNS)
  |------- UDP Packet ---------->| Port 161 (SNMP)
  |------- UDP Packet ---------->| Port 12345 (Random)
  |------- UDP Packet ---------->| Port 9999 (Closed)
  |       (thousands/sec)        |
  |                              |
  |                         [Server Process]
  |                         1. تلقي الحزمة
  |                         2. فحص المنفذ
  |                         3. المنفذ مغلق؟
  |                         4. إرسال ICMP Port Unreachable
  |                              |
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |                              |
  |                    [Result: Resource Exhaustion]
  |                    • CPU: 100%
  |                    • Bandwidth: Saturated
  |                    • Service: Unavailable
```

### 🔬 التطبيق العملي (بيئة مختبرية معزولة فقط)

#### البيئة المطلوبة
- شبكة مختبرية معزولة تماماً
- خادم اختبار مخصص
- إذن خطي موثق

#### استخدام hping3

**الهجوم الأساسي:**
```bash
sudo hping3 --udp -p 53 --flood TARGET_IP
```

**الشرح:**
- `--udp`: استخدام بروتوكول UDP
- `-p 53`: استهداف منفذ DNS
- `--flood`: إرسال بأقصى سرعة ممكنة

**هجوم مع منافذ عشوائية:**
```bash
sudo hping3 --udp --rand-dest --flood TARGET_IP
```

- `--rand-dest`: منافذ وجهة عشوائية
- يُضعف الخادم أكثر (يفحص كل منفذ)

**هجوم مع حجم حزمة كبير:**
```bash
sudo hping3 --udp -p 53 --flood -d 65500 TARGET_IP
```

- `-d 65500`: حجم البيانات (أقصى حد لـ UDP)
- يستهلك bandwidth أكثر

### 📊 المراقبة على الخادم المستهدف

#### عرض إحصائيات UDP

```bash
netstat -s | grep UDP
```

**مثال على النتائج أثناء الهجوم:**
```
UDP:
    125847 packets received
    98234 packets to unknown port received
    89123 packet receive errors
    45678 packets sent
    RcvbufErrors: 12345
    SndbufErrors: 0
    InErrors: 89123
```

#### مراقبة الحزم في الوقت الفعلي

```bash
# عرض معدل حزم UDP
watch -n 1 'netstat -su | grep "packets received"'

# مراقبة منافذ محددة
tcpdump -i eth0 udp port 53 -nn
```

**مؤشرات الهجوم:**
```
UDP packets/sec: 50,000+     🔴 (طبيعي: < 1,000)
Port Unreachable: 40,000+    🔴 (طبيعي: < 10)
Bandwidth usage: 90%+        🔴 (طبيعي: < 50%)
CPU usage: 100%              🔴 (طبيعي: < 30%)
```

#### تحديد أكثر المنافذ المستهدفة

```bash
tcpdump -i eth0 -nn udp -c 1000 | awk '{print $5}' | cut -d'.' -f5 | sort | uniq -c | sort -nr | head -10
```

### 🛡️ آليات الحماية

#### 1. Rate Limiting على Linux

```bash
# تحديد معدل حزم UDP من كل IP
iptables -A INPUT -p udp -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p udp -j DROP

# حماية منافذ محددة
iptables -A INPUT -p udp --dport 53 -m limit --limit 50/s -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j DROP
```

#### 2. تعطيل ICMP Port Unreachable

```bash
# منع إرسال رسائل Port Unreachable
iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP

# أو عبر sysctl
sysctl -w net.ipv4.icmp_ignore_bogus_error_responses=1
```

**الفائدة:**
- يوفر موارد CPU
- يمنع استنزاف bandwidth بالردود

#### 3. حماية على مستوى الخادم

```bash
# زيادة حجم buffer للـ UDP
sysctl -w net.core.rmem_max=26214400
sysctl -w net.core.rmem_default=26214400

# تحسين معالجة الحزم
sysctl -w net.core.netdev_max_backlog=5000
```

#### 4. Firewall Rules على Cisco

```cisco
! تحديد معدل UDP
access-list 110 permit udp any host 203.0.113.50 eq domain
access-list 110 deny udp any any

interface GigabitEthernet0/0
 ip access-group 110 in
 rate-limit input access-group 110 10000000 1000000 2000000 conform-action transmit exceed-action drop
```

#### 5. استخدام fail2ban

```bash
# تثبيت fail2ban
sudo apt install fail2ban

# إنشاء قاعدة لـ UDP flood
sudo nano /etc/fail2ban/filter.d/udp-flood.conf
```

**محتوى الملف:**
```ini
[Definition]
failregex = .*kernel.*UDP.*SRC=<HOST>.*
ignoreregex =
```

**التفعيل:**
```ini
# /etc/fail2ban/jail.local
[udp-flood]
enabled = true
filter = udp-flood
logpath = /var/log/syslog
maxretry = 100
findtime = 60
bantime = 3600
```

### 📈 المنافذ الشائعة المستهدفة

| المنفذ | الخدمة | سبب الاستهداف |
|--------|---------|---------------|
| 53 | DNS | حيوي للشبكة، يولد استجابات |
| 123 | NTP | يمكن تضخيمه (Amplification) |
| 161 | SNMP | إدارة، غالباً مفتوح |
| 1900 | SSDP | UPnP، شائع في الأجهزة المنزلية |
: SHA-512
- `$y# تدريبات عملية في أمن الشبكات
## Network Security - Practical Labs

---

## 📑 جدول المحتويات

1. [NAT/PAT - ترجمة عناوين الشبكة](#1-natpat---ترجمة-عناوين-الشبكة)
2. [ACL - قوائم التحكم بالوصول](#2-acl---قوائم-التحكم-بالوصول)
3. [FTP وآلية Established](#3-ftp-وآلية-established)
4. [WAF و Load Balancer](#4-waf-و-load-balancer)
5. [Cisco Firepower - تحليل البرمجيات الخبيثة](#5-cisco-firepower---تحليل-البرمجيات-الخبيثة)
6. [ARP - بروتوكول تحليل العناوين](#6-arp---بروتوكول-تحليل-العناوين)
7. [Nmap - مسح الشبكات](#7-nmap---مسح-الشبكات)
8. [ICMP - اختبارات الشبكة](#8-icmp---اختبارات-الشبكة)
9. [SYN Flooding Attack](#9-syn-flooding-attack)
10. [UDP Flood Attack](#10-udp-flood-attack)
11. [Nmap Port Scanning](#11-nmap-port-scanning)
12. [John the Ripper - كسر كلمات المرور](#12-john-the-ripper---كسر-كلمات-المرور)
13. [DHCP Snooping](#13-dhcp-snooping)

---

## 1. NAT/PAT - ترجمة عناوين الشبكة

### 🎯 الهدف
فهم آلية عمل NAT/PAT في ترجمة العناوين الخاصة إلى عناوين عامة.

### 📋 السيناريو
جهاز داخلي بعنوان `10.0.0.5` يريد تصفح الإنترنت عبر جدار ناري يستخدم NAT.

### 🔄 خطوات العمل

1. **الطلب الصادر:**
   - الجهاز الداخلي `10.0.0.5` يرسل طلب HTTP
   - الجدار الناري يترجم العنوان إلى `203.0.113.2:10500`
   - العنوان `203.0.113.2` هو العنوان العام
   - المنفذ `10500` يُستخدم لتمييز الاتصال

2. **الرد الوارد:**
   - عند وصول الرد من الإنترنت إلى `203.0.113.2:10500`
   - يعيد NAT الترجمة العكسية إلى `10.0.0.5` بالمنفذ الأصلي
   - يتم تسليم البيانات للجهاز الداخلي

### ✅ الفائدة
- مئات الأجهزة يمكنها مشاركة عنوان عام واحد
- لا يوجد تضارب بفضل استخدام أرقام المنافذ (Port Numbers)
- توفير في عناوين IPv4 العامة

### 📝 ملاحظات
- PAT (Port Address Translation) هو نوع من NAT
- يُسمى أيضاً NAT Overload
- الجدول الداخلي يحتفظ بالربط بين العناوين الداخلية والمنافذ

---

## 2. ACL - قوائم التحكم بالوصول

### 🎯 الهدف
تطبيق سياسات أمنية باستخدام قوائم التحكم بالوصول (Access Control Lists).

### 📋 السيناريو
لديك:
- **Web Server** في DMZ (Demilitarized Zone)
- **Database Server** داخل الشبكة الداخلية

> 💡 **DMZ:** منطقة منزوعة السلاح، وهي منطقة بين الشبكة الداخلية والإنترنت لتقليل المخاطر الأمنية.

### 🔧 التكوين العملي

```cisco
access-list 100 permit tcp any 10.10.10.10 eq www
access-list 100 permit tcp any 10.10.10.10 eq 443
access-list 100 permit tcp any 10.10.10.12 eq ftp
access-list 100 permit tcp any 10.10.10.12 eq ftp-data
access-list 100 deny ip any any log

interface gi0/1
ip access-group 100 in
```

### 📊 شرح الأوامر

| الأمر | الوظيفة |
|------|---------|
| `permit tcp any 10.10.10.10 eq www` | السماح بالوصول لخادم الويب عبر HTTP (منفذ 80) |
| `permit tcp any 10.10.10.10 eq 443` | السماح بالوصول عبر HTTPS (منفذ 443) |
| `permit tcp any 10.10.10.12 eq ftp` | السماح بـ FTP Control (منفذ 21) |
| `permit tcp any 10.10.10.12 eq ftp-data` | السماح بـ FTP Data (منفذ 20) |
| `deny ip any any log` | حظر أي حركة أخرى مع التسجيل |
| `ip access-group 100 in` | تطبيق القائمة على الواجهة |

### 🎯 الهدف الأمني
- السماح بالوصول فقط للمنافذ المطلوبة
- منع الوصول المباشر لخادم قاعدة البيانات
- تسجيل المحاولات المشبوهة
- تقليل نطاق الهجوم (Attack Surface)

### ⚠️ ملاحظات مهمة
- الوصول لقاعدة البيانات يجب أن يتم فقط من خلال خادم الويب
- ترتيب القواعد مهم جداً (من الأعلى للأسفل)
- القاعدة الضمنية في النهاية: `deny any any`

---

## 3. FTP وآلية Established

### 🎯 الهدف
فهم الفرق بين FTP Passive و Active Mode وتأثيره على ACL.

### 📋 بنية FTP
بروتوكول FTP يستخدم قناتين منفصلتين:
- **Port 21:** قناة التحكم (Control Channel)
- **Port 20:** قناة البيانات (Data Channel)

### 🔄 FTP Passive Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Random High Port (Data)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يطلب وضع Passive
3. الخادم يرد بمنفذ عشوائي عالي
4. العميل يفتح اتصال البيانات نحو هذا المنفذ
5. كل الردود تحتوي على بت ACK ← **مسموح بها** ✅

### 🔄 FTP Active Mode

```
[Client] -----> [Server]
         Port 21 (Control)
         
[Client] <----- [Server]
         Port 20 (Data initiated by server)
```

**آلية العمل:**
1. العميل يفتح اتصال التحكم نحو المنفذ 21
2. العميل يخبر الخادم بمنفذ معين لاستقبال البيانات
3. **الخادم يبدأ اتصال البيانات** من المنفذ 20
4. أول حزمة من الخادم تحتوي على SYN فقط ← **تُرفض** 🚫

### ⚠️ المشكلة مع Established

```cisco
access-list 110 permit tcp any any established
```

هذه القاعدة تسمح فقط بالحزم التي تحتوي على:
- بت ACK
- أو بت RST

**النتيجة:**
- في Active Mode، الخادم يرسل SYN أولاً
- هذه الحزمة لا تحقق شرط "established"
- الاتصال يفشل 🚫

### ✅ الحل الأمثل
استخدام جدار ناري حقيقي (Stateful Firewall) قادر على:
- تتبع الجلسات النشطة
- فهم البروتوكولات المعقدة مثل FTP
- السماح بالاتصالات الديناميكية تلقائياً

### 📝 الخلاصة
- `established` لا يكفي للبروتوكولات المعقدة
- FTP Passive Mode أكثر توافقاً مع الجدران النارية
- Stateful Firewalls ضرورية للبيئات الإنتاجية

---

## 4. WAF و Load Balancer

### 🎯 الهدف
بناء بنية تحتية آمنة وقابلة للتوسع لموقع ويب.

### 📋 السيناريو
موقع ويب يحتوي على عدة خوادم Web Servers خلف Load Balancer، مع حماية من هجمات XSS.

### 🏗️ البنية المعمارية

```
[Internet]
    |
    ↓
[WAF - Web Application Firewall]
    |
    ↓
[Load Balancer]
    |
    ↓------------------↓------------------↓------------------↓
[Web Server 1]   [Web Server 2]   [Web Server 3]   [Web Server 4]
```

### ⚖️ مهام Load Balancer

**خوارزمية التوزيع:**
- **Round Robin:** توزيع متساوٍ على جميع الخوادم
- دورة منتظمة: Server 1 → Server 2 → Server 3 → Server 4 → تكرار

**إدارة الأعطال:**
1. مراقبة صحة الخوادم (Health Checks)
2. عند فشل أحد الخوادم:
   - إزالته من قائمة التوزيع تلقائياً
   - تحويل طلباته إلى الخوادم السليمة
3. عند عودة الخادم: إعادة إدراجه تلقائياً

### 🛡️ مهام WAF (كـ Reverse Proxy)

**الحماية من XSS:**
```javascript
// مثال على محاولة هجوم XSS
<script>alert('XSS Attack')</script>
```

**آلية الحماية:**
1. فحص كل طلب HTTP قبل وصوله للخوادم
2. كشف الشيفرات الخبيثة:
   - JavaScript مشبوه
   - SQL Injection
   - Command Injection
3. منع الطلبات المشبوهة فوراً ⛔
4. منع تسريب البيانات الحساسة في الاستجابات

### 🔍 سياسات الأمان

| نوع الحماية | الوظيفة |
|-------------|---------|
| XSS Prevention | منع حقن JavaScript |
| SQL Injection | حماية قواعد البيانات |
| CSRF Protection | منع الطلبات المزورة |
| Rate Limiting | الحد من معدل الطلبات |
| IP Blacklisting | حظر IP المشبوهة |

### ✅ المزايا المحققة

**الأداء:**
- ⚡ توزيع الأحمال بشكل متوازن
- 🔄 تحمل الأعطال (Fault Tolerance)
- 📈 قابلية التوسع الأفقي

**الأمان:**
- 🛡️ حماية متقدمة من WAF
- 🔒 فحص جميع الطلبات
- 📝 تسجيل المحاولات المشبوهة
- 🚫 حظر الهجمات في الوقت الحقيقي

### 📊 مثال على سجلات WAF

```
2025-10-26 10:15:32 | BLOCKED | XSS Attempt | IP: 192.168.1.100
2025-10-26 10:16:45 | BLOCKED | SQL Injection | IP: 10.0.0.50
2025-10-26 10:17:12 | ALLOWED | Normal Request | IP: 172.16.0.25
```

---

## 5. Cisco Firepower - تحليل البرمجيات الخبيثة

### 🎯 الهدف
فهم كيفية عمل أنظمة كشف البرمجيات الخبيثة المتقدمة وتتبع انتشارها.

### 📋 السيناريو
ملف باسم `tool.exe` تم اكتشافه ضمن حركة المرور على الشبكة.

### 🔍 مراحل الكشف والتحليل

#### المرحلة 1: الفحص الأولي
1. **اعتراض الملف:** جهاز Cisco Firepower يفحص جميع الملفات المارة
2. **حساب البصمة:** استخراج بصمة SHA-256 للملف
   ```
   SHA-256: a3f7b8c9d2e1f4a6b5c8d7e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
   ```
3. **المقارنة:** مقارنة البصمة مع قاعدة بيانات التهديدات

#### المرحلة 2: التصنيف
```
Status: ⚠️ MALWARE DETECTED
Threat Name: Generic.Trojan.Downloader
Threat Score: 95/100 (High)
Category: Trojan
First Seen: 2025-10-25 14:30:22
```

#### المرحلة 3: تحديد الأجهزة المصابة
الأجهزة التالية تم تحديدها كمصابة:
- 🖥️ `192.168.10.90`
- 🖥️ `192.168.133.50`

### 📈 Trajectory Map (خريطة المسار)

```
Time →
Device 1: --------[*]==========[*]-------
                   ↓            ↓
Device 2: --------[*]----[*]----[*]------
                         ↓      ↓
Device 3: ---------------[*]=====[*]-----
```

**رموز الخريطة:**
- `[*]` نقطة انتقال الملف
- `=` فترة نشاط الملف على الجهاز
- `↓` اتجاه انتقال العدوى (من جهاز لآخر)
- `-` فترة خمول

### 📊 جدول الأحداث (Events Table)

| الوقت | الجهاز المصدر | الجهاز الهدف | الحدث | الحالة |
|------|---------------|--------------|--------|--------|
| 14:30:22 | External | 192.168.10.90 | File Download | Infected |
| 14:35:18 | 192.168.10.90 | 192.168.133.50 | File Transfer | Infected |
| 14:42:33 | 192.168.133.50 | Network Share | Propagation Attempt | Blocked |

### 🎯 استخدامات Trajectory Map

1. **تحديد المصدر الأولي:**
   - أول جهاز تلقى الملف
   - نقطة الدخول إلى الشبكة

2. **تتبع مسار الانتشار:**
   - الأجهزة الوسيطة
   - طرق الانتقال المستخدمة

3. **تقييم نطاق الإصابة:**
   - عدد الأجهزة المصابة
   - الأقسام المتأثرة

4. **التخطيط للاستجابة:**
   - أولوية المعالجة
   - خطة العزل والإزالة

### 🚨 خطة الاستجابة للحادث

**الخطوات الفورية:**
1. ✅ عزل الأجهزة المصابة عن الشبكة
2. ✅ حظر بصمة الملف على جميع الأجهزة
3. ✅ فحص شامل للأجهزة المجاورة
4. ✅ تحليل سجلات النشاط

**الإجراءات التصحيحية:**
1. 🔧 إزالة البرمجية الخبيثة
2. 🔧 تحديث تعريفات مكافح الفيروسات
3. 🔧 تصحيح الثغرات المستغلة
4. 🔧 استعادة النظام من نسخة احتياطية نظيفة

**المتابعة:**
1. 📝 توثيق الحادث
2. 📝 تحديث السياسات الأمنية
3. 📝 تدريب المستخدمين
4. 📝 مراقبة معززة لفترة

### 💡 الدروس المستفادة
- أهمية الفحص العميق لحركة المرور
- ضرورة التسجيل الشامل للأحداث
- قيمة أدوات التحليل البصري
- أهمية الاستجابة السريعة

---

## 6. ARP - بروتوكول تحليل العناوين

### 🎯 الهدف
فهم آلية عمل بروتوكول ARP ومراقبة الجدول الخاص به.

### 📋 نظرة عامة
ARP (Address Resolution Protocol) يربط بين:
- **عنوان IP** (Layer 3)
- **عنوان MAC** (Layer 2)

### 🔍 التطبيق العملي

#### الخطوة 1: عرض جدول ARP

**على Windows:**
```cmd
arp -a
```

**على Linux/macOS:**
```bash
arp -a
```

#### الخطوة 2: قراءة النتائج

```
Interface: 192.168.1.100 --- 0x4
  Internet Address      Physical Address      Type
  192.168.1.1          00-1A-2B-3C-4D-5E     dynamic
  192.168.1.10         00-AA-BB-CC-DD-EE     dynamic
  192.168.1.255        FF-FF-FF-FF-FF-FF     static
  224.0.0.22           01-00-5E-00-00-16     static
```

### 📊 تحليل الجدول

| المكون | الشرح |
|--------|--------|
| **Internet Address** | عنوان IP للجهاز |
| **Physical Address** | عنوان MAC للجهاز |
| **Type** | نوع الإدخال |

### 🏷️ أنواع الإدخالات

**Dynamic:**
- يتم تعلمه تلقائياً من حركة المرور
- يُحذف بعد فترة عدم النشاط (عادة 2-10 دقائق)
- الأكثر شيوعاً في الشبكات

**Static:**
- يتم إضافته يدوياً بواسطة المدير
- لا يُحذف تلقائياً
- يُستخدم للأجهزة الحساسة

### 🔄 عناوين خاصة

**Broadcast:**
```
192.168.1.255 → FF-FF-FF-FF-FF-FF
```
- يُرسل لجميع الأجهزة في الشبكة

**Multicast:**
```
224.0.0.22 → 01-00-5E-00-00-16
```
- يُرسل لمجموعة محددة من الأجهزة

### 🧹 مسح الكاش

**مسح جميع الإدخالات (Windows):**
```cmd
arp -d *
```

**مسح إدخال محدد:**
```cmd
arp -d 192.168.1.10
```

**على Linux:**
```bash
sudo ip -s -s neigh flush all
```

### 📝 إضافة إدخال ثابت

**Windows:**
```cmd
arp -s 192.168.1.50 00-11-22-33-44-55
```

**Linux:**
```bash
sudo arp -s 192.168.1.50 00:11:22:33:44:55
```

### 🔍 حالات الاستخدام

**1. تشخيص مشاكل الاتصال:**
```bash
# فحص وجود الجهاز في الجدول
arp -a | grep 192.168.1.10

# إذا لم يظهر، المشكلة قد تكون في Layer 2
```

**2. كشف تضارب IP:**
```bash
# إذا ظهر نفس IP بعنواني MAC مختلفين
192.168.1.50    00-AA-BB-CC-DD-EE
192.168.1.50    00-11-22-33-44-55  # ⚠️ تضارب!
```

**3. مراقبة الأجهزة النشطة:**
```bash
# عد الأجهزة المتصلة
arp -a | grep dynamic | wc -l
```

### ⚠️ ملاحظات أمنية

**ARP Spoofing/Poisoning:**
- مهاجم يرسل رسائل ARP مزيفة
- يخدع الأجهزة لإرسال البيانات له
- **الحماية:** ARP Inspection على السويتشات

**الوقاية:**
```cisco
# تفعيل Dynamic ARP Inspection
Switch(config)# ip arp inspection vlan 10
Switch(config)# interface gi0/1
Switch(config-if)# ip arp inspection trust
```

### 💡 نصائح عملية
- راقب جدول ARP بشكل دوري
- انتبه للتغييرات المفاجئة في عناوين MAC
- استخدم إدخالات ثابتة للأجهزة الحساسة
- فعّل ARP Inspection في البيئات الحساسة

---

## 7. Nmap - مسح الشبكات

### 🎯 الهدف
استخدام Nmap لاكتشاف الأجهزة والخدمات النشطة على الشبكة.

### 📋 المتطلبات
- نظام Linux (يفضل Kali Linux)
- صلاحيات root للمسح المتقدم
- **بيئة اختبارية فقط** ⚠️

### 🔍 أنواع المسح

#### 1. اكتشاف الأجهزة النشطة (Host Discovery)

```bash
nmap -sn 192.168.1.0/24
```

**الشرح:**
- `-sn`: Ping Scan (بدون فحص المنافذ)
- يستخدم ICMP Echo Request
- يُظهر الأجهزة المتصلة والنشطة

**مثال على النتائج:**
```
Starting Nmap 7.94
Nmap scan report for 192.168.1.1
Host is up (0.0012s latency).

Nmap scan report for 192.168.1.50
Host is up (0.0034s latency).

Nmap scan report for 192.168.1.100
Host is up (0.0028s latency).

Nmap done: 256 IP addresses (3 hosts up) scanned in 2.85 seconds
```

#### 2. فحص المنافذ (Port Scanning)

**Basic TCP Scan:**
```bash
nmap 192.168.1.50
```

**Comprehensive Scan:**
```bash
nmap -sS -sV -O 192.168.1.50
```

**الشرح:**
- `-sS`: SYN Stealth Scan (أقل قابلية للكشف)
- `-sV`: كشف إصدارات الخدمات
- `-O`: تخمين نظام التشغيل

**مثال على النتائج:**
```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 8.2p1
80/tcp   open  http       Apache httpd 2.4.41
443/tcp  open  ssl/http   Apache httpd 2.4.41
3306/tcp open  mysql      MySQL 5.7.33

OS details: Linux 3.2 - 4.9
```

#### 3. فحص جميع المنافذ

```bash
nmap -p- 192.168.1.50
```

- `-p-`: فحص جميع المنافذ (1-65535)
- يستغرق وقتاً أطول

#### 4. فحص منافذ محددة

```bash
nmap -p 21,22,80,443,3306 192.168.1.50
```

### 🎭 أنواع مسح المنافذ

#### TCP SYN Scan (الأكثر شيوعاً)
```bash
nmap -sS 192.168.1.50
```
- يرسل SYN فقط
- لا يُكمل المصافحة الثلاثية
- أقل قابلية للكشف

#### TCP Connect Scan
```bash
nmap -sT 192.168.1.50
```
- يُكمل المصافحة الثلاثية
- لا يحتاج صلاحيات root
- أكثر قابلية للكشف

#### UDP Scan
```bash
nmap -sU 192.168.1.50
```
- فحص منافذ UDP
- بطيء جداً
- مهم لخدمات مثل DNS, DHCP

### 🔬 فحص الثغرات

```bash
nmap --script vuln 192.168.1.50
```

يستخدم NSE (Nmap Scripting Engine) للبحث عن ثغرات معروفة.

### 📊 تقييم المخاطر

#### منافذ عالية الخطورة:

| المنفذ | الخدمة | الخطر |
|--------|---------|-------|
| 21 | FTP | نقل بيانات بدون تشفير |
| 23 | Telnet | إدارة عن بعد بدون تشفير |
| 445 | SMB | عرضة لهجمات WannaCry وغيرها |
| 3389 | RDP | هدف شائع للهجمات |
| 1433 | MSSQL | قاعدة بيانات قد تكون عرضة |

### 🛡️ الحماية من Nmap

**1. على مستوى الجدار الناري:**
```cisco
# رفض المسح من مصادر خارجية
access-list 100 deny tcp any any range 1 1024
```

**2. على مستوى الخادم:**
```bash
# استخدام iptables لمنع المسح
iptables -A INPUT -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT
```

**3. أدوات كشف المسح:**
```bash
# تثبيت portsentry
sudo apt install portsentry

# مراقبة محاولات المسح
tail -f /var/log/syslog | grep portsentry
```

### 💡 نصائح للاستخدام الآمن
- ⚠️ **لا تقم بمسح شبكات لا تملكها**
- 📝 احصل على إذن خطي قبل أي اختبار اختراق
- 🔒 استخدم بيئة مختبرية معزولة للتدريب
- 📊 وثّق جميع عمليات المسح ونتائجها

### 🎓 أمثلة تعليمية إضافية

**مسح سريع لأهم 100 منفذ:**
```bash
nmap --top-ports 100 192.168.1.50
```

**مسح مع تجنب الكشف:**
```bash
nmap -sS -T2 -f 192.168.1.50
```
- `-T2`: سرعة بطيئة (أقل إزعاجاً)
- `-f`: تجزئة الحزم

**حفظ النتائج:**
```bash
nmap -oN results.txt 192.168.1.50
nmap -oX results.xml 192.168.1.50
```

---

## 8. ICMP - اختبارات الشبكة

### 🎯 الهدف
استخدام أدوات ICMP لتشخيص واختبار الاتصال بالشبكة.

### 📋 بروتوكول ICMP
Internet Control Message Protocol - بروتوكول رسائل التحكم في الإنترنت
- Layer 3 Protocol
- يُستخدم للتشخيص وإبلاغ الأخطاء
- لا يحمل بيانات المستخدم

### 🔍 الأوامر الأساسية

#### 1. Ping - اختبار الاتصال

**الاستخدام البسيط:**
```bash
ping 8.8.8.8
```

**مع خيارات متقدمة:**
```bash
# إرسال 10 حزم فقط
ping -c 10 8.8.8.8

# تغيير حجم الحزمة
ping -s 1000 8.8.8.8

# تحديد الوقت بين الحزم (ثانية واحدة)
ping -i 1 8.8.8.8
```

**مثال على النتائج:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=11.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=12.1 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 11.8/12.1/12.4/0.3 ms
```

#### 2. Traceroute - تتبع المسار

**على Linux:**
```bash
traceroute 8.8.8.8
```

**على Windows:**
```cmd
tracert 8.8.8.8
```

**مثال على النتائج:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.245 ms  1.123 ms  1.089 ms
 2  10.0.0.1 (10.0.0.1)  8.456 ms  8.234 ms  8.123 ms
 3  172.16.0.1 (172.16.0.1)  15.678 ms  15.456 ms  15.234 ms
 4  * * *
 5  8.8.8.8 (8.8.8.8)  22.345 ms  22.123 ms  21.987 ms
```

**التحليل:**
- كل سطر يمثل موجّه (router) في المسار
- الأرقام الثلاثة: زمن الاستجابة للمحاولات الثلاث
- `* * *`: الموجّه لا يستجيب لطلبات ICMP

#### 3. MTR - تتبع متقدم

```bash
mtr 8.8.8.8
```

يجمع بين ping و traceroute مع إحصائيات مستمرة:
```
                             Packets               Pings
 Host                      Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.1.1             0.0%    10    1.2   1.3   1.1   1.5   0.1
 2. 10.0.0.1                0.0%    10    8.4   8.5   8.1   9.2   0.3
 3. 8.8.8.8                 0.0%    10   22.1  22.3  21.8  23.1   0.4
```

### 📊 أنواع رسائل ICMP

| النوع | الاسم | الاستخدام |
|-------|-------|-----------|
| Type 0 | Echo Reply | رد على Ping |
| Type 3 | Destination Unreachable | الوجهة غير متاحة |
| Type 5 | Redirect | إعادة توجيه المسار |
| Type 8 | Echo Request | طلب Ping |
| Type 11 | Time Exceeded | انتهاء TTL (يستخدمه traceroute) |

### ⚠️ اختبارات متقدمة (بيئة مختبرية فقط)

#### ICMP Flood Test

```bash
# ⚠️ للاختبار في بيئة معزولة فقط
sudo ping -f -s 65500 TARGET_IP
```

**الشرح:**
- `-f`: Flood mode (إرسال سريع جداً)
- `-s 65500`: حجم الحزمة (أقصى حد)
- **يُستخدم لاختبار تحمل الشبكة**

**النتائج المتوقعة:**
```
PING target (192.168.1.50) 65500(65528) bytes of data.
..................................
--- target ping statistics ---
10000 packets transmitted, 9856 received, 1.44% packet loss
```

#### Ping of Death (تاريخي)

```bash
# ⚠️ لا يعمل على الأنظمة الحديثة
ping -s 65510 TARGET_IP
```

**الخلفية:**
- كان يُسبب تعطل الأنظمة القديمة
- الأنظمة الحديثة محمية ضده

### 🛡️ الحماية من هجمات ICMP

#### 1. تحديد معدل ICMP

**على Linux:**
```bash
# تحديد عدد حزم ICMP في الثانية
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

#### 2. على Cisco Router

```cisco
# تحديد معدل ICMP
access-list 100 permit icmp any any echo-reply
access-list 100 deny icmp any any echo-request
interface gi0/0
ip access-group 100 in
rate-limit input access-group 100 1000000 250000 500000 conform-action transmit exceed-action drop
```

#### 3. حظر ICMP نهائياً (غير مستحسن)

```bash
# يُفقد القدرة على التشخيص
iptables -A INPUT -p icmp -j DROP
```

### 📝 حالات الاستخدام العملية

**1. تشخيص انقطاع الاتصال:**
```bash
# خطوة بخطوة
ping 192.168.1.1  # البوابة المحلية
ping 8.8.8.8      # خادم خارجي
traceroute 8.8.8.8  # تحديد نقطة الانقطاع
```

**2. قياس جودة الاتصال:**
```bash
# مراقبة لمدة 5 دقائق
ping -c 300 -i 1 8.8.8.8 | tee ping_results.txt

# تحليل النتائج
grep "time=" ping_results.txt | awk '{print $7}' | sed 's/time=//' | sort -n
```

**3. اختبار MTU (Maximum Transmission Unit):**
```bash
# ابدأ بحجم كبير وقلله تدريجياً
ping -M do -s 1472 8.8.8.8

# إذا فشل، قلل الحجم حتى ينجح
ping -M do -s 1400 8.8.8.8
```

### 💡 نصائح مهمة
- ICMP ليس آمناً 100% - يمكن استخدامه في الهجمات
- بعض الشبكات تحظر ICMP للأمان
- استخدم أدوات بديلة (TCP ping) عند الحاجة
- لا تعتمد على ICMP فقط للمراقبة

---

## 9. SYN Flooding Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم SYN Flood وتأثيره على الخوادم.

### ⚠️ تحذير مهم جداً
```
┌─────────────────────────────────────────────────┐
│  ⚠️  هذا المحتوى لأغراض تعليمية فقط  ⚠️       │
│                                                  │
│  تنفيذ هذا الهجوم على أنظمة حقيقية:           │
│  • جريمة يعاقب عليها القانون                   │
│  • يتطلب إذن خطي مسبق                          │
│  • يُسمح فقط في بيئة مختبرية معزولة           │
└─────────────────────────────────────────────────┘
```

### 📋 نظرة عامة على الهجوم

#### المصافحة الثلاثية الطبيعية (TCP Three-Way Handshake)

```
Client                    Server
  |                         |
  |-------- SYN ---------->|  [1]
  |<----- SYN-ACK ---------|  [2]
  |-------- ACK ---------->|  [3]
  |                         |
  |==== Connection Open ====|
```

#### هجوم SYN Flood

```
Attacker                  Server
  |                         |
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |-------- SYN ---------->|  [1] ✅
  |      (thousands)        |
  |                         |
  |   ❌ No ACK Sent ❌    |
  |                         |
  |                    [Server]
  |                   SYN Queue
  |                   ┌─────────┐
  |                   │ SYN #1  │ ⏳
  |                   │ SYN #2  │ ⏳
  |                   │ SYN #3  │ ⏳
  |                   │ SYN #... │ ⏳
  |                   │ FULL!   │ 🔴
  |                   └─────────┘
```

### 🔬 التطبيق العملي (بيئة مختبرية فقط)

#### البيئة المطلوبة
- آلة افتراضية للمهاجم (Kali Linux)
- خادم اختبار معزول
- شبكة محلية منعزلة تماماً

#### الأداة: hping3

**التثبيت:**
```bash
sudo apt update
sudo apt install hping3
```

**الهجوم الأساسي:**
```bash
sudo hping3 -S -p 80 --flood TARGET_IP
```

**الشرح:**
- `-S`: إرسال حزم SYN فقط
- `-p 80`: استهداف المنفذ 80 (HTTP)
- `--flood`: إرسال أسرع ما يمكن (بدون انتظار)
- `TARGET_IP`: عنوان الخادم المستهدف

**هجوم متقدم مع عناوين مزيفة:**
```bash
sudo hping3 -S -p 80 --flood --rand-source TARGET_IP
```

- `--rand-source`: عناوين IP مصدر عشوائية
- يُصعّب تتبع المهاجم
- يتجاوز بعض آليات الحماية البسيطة

### 📊 المراقبة على الخادم المستهدف

#### عرض الاتصالات نصف المفتوحة

```bash
netstat -ant | grep SYN_RECV
```

**مثال على النتائج أثناء الهجوم:**
```
tcp    0    0 192.168.1.50:80    10.0.0.100:45123    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.101:45124    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.102:45125    SYN_RECV
tcp    0    0 192.168.1.50:80    10.0.0.103:45126    SYN_RECV
... (مئات أو آلاف السطور)
```

#### مراقبة استهلاك الموارد

```bash
# عرض قائمة الانتظار
ss -tan | grep SYN-RECV | wc -l

# مراقبة الذاكرة
free -h

# مراقبة CPU
top
```

**مؤشرات الهجوم:**
```
SYN-RECV connections: 5000+  🔴 (طبيعي: < 100)
Memory usage: 95%             🔴 (طبيعي: < 70%)
CPU load: 100%                🔴 (طبيعي: < 50%)
```

### 🛡️ آليات الحماية

#### 1. SYN Cookies

**تفعيل SYN Cookies على Linux:**
```bash
# التحقق من الحالة الحالية
cat /proc/sys/net/ipv4/tcp_syncookies

# التفعيل (1 = enabled)
sudo sysctl -w net.ipv4.tcp_syncookies=1

# جعله دائماً
echo "net.ipv4.tcp_syncookies = 1" | sudo tee -a /etc/sysctl.conf
```

**كيف تعمل:**
```
بدلاً من تخزين الاتصال في الذاكرة:
1. الخادم يحسب قيمة مشفرة (cookie)
2. يُرسلها في SYN-ACK
3. لا يحفظ شيء في الذاكرة
4. عند استلام ACK، يتحقق من الـ cookie
5. إن صح، يُكمل الاتصال

النتيجة: لا استنزاف للذاكرة ✅
```

#### 2. تحديد معدل الاتصالات الجديدة

**باستخدام iptables:**
```bash
# السماح بـ 5 اتصالات جديدة فقط في الثانية من نفس IP
iptables -A INPUT -p tcp --syn -m limit --limit 5/s --limit-burst 10 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP
```

**باستخدام nftables:**
```bash
nft add rule ip filter input tcp flags syn ct state new limit rate 5/second accept
```

#### 3. زيادة حجم قائمة SYN

```bash
# زيادة الحد الأقصى للاتصالات نصف المفتوحة
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# تقليل وقت الانتظار
sudo sysctl -w net.ipv4.tcp_synack_retries=2
```

#### 4. استخدام Firewall متقدم

**Cisco ASA:**
```cisco
! تحديد عدد الاتصالات المتزامنة
access-list OUTSIDE_IN extended permit tcp any host 203.0.113.50 eq www
class-map SYN-ATTACK
 match access-list OUTSIDE_IN
policy-map OUTSIDE_POLICY
 class SYN-ATTACK
  set connection conn-max 100 embryonic-conn-max 50
service-policy OUTSIDE_POLICY interface outside
```

**pfSense/OPNsense:**
- تفعيل "SYN Proxy"
- ضبط "Maximum States" و "Maximum Source Nodes"

#### 5. استخدام CDN/DDoS Protection

خدمات مثل:
- Cloudflare
- AWS Shield
- Akamai

تقوم بـ:
- امتصاص الهجمات الضخمة
- توزيع الحمل
- فلترة الحركة الخبيثة

### 📈 التأثير على الخادم

#### مراحل الهجوم

**المرحلة 1: البداية (0-30 ثانية)**
```
SYN Queue: 20% full
Response Time: 100ms → 500ms
```

**المرحلة 2: التصعيد (30-60 ثانية)**
```
SYN Queue: 80% full
Response Time: 500ms → 3000ms
New connections: بطيئة جداً
```

**المرحلة 3: الانهيار (60+ ثانية)**
```
SYN Queue: 100% full
Response Time: timeout
New connections: مرفوضة بالكامل
Service: غير متاح 🔴
```

### 🔍 كشف الهجوم

#### علامات التحذير

```bash
# ارتفاع مفاجئ في SYN_RECV
watch -n 1 'netstat -ant | grep SYN_RECV | wc -l'

# تنبيه عند تجاوز حد معين
while true; do
  COUNT=$(netstat -ant | grep SYN_RECV | wc -l)
  if [ $COUNT -gt 1000 ]; then
    echo "⚠️ Possible SYN Flood: $COUNT connections"
  fi
  sleep 5
done
```

#### تحليل السجلات

```bash
# عرض أكثر IP مصدرية نشاطاً
netstat -ant | grep SYN_RECV | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -10
```

### 💡 الدروس المستفادة

**للمدافعين:**
1. ✅ فعّل SYN Cookies دائماً
2. ✅ راقب معدلات الاتصال بشكل مستمر
3. ✅ استخدم Firewall مع إمكانيات DDoS protection
4. ✅ احتفظ بخطة استجابة جاهزة

**للمهاجمين الأخلاقيين:**
1. 📝 احصل على إذن خطي قبل الاختبار
2. 🔒 استخدم بيئة معزولة تماماً
3. ⏰ حدد نافذة زمنية محدودة للاختبار
4. 📊 وثّق النتائج بشكل احترافي

---

## 10. UDP Flood Attack

### 🎯 الهدف التعليمي
فهم آلية هجوم UDP Flood والفرق بينه وبين SYN Flood.

### ⚠️ تحذير قانوني
```
┌──────────────────────────────────────────────────┐
│  🚨 هذا المحتوى للتعليم والأبحاث الأمنية فقط  │
│                                                   │
│  القيام بهذا الهجوم دون إذن هو:                │
│  • جريمة إلكترونية                              │
│  • يُعاقب عليها بالسجن والغرامة                │
│  • يُسمح فقط في بيئة مختبرية مملوكة لك         │
└──────────────────────────────────────────────────┘
```

### 📋 الفرق بين TCP و UDP

| الميزة | TCP | UDP |
|--------|-----|-----|
| الاتصال | Connection-oriented | Connectionless |
| المصافحة | نعم (3-way handshake) | لا |
| الموثوقية | مضمون الوصول | غير مضمون |
| السرعة | أبطأ | أسرع |
| الترتيب | مرتب | غير مرتب |

### 🔬 آلية هجوم UDP Flood

```
Attacker                    Target Server
  |                              |
  |------- UDP Packet ---------->| Port 53 (DNS)
  |------- UDP Packet ---------->| Port 161 (SNMP)
  |------- UDP Packet ---------->| Port 12345 (Random)
  |------- UDP Packet ---------->| Port 9999 (Closed)
  |       (thousands/sec)        |
  |                              |
  |                         [Server Process]
  |                         1. تلقي الحزمة
  |                         2. فحص المنفذ
  |                         3. المنفذ مغلق؟
  |                         4. إرسال ICMP Port Unreachable
  |                              |
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |<---- ICMP Unreachable -------|
  |                              |
  |                    [Result: Resource Exhaustion]
  |                    • CPU: 100%
  |                    • Bandwidth: Saturated
  |                    • Service: Unavailable
```

### 🔬 التطبيق العملي (بيئة مختبرية معزولة فقط)

#### البيئة المطلوبة
- شبكة مختبرية معزولة تماماً
- خادم اختبار مخصص
- إذن خطي موثق

#### استخدام hping3

**الهجوم الأساسي:**
```bash
sudo hping3 --udp -p 53 --flood TARGET_IP
```

**الشرح:**
- `--udp`: استخدام بروتوكول UDP
- `-p 53`: استهداف منفذ DNS
- `--flood`: إرسال بأقصى سرعة ممكنة

**هجوم مع منافذ عشوائية:**
```bash
sudo hping3 --udp --rand-dest --flood TARGET_IP
```

- `--rand-dest`: منافذ وجهة عشوائية
- يُضعف الخادم أكثر (يفحص كل منفذ)

**هجوم مع حجم حزمة كبير:**
```bash
sudo hping3 --udp -p 53 --flood -d 65500 TARGET_IP
```

- `-d 65500`: حجم البيانات (أقصى حد لـ UDP)
- يستهلك bandwidth أكثر

### 📊 المراقبة على الخادم المستهدف

#### عرض إحصائيات UDP

```bash
netstat -s | grep UDP
```

**مثال على النتائج أثناء الهجوم:**
```
UDP:
    125847 packets received
    98234 packets to unknown port received
    89123 packet receive errors
    45678 packets sent
    RcvbufErrors: 12345
    SndbufErrors: 0
    InErrors: 89123
```

#### مراقبة الحزم في الوقت الفعلي

```bash
# عرض معدل حزم UDP
watch -n 1 'netstat -su | grep "packets received"'

# مراقبة منافذ محددة
tcpdump -i eth0 udp port 53 -nn
```

**مؤشرات الهجوم:**
```
UDP packets/sec: 50,000+     🔴 (طبيعي: < 1,000)
Port Unreachable: 40,000+    🔴 (طبيعي: < 10)
Bandwidth usage: 90%+        🔴 (طبيعي: < 50%)
CPU usage: 100%              🔴 (طبيعي: < 30%)
```

#### تحديد أكثر المنافذ المستهدفة

```bash
tcpdump -i eth0 -nn udp -c 1000 | awk '{print $5}' | cut -d'.' -f5 | sort | uniq -c | sort -nr | head -10
```

### 🛡️ آليات الحماية

#### 1. Rate Limiting على Linux

```bash
# تحديد معدل حزم UDP من كل IP
iptables -A INPUT -p udp -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p udp -j DROP

# حماية منافذ محددة
iptables -A INPUT -p udp --dport 53 -m limit --limit 50/s -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j DROP
```

#### 2. تعطيل ICMP Port Unreachable

```bash
# منع إرسال رسائل Port Unreachable
iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP

# أو عبر sysctl
sysctl -w net.ipv4.icmp_ignore_bogus_error_responses=1
```

**الفائدة:**
- يوفر موارد CPU
- يمنع استنزاف bandwidth بالردود

#### 3. حماية على مستوى الخادم

```bash
# زيادة حجم buffer للـ UDP
sysctl -w net.core.rmem_max=26214400
sysctl -w net.core.rmem_default=26214400

# تحسين معالجة الحزم
sysctl -w net.core.netdev_max_backlog=5000
```

#### 4. Firewall Rules على Cisco

```cisco
! تحديد معدل UDP
access-list 110 permit udp any host 203.0.113.50 eq domain
access-list 110 deny udp any any

interface GigabitEthernet0/0
 ip access-group 110 in
 rate-limit input access-group 110 10000000 1000000 2000000 conform-action transmit exceed-action drop
```

#### 5. استخدام fail2ban

```bash
# تثبيت fail2ban
sudo apt install fail2ban

# إنشاء قاعدة لـ UDP flood
sudo nano /etc/fail2ban/filter.d/udp-flood.conf
```

**محتوى الملف:**
```ini
[Definition]
failregex = .*kernel.*UDP.*SRC=<HOST>.*
ignoreregex =
```

**التفعيل:**
```ini
# /etc/fail2ban/jail.local
[udp-flood]
enabled = true
filter = udp-flood
logpath = /var/log/syslog
maxretry = 100
findtime = 60
bantime = 3600
```

### 📈 المنافذ الشائعة المستهدفة

| المنفذ | الخدمة | سبب الاستهداف |
|--------|---------|---------------|
| 53 | DNS | حيوي للشبكة، يولد استجابات |
| 123 | NTP | يمكن تضخيمه (Amplification) |
| 161 | SNMP | إدارة، غالباً مفتوح |
| 1900 | SSDP | UPnP، شائع في الأجهزة المنزلية |
: yescrypt (الأحدث)

#### الخطوة 2: إنشاء ملف اختبار

```bash
# إنشاء مستخدم تجريبي ضعيف
sudo useradd -m testuser
echo "testuser:password123" | sudo chpasswd

# استخراج الـ hash
sudo grep testuser /etc/shadow > test_hash.txt
```

#### الخطوة 3: كسر كلمة المرور

```bash
john test_hash.txt
```

**النتيجة:**
```
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Proceeding with single crack mode
Press 'q' or Ctrl-C to abort, almost any other key for status
password123      (testuser)
1g 0:00:00:03 DONE (2025-10-26 15:45) 0.3125g/s 800.0p/s 800.0c/s 800.0C/s
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

### 🎯 أوضاع الكسر

#### 1. Single Crack Mode (الوضع الافتراضي)

```bash
john test_hash.txt
```

- يستخدم اسم المستخدم ومعلومات GECOS
- يُجرب تنويعات على الاسم
- سريع جداً

#### 2. Wordlist Mode (قائمة كلمات)

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt test_hash.txt
```

**rockyou.txt:**
- يحتوي على 14 مليون كلمة مرور
- من تسريبات حقيقية
- متوفر في Kali Linux

#### 3. Incremental Mode (Brute Force)

```bash
john --incremental test_hash.txt
```

- يجرب جميع التوافيق الممكنة
- بطيء جداً للكلمات الطويلة
- يبدأ من الأقصر للأطول

#### 4. مع قواعد (Rules)

```bash
john --wordlist=wordlist.txt --rules test_hash.txt
```

**أمثلة على القواعد:**
- `password` → `Password`, `password1`, `p@ssword`
- `admin` → `Admin123`, `admin!`, `4dm1n`

### 📊 عرض النتائج

```bash
# عرض كلمات المرور المكسورة
john --show test_hash.txt

# النتيجة:
# testuser:password123:1001:1001::/home/testuser:/bin/bash
# 1 password hash cracked, 0 left
```

### ⏱️ تقدير الوقت

```bash
john --stdout --incremental | wc -l
```

**مثال على أوقات الكسر:**

| طول الكلمة | نوع الأحرف | الوقت التقريبي |
|------------|-------------|-----------------|
| 4 أحرف | أرقام فقط | ثوانٍ |
| 6 أحرف | أحرف صغيرة | دقائق |
| 8 أحرف | أحرف + أرقام | ساعات |
| 10 أحرف | مختلطة | أيام |
| 12 أحرف | مختلطة + رموز | سنوات |

### 🛡️ إنشاء كلمات مرور قوية

#### معايير كلمة المرور القوية

```
✅ طول: 12+ حرف
✅ أحرف كبيرة: A-Z
✅ أحرف صغيرة: a-z
✅ أرقام: 0-9
✅ رموز: !@#$%^&*
✅ عدم استخدام كلمات قاموسية
✅ فريدة لكل حساب
```

#### أمثلة

| كلمة المرور | القوة | الوقت للكسر |
|-------------|--------|--------------|
| `password` | 🔴 ضعيفة جداً | فوري |
| `password123` | 🔴 ضعيفة | ثوانٍ |
| `Password123` | 🟡 متوسطة | دقائق |
| `P@ssw0rd123!` | 🟡 جيدة | ساعات |
| `Tr0ub4dor&3` | 🟢 قوية | أيام |
| `correct horse battery staple` | 🟢 قوية جداً | سنوات |
| `R7$mK9#pL2@vN4qW` | 🟢 ممتازة | قرون |

### 🔒 آليات الحماية

#### 1. على مستوى النظام

```bash
# زيادة تعقيد SHA-512
sudo nano /etc/pam.d/common-password

# إضافة:
password required pam_unix.so sha512 shadow rounds=100000
```

#### 2. سياسات كلمة المرور

```bash
# تثبيت libpam-pwquality
sudo apt install libpam-pwquality

# التكوين
sudo nano /etc/security/pwquality.conf
```

**محتوى مقترح:**
```ini
minlen = 12
dcredit = -1    # رقم واحد على الأقل
ucredit = -1    # حرف كبير واحد
lcredit = -1    # حرف صغير واحد
ocredit = -1    # رمز واحد
```

#### 3. المصادقة متعددة العوامل (MFA)

```bash
# تثبيت Google Authenticator
sudo apt install libpam-google-authenticator

# تفعيل لمستخدم
google-authenticator
```

#### 4. قفل الحساب بعد محاولات فاشلة

```bash
# إضافة لـ /etc/pam.d/common-auth
auth required pam_tally2.so deny=5 unlock_time=1800
```

### 💡 أفضل الممارسات

**للمستخدمين:**
1. ✅ استخدم مدير كلمات مرور (Password Manager)
2. ✅ فعّل MFA على جميع الحسابات المهمة
3. ✅ لا تعيد استخدام نفس كلمة المرور
4. ✅ غيّر كلمات المرور دورياً (كل 90 يوم للحسابات الحساسة)
5. ✅ استخدم passphrase بدلاً من password

**للمسؤولين:**
1. ✅ فرض سياسات كلمات مرور قوية
2. ✅ استخدم خوارزميات تجزئة حديثة (bcrypt, scrypt, Argon2)
3. ✅ راقب محاولات تسجيل الدخول الفاشلة
4. ✅ فعّل تسجيل دخول المسؤولين بـ MFA إلزامياً
5. ✅ قم بتدقيق كلمات المرور دورياً

### 🎓 تمرين عملي

**الهدف:** اختبار قوة كلمات المرور الخاصة بك

```bash
# 1. إنشاء ملف اختبار
echo '$6$salt$hash' > my_test.txt

# 2. توليد hash لكلمة مرورك التجريبية
python3 -c "import crypt; print(crypt.crypt('your_password', crypt.mksalt(crypt.METHOD_SHA512)))"

# 3. كسرها باستخدام John
john my_test.txt

# 4. تحليل النتائج
# - كم استغرق من الوقت؟
# - هل كُسرت بسهولة؟
# - ما التحسينات الممكنة؟
```

---

## 13. DHCP Snooping

### 🎯 الهدف
حماية الشبكة من هجمات DHCP المزيفة (Rogue DHCP Servers).

### 📋 المشكلة: Rogue DHCP Server

#### السيناريو الطبيعي

```
[Client] ---DHCP Discover---> [Legitimate DHCP Server]
[Client] <--DHCP Offer------- [Legitimate DHCP Server]
[Client] ---DHCP Request----> [Legitimate DHCP Server]
[Client] <--DHCP ACK--------- [Legitimate DHCP Server]

Result: ✅ إعدادات صحيحة
```

#### السيناريو المُخترق

```
[Client] ---DHCP Discover---> [Legitimate Server]
         ---DHCP Discover---> [Rogue Server] 🚨
         
[Client] <--DHCP Offer------- [Legitimate Server]
         <--DHCP Offer------- [Rogue Server] ⚡ (أسرع!)

[Client] ---DHCP Request----> [Rogue Server]
[Client] <--DHCP ACK--------- [Rogue Server]

Result: ❌ الأجهزة تحصل على:
• Gateway مزيف (Man-in-the-Middle)
• DNS مزيف (Pharming)
• عنوان IP من نطاق خاطئ
```

### 🎯 التأثيرات الأمنية

**1. Man-in-the-Middle Attack:**
```
[Client] ---> [Attacker] ---> [Internet]
              (يرى كل شيء)
```

**2. DNS Spoofing:**
```
Client: "ما هو عنوان google.com؟"
Rogue DNS: "عنوانه 10.0.0.66" (موقع مزيف!)
```

**3. Denial of Service:**
```
Rogue DHCP يُعطي عناوين متضاربة
→ الأجهزة لا تستطيع الاتصال
```

### 🛡️ الحل: DHCP Snooping

#### المفهوم الأساسي

**Trusted Ports:**
- منافذ متصلة بخوادم DHCP شرعية
- يُسمح لها بإرسال DHCP
