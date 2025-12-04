# ملخص المختبرات العملية - Module 2

## فهرس المختبرات

1. [Lab: Public Key Infrastructure (PKI) Using OpenSSL](#1-lab-public-key-infrastructure-pki-using-openssl)
2. [Lab: Steganography](#2-lab-steganography)

---

## 1. Lab: Public Key Infrastructure (PKI) Using OpenSSL

### الهدف من المختبر
التطبيق العملي لـ Public Key Infrastructure (PKI) باستخدام OpenSSL، من خلال إنشاء Certificate Authority وإدارة الشهادات الرقمية.

### المتطلبات الأساسية
- معرفة بـ Linux command line
- فهم أساسي لمفاهيم التشفير: Public/Private Keys، Digital Certificates

### المدة المقدرة: 60 دقيقة

---

### الخطوات العملية

#### Step 1: إنشاء Certificate Authority (CA)

**Step A: توليد Private Key للـ CA**
```bash
openssl genpkey -algorithm RSA -out ca.key -aes256 -pass pass:yourpassword
```

**الشرح:**
- `genpkey -algorithm RSA`: توليد RSA private key
- `-out ca.key`: حفظ المفتاح في `ca.key`
- `-aes256`: تشفير المفتاح بـ AES-256
- `-pass pass:yourpassword`: كلمة المرور للتشفير

**Step B: توليد شهادة CA**
```bash
openssl req -new -x509 -key ca.key -sha256 -days 365 -out ca.crt -passin pass:yourpassword
```

**الشرح:**
- `req -new -x509`: إنشاء شهادة جديدة وتوقيعها ذاتياً Self-Signed
- `-key ca.key`: تحديد CA private key
- `-sha256`: استخدام خوارزمية hash SHA-256
- `-days 365`: صلاحية الشهادة 365 يوم
- `-out ca.crt`: حفظ الشهادة في `ca.crt`

**ملاحظة:** يجب ملء تفاصيل الشهادة عند المطالبة:
- Country Name
- Common Name
- Organization، إلخ.

#### Step 2: توليد Private Key للخادم (Server)
```bash
openssl genpkey -algorithm RSA -out server.key
```

**النتيجة:** توليد `server.key` - المفتاح الخاص بالخادم

#### Step 3: إنشاء Certificate Signing Request (CSR) للخادم
```bash
openssl req -new -key server.key -out server.csr
```

**الشرح:**
- `-key server.key`: تحديد مفتاح الخادم الخاص
- `-out server.csr`: حفظ CSR في `server.csr`

**ملاحظة:** يجب توفير:
- **Common Name**: عادةً اسم المضيف Hostname للخادم
- تفاصيل أخرى حسب الحاجة

#### Step 4: توقيع شهادة الخادم بواسطة CA
```bash
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256 -passin pass:yourpassword
```

**الشرح:**
- `x509 -req`: معالجة CSR للتوقيع
- `-in server.csr`: ملف CSR للخادم
- `-CA ca.crt`: شهادة CA
- `-CAkey ca.key`: مفتاح CA الخاص
- `-CAcreateserial`: إنشاء ملف رقم تسلسلي Serial Number
- `-out server.crt`: حفظ الشهادة الموقعة في `server.crt`

#### Step 5: التحقق من شهادة الخادم
```bash
openssl verify -CAfile ca.crt server.crt
```

**الشرح:**
- `verify`: التحقق من الشهادة
- `-CAfile ca.crt`: شهادة CA للتحقق ضدها
- `server.crt`: شهادة الخادم المراد التحقق منها

**النتيجة المتوقعة:** 
```
server.crt: OK
```

#### Step 6: محاكاة Client-Server Authentication

**Step A: بدء الخادم Server**
```bash
openssl s_server -cert server.crt -key server.key -accept 8443
```
- يبدأ خادماً بسيطاً يستخدم شهادة ومفتاح الخادم
- يستمع على المنفذ Port 8443

**Step B: الاتصال كعميل Client**
من Terminal آخر:
```bash
openssl s_client -connect localhost:8443 -CAfile ca.crt
```
- الاتصال بالخادم كعميل
- استخدام شهادة CA للتحقق

**الفحص:** فحص تفاصيل الشهادة المعروضة في Terminal العميل

---

### التمارين العملية

#### Exercise 1: إعداد Certificate Authority أساسية

**الهدف:** تعلم إنشاء CA بسيطة - أساس أي بنية PKI.

**المهام:**

**1. إنشاء هيكل الدلائل للـ CA:**
```bash
mkdir -p ~/myCA/newcerts ~/myCA/private
touch ~/myCA/myCAindex
chmod 700 ~/myCA/private
chmod 755 ~/myCA/newcerts
```

**الشرح:**
- `~/myCA/newcerts`: لتخزين الشهادات المُصدرة
- `~/myCA/private`: لتخزين المفاتيح الخاصة (أذونات 700 للأمان)
- `~/myCA/myCAindex`: قاعدة بيانات للشهادات المُصدرة

**2. توليد Root Private Key للـ CA:**
```bash
openssl genpkey -algorithm RSA -out ~/myCA/private/myCAkey.pem -pkeyopt rsa_keygen_bits:2048
```
- توليد مفتاح RSA بطول 2048 bits

**3. توليد Root Certificate للـ CA:**
```bash
openssl req -new -x509 -key ~/myCA/private/myCAkey.pem -out ~/myCA/myCAcert.pem -days 3650
```
- شهادة ذاتية التوقيع Self-Signed
- صلاحية 3650 يوم (10 سنوات)

**نصيحة مهمة:**
- احتفظ بـ Root Certificate (`myCAcert.pem`) بشكل آمن
- هو أساس سلسلة الثقة Trust Chain
- يُستخدم لتوقيع جميع الشهادات الأخرى في PKI

#### Exercise 2: إصدار شهادة باستخدام CA الخاصة بك

**الهدف:** محاكاة عملية إصدار شهادة باستخدام CA أنشأتها في Exercise 1.

**المهام:**

**1. إنشاء Private Key لـ user1:**
```bash
openssl genpkey -algorithm RSA -out user1_private_key.pem -pkeyopt rsa_keygen_bits:2048
```

**2. إنشاء CSR لـ user1:**
```bash
openssl req -new -key user1_private_key.pem -out user1_csr.pem
```
- إنشاء Certificate Signing Request
- يتضمن معلومات user1

**3. استخدام Root CA لتوقيع CSR وإصدار الشهادة:**
```bash
openssl ca -in user1_csr.pem -out user1_cert.pem -cert ~/myCA/myCAcert.pem -keyfile ~/myCA/private/myCAkey.pem -days 365
```

**الشرح:**
- `ca`: استخدام وظيفة CA
- `-in user1_csr.pem`: CSR المراد توقيعه
- `-out user1_cert.pem`: الشهادة الناتجة
- `-cert`: شهادة CA
- `-keyfile`: مفتاح CA الخاص
- `-days 365`: صلاحية سنة واحدة

**التحقق من المحتويات:**
```bash
# فحص CSR
openssl req -text -noout -verify -in user1_csr.pem

# فحص الشهادة
openssl x509 -text -noout -in user1_cert.pem
```

#### حل مشكلة './demoCA/newcerts is not a directory'

إذا واجهت هذا الخطأ، اتبع الخطوات التالية:

**1. إنشاء هيكل الدلائل المطلوب:**
```bash
mkdir -p demoCA/newcerts
mkdir -p demoCA/private
touch demoCA/index.txt
echo 1000 > demoCA/serial
```

**الشرح:**
- `demoCA/newcerts`: لتخزين الشهادات المُصدرة
- `demoCA/private`: لتخزين المفاتيح الخاصة
- `demoCA/index.txt`: ملف قاعدة بيانات للشهادات
- `demoCA/serial`: الرقم التسلسلي الأولي (1000)

**2. تعديل ملف إعدادات OpenSSL (اختياري):**

إذا كنت تفضل استخدام دليل مخصص (`~/myCA`):
- حدد موقع ملف الإعدادات: `/usr/lib/ssl/openssl.cnf` أو `/etc/ssl/openssl.cnf`
- عدّل التوجيه `dir`:

```
[ CA_default ]
dir = /path/to/your/CA
```

**أو حدد الإعدادات صراحة في الأمر:**
```bash
openssl ca -in user1_csr.pem -out user1_cert.pem -cert ~/myCA/myCAcert.pem -keyfile ~/myCA/private/myCAkey.pem -days 365 -config /usr/lib/ssl/openssl.cnf
```

**3. إعادة تنفيذ الأمر:**
```bash
openssl ca -in user1_csr.pem -out user1_cert.pem -cert ~/myCA/myCAcert.pem -keyfile ~/myCA/private/myCAkey.pem -days 365
```

---

### المهارات المكتسبة

1. **إنشاء وإدارة Certificate Authority:**
   - توليد Root CA
   - إدارة هيكل PKI
   - توقيع الشهادات

2. **إدارة دورة حياة الشهادات:**
   - إنشاء Private Keys
   - توليد CSRs
   - توقيع الشهادات
   - التحقق من الشهادات

3. **تطبيق عملي لـ PKI:**
   - محاكاة Client-Server Authentication
   - فهم سلسلة الثقة Trust Chain
   - استخدام الشهادات في الاتصالات الآمنة

4. **حل المشاكل:**
   - إعداد بيئة PKI صحيحة
   - تكوين OpenSSL
   - فهم هيكل الملفات المطلوب

### الأهمية العملية

هذا المختبر يوفر أساساً لفهم:
- كيفية عمل HTTPS
- آلية SSL/TLS
- أمان VPNs
- الاتصالات المشفرة في الأنظمة الحقيقية

---

## 2. Lab: Steganography

### الهدف من المختبر
تعلم Steganography - إخفاء المعلومات داخل ملفات أخرى - باستخدام أدوات Linux.

### المتطلبات الأساسية
- نظام Linux مع `steghide` مثبت
- فهم أساسي لـ Steganography

### المدة المقدرة: 45 دقيقة

---

### إعداد المختبر

#### تثبيت Steghide
```bash
sudo apt update
sudo apt install steghide
```

#### تحضير الملفات

**1. رفع ملف صورة:**
- إعادة تسمية أي صورة إلى `sample.jpg`
- رفعها عبر File Explorer في بيئة المختبر

**الخطوات:**
1. اختر File Explorer
2. افتح تبويب Project
3. انقر بزر الماوس الأيمن → Upload File
4. رفع `sample.jpg`

**2. إنشاء ملف نصي سري:**
```bash
echo "This is a secret file" > secret.txt
```

---

### الخطوات العملية

#### Step 1: التحقق من توافق الملف File Compatibility

```bash
steghide info sample.jpg
```

**الشرح:**
- `steghide info`: عرض معلومات عن الملف الناقل Carrier File
- `sample.jpg`: الملف المراد فحصه

**النتيجة المتوقعة:**
- معلومات عن ملاءمة الملف لتضمين البيانات
- السعة المتاحة Available Capacity

#### Step 2: إخفاء رسالة Hide a Message

```bash
steghide embed -cf sample.jpg -ef secret.txt
```

**الشرح:**
- `embed`: وضع Embedding
- `-cf sample.jpg`: الملف الناقل Carrier File
- `-ef secret.txt`: الملف المحتوي على الرسالة السرية

**المطالبة Prompt:**
- سيُطلب منك إدخال كلمة مرور Passphrase
- اختر كلمة مرور قوية

**ما يحدث:**
- يتم إدخال محتوى `secret.txt` داخل `sample.jpg`
- الصورة تبدو طبيعية تماماً
- لا يمكن كشف البيانات المخفية بالعين المجردة

#### Step 3: استخراج الرسالة المخفية Extract Hidden Message

```bash
steghide extract -sf sample.jpg
```

**الشرح:**
- `extract`: استخراج الملف المخفي
- `-sf sample.jpg`: الملف الناقل Steganographic File

**المطالبة:**
- سيُطلب منك كلمة المرور المستخدمة أثناء الإخفاء

**النتيجة المتوقعة:**
- ظهور الملف المستخرج (`secret.txt`) في الدليل الحالي
- يمكن قراءة المحتوى:
```bash
cat secret.txt
```

#### Step 4: التجربة مع ملفات الصوت Audio Files

**Step A: استخدام ملف صوتي كناقل**

```bash
# إخفاء البيانات
steghide embed -cf sample.wav -ef secret.txt

# استخراج البيانات
steghide extract -sf sample.wav
```

**الملاحظة:**
- نفس الأوامر تعمل مع الملفات الصوتية
- الصوت يبقى قابلاً للتشغيل بشكل طبيعي

**Step B: تشغيل الملف الصوتي**

```bash
aplay sample.wav
```

**التحقق:**
- الملف الصوتي يعمل بشكل طبيعي
- لا يوجد تشويه ملحوظ
- البيانات المخفية غير قابلة للكشف سمعياً

---

### التمارين العملية

#### Exercise 1: إخفاء ملفات أكبر

**الهدف:** تجربة إدراج ملفات أكبر (PDF، ZIP) داخل صورة أو ملف صوتي.

**الأمر:**
```bash
steghide embed -cf image.jpg -ef secret.zip -p mypassword
```

**الشرح:**
- `-p mypassword`: تحديد كلمة المرور مباشرة

**النتيجة المتوقعة:**
```
Enter passphrase: ******
Re-Enter passphrase: ******
embedding "secret.zip" in "image.jpg"... done
```

**الملاحظة:**
- حجم الملف File Size يزيد قليلاً بعد الإدراج
- الزيادة تعتمد على حجم البيانات المخفية

#### Exercise 2: مقارنة أحجام الملفات

**الهدف:** ملاحظة كيف يؤثر إدراج البيانات على حجم الملف الناقل.

**التحقق من حجم الملف:**
```bash
ls -lh image.jpg
```

**النتيجة المثالية:**
```
-rw-r--r-- 1 user user 1.2M Nov 27 12:34 image.jpg
```

**التحليل:**
- قارن الحجم قبل وبعد الإدراج
- لاحظ الزيادة الطفيفة
- الزيادة تتناسب مع حجم البيانات المخفية

#### Exercise 3: التجربة مع أدوات أخرى

**الهدف:** استكشاف أدوات Steganography بديلة.

**1. Outguess:**
```bash
sudo apt install outguess
```

**الاستخدام:**
```bash
# إخفاء
outguess -d secret.txt sample.jpg output.jpg

# استخراج
outguess -r sample.jpg secret_extracted.txt
```

**2. Stegsnow (للملفات النصية):**
```bash
sudo apt install stegsnow
```

**الاستخدام:**
```bash
# إخفاء رسالة في ملف نص
stegsnow -C -m "Hidden message" -p password input.txt output.txt

# استخراج الرسالة
stegsnow -C -p password output.txt
```

**المقارنة بين الأدوات:**

| الأداة | نوع الملفات | الميزات | سهولة الاستخدام |
|--------|-------------|---------|------------------|
| **steghide** | صور، صوت | تشفير، ضغط | عالية |
| **outguess** | صور (JPG) | خوارزميات متقدمة | متوسطة |
| **stegsnow** | ملفات نصية | إخفاء في مسافات بيضاء | عالية |

---

### المهارات المكتسبة

1. **تقنيات Steganography الأساسية:**
   - إخفاء البيانات داخل ملفات وسائط متعددة
   - استخراج البيانات المخفية
   - حماية البيانات بكلمات مرور

2. **العمل مع أنواع ملفات مختلفة:**
   - الصور (JPG، PNG)
   - الملفات الصوتية (WAV)
   - أنواع بيانات متنوعة (نصوص، ملفات مضغوطة)

3. **التحليل والمقارنة:**
   - تأثير الإخفاء على حجم الملف
   - مقارنة أدوات مختلفة
   - تقييم فعالية الإخفاء

4. **الأمان والخصوصية:**
   - استخدام كلمات مرور لحماية البيانات المخفية
   - فهم حدود Steganography
   - إدراك استخدامات الأمان والاتصالات السرية

### الاستخدامات العملية

**استخدامات مشروعة Legitimate Uses:**
- حماية الملكية الفكرية Digital Watermarking
- الاتصالات الآمنة في بيئات محدودة
- حماية البيانات الحساسة أثناء النقل
- الطب الشرعي الرقمي Digital Forensics

**اعتبارات أمنية:**
- Steganography ليس بديلاً عن التشفير
- يمكن كشفه بأدوات Steganalysis
- يجب استخدامه مع طبقات أمان أخرى
- الاستخدام الأخلاقي والقانوني مهم

### الحدود والقيود

1. **حجم البيانات:**
   - محدود بحجم الملف الناقل
   - البيانات الكبيرة قد تؤثر على جودة الملف

2. **قابلية الكشف:**
   - أدوات Steganalysis يمكنها الكشف عن الإخفاء
   - التحليل الإحصائي Statistical Analysis قد يكشف الأنماط

3. **جودة الملف:**
   - قد تتأثر جودة الصورة أو الصوت قليلاً
   - الضغط المتكرر قد يفقد البيانات المخفية

4. **الأمان:**
   - ليس آمناً تماماً بدون تشفير إضافي
- كلمة مرور ضعيفة = حماية ضعيفة

---

## الخلاصة العامة للمختبرات

### المهارات التقنية المكتسبة:

**من PKI Lab:**
- إنشاء وإدارة Certificate Authority
- توليد وتوقيع الشهادات الرقمية
- التحقق من سلاسل الثقة Certificate Chains
- محاكاة اتصالات آمنة Client-Server

**من Steganography Lab:**
- إخفاء واستخراج البيانات في الوسائط
- استخدام أدوات Steganography المتعددة
- تحليل تأثير الإخفاء على الملفات
- فهم حدود وإمكانيات الإخفاء

### المفاهيم العملية المطبقة:

1. **PKI في العالم الحقيقي:**
   - أساس HTTPS والاتصالات الآمنة
   - كيفية عمل SSL/TLS
   - Trust Models في الأنظمة الموزعة
   - إدارة الشهادات في المؤسسات

2. **Steganography في الأمان:**
   - طبقة إضافية للخصوصية
   - حماية البيانات في بيئات معادية
   - Digital Watermarking
   - الاتصالات السرية Covert Communications

### الأدوات والتقنيات:

**OpenSSL:**
- أداة قوية لإدارة PKI
- واسعة الاستخدام في الصناعة
- دعم شامل للمعايير التشفيرية

**Steghide وأدوات Steganography:**
- سهلة الاستخدام
- فعالة للإخفاء الأساسي
- متنوعة في التطبيقات

### السيناريوهات الأمنية:

- إنشاء بنية أمنية كاملة (PKI)
- إدارة الثقة والمصادقة
- حماية البيانات بطرق غير تقليدية
- التوازن بين الأمان وسهولة الاستخدام

### الأهمية في الحياة العملية:

هذه المختبرات توفر خبرة عملية في:
- تقنيات تُستخدم يومياً في تأمين الإنترنت
- مهارات مطلوبة لمديري الأنظمة System Administrators
- فهم عميق لآليات الأمان الحديثة
- القدرة على تطبيق وحل مشاكل PKI الحقيقية

المعرفة المكتسبة من هذه المختبرات أساسية لأي محترف في مجال الأمن السيبراني Cybersecurity Professional.