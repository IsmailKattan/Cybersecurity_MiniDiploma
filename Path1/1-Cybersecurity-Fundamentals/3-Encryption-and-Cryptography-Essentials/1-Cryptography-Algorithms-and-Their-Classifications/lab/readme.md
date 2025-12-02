# ملخص المختبرات العملية - Module 1

## فهرس المختبرات

1. [Lab: Cryptanalysis - Caesar Cipher](#1-lab-cryptanalysis---caesar-cipher)
2. [Lab: RSA Using OpenSSL](#2-lab-rsa-using-openssl)
3. [Lab: AES Using OpenSSL](#3-lab-aes-using-openssl)
4. [Lab: Diffie-Hellman Using OpenSSL](#4-lab-diffie-hellman-using-openssl)

---

## 1. Lab: Cryptanalysis - Caesar Cipher

### الهدف من المختبر
تعلم أساسيات Cryptanalysis من خلال كسر شفرة Caesar cipher البسيطة.

### ما هو Caesar Cipher؟
- واحد من أقدم وأبسط تقنيات التشفير
- Substitution cipher يزيح كل حرف بعدد ثابت من المواضع في الأبجدية
- مثال: بإزاحة 3، A تصبح D، B تصبح E
- يُعتبر الآن **غير آمن** لسهولة كسره بـ Brute Force

### الأدوات المستخدمة
- **cryptii tool** (أداة ويب لتحليل التشفير)

### الخطوات العملية

#### الخطوة 1: فتح أداة cryptii
زيارة موقع cryptii

#### الخطوة 2: اختيار نوع التشفير
- اختيار ENCODE من الصفحة الرئيسية
- اختيار Caesar cipher من قائمة الخيارات

#### الخطوة 3: إدخال النص المشفر
مثال: `"Wklv lv d whvw phvvdjh."`

#### الخطوة 4: فك التشفير
- **طريقة يدوية Manual Decryption**: تجربة قيم Shift من 1 إلى 25
- **Brute Force**: تجربة كل القيم حتى الحصول على نص مقروء
- في المثال: Shift value = 23 ينتج plaintext مقروء

### التمارين العملية

**التمرين 1:**
```
Ciphertext: Uifsf jt b tfdsfu dpef!
Plaintext: There is a secret code!
```

**التمرين 2:**
```
Ciphertext: Byffi Zlcyhxm
Plaintext: Hello Friends
```

**التمرين 3:**
```
Ciphertext: Gojs Hvs Sbjwfcbasbh
Plaintext: Save The Environment
```

### المهارات المكتسبة
- فهم مبدأ عمل Caesar cipher
- تطبيق تقنيات Brute Force
- التعرف على نقاط الضعف في أنظمة التشفير البسيطة
- تطوير مهارات التفكير النقدي والتحليلي

---

## 2. Lab: RSA Using OpenSSL

### الهدف من المختبر
تعلم تشفير وفك تشفير الملفات باستخدام RSA encryption عبر OpenSSL.

### المتطلبات الأساسية
- معرفة بـ Linux command line
- فهم أساسي لمفاهيم التشفير

### الخطوات العملية

#### Step 1: توليد RSA Private Key
```bash
openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048
```
**الشرح:**
- `genpkey`: توليد private key
- `-algorithm RSA`: تحديد خوارزمية RSA
- `-out private_key.pem`: حفظ المفتاح في ملف
- `rsa_keygen_bits:2048`: طول المفتاح 2048 bits (آمن ومستخدم بكثرة)

#### Step 2: استخراج Public Key من Private Key
```bash
openssl rsa -pubout -in private_key.pem -out public_key.pem
```
**الشرح:**
- `-pubout`: استخراج المفتاح العام
- ينتج `public_key.pem` من `private_key.pem`

#### Step 3: إنشاء ملف اختبار
```bash
echo "This is a test file for RSA encryption." > test_file.txt
```

#### Step 4: تشفير الملف باستخدام Public Key
```bash
openssl pkeyutl -encrypt -in test_file.txt -pubin -inkey public_key.pem -out test_file_encrypted.bin
```
**الشرح:**
- `pkeyutl`: أداة لعمليات Public Key
- `-encrypt`: تحديد عملية التشفير
- `-pubin`: المفتاح المدخل هو public key
- الناتج: `test_file_encrypted.bin` (بيانات ثنائية غير قابلة للقراءة)

#### Step 5: فك التشفير باستخدام Private Key
```bash
openssl pkeyutl -decrypt -in test_file_encrypted.bin -inkey private_key.pem -out test_file_decrypted.bin
```
**التحقق:**
```bash
cat test_file_decrypted.bin
```
النتيجة: النص الأصلي يظهر مرة أخرى

### التمارين العملية

#### Exercise 1: تشفير وفك تشفير رسالة قصيرة
**المهمة:**
1. توليد RSA key pair
2. إنشاء ملف برسالة قصيرة
3. تشفيره بـ Public Key
4. فك التشفير بـ Private Key والتحقق

#### Exercise 2: توليد Key Pair وتشفير ملف
**المهمة:**
- توليد key pair بحجم 2048 bits
- تشفير وفك تشفير ملف plaintext
- التحقق من مطابقة الناتج للأصل

#### Exercise 3: محاكاة تأثير طول المفتاح على الأداء
**المهمة:**
1. توليد key pairs بأطوال مختلفة (1024, 2048, 4096 bits)
2. تشفير وفك تشفير نفس الملف
3. قياس ومقارنة الوقت المستغرق لكل عملية

**الأوامر:**
```bash
openssl genpkey -algorithm RSA -out private_key_1024.pem -pkeyopt rsa_keygen_bits:1024
openssl genpkey -algorithm RSA -out private_key_2048.pem -pkeyopt rsa_keygen_bits:2048
openssl genpkey -algorithm RSA -out private_key_4096.pem -pkeyopt rsa_keygen_bits:4096

# قياس الوقت
time openssl pkeyutl -encrypt -in plaintext.txt -pubin -inkey public_key.pem -out encrypted_data.bin
```

### المهارات المكتسبة
- إنشاء RSA key pairs
- تشفير وفك تشفير ملفات باستخدام RSA
- فهم مبادئ Asymmetric Encryption العملية
- تحليل تأثير طول المفتاح على الأمان والأداء

---

## 3. Lab: AES Using OpenSSL

### الهدف من المختبر
تعلم تشفير وفك تشفير الملفات باستخدام AES (Advanced Encryption Standard).

### المتطلبات الأساسية
- معرفة بـ Linux command line
- فهم أساسي لمفاهيم التشفير

### الخطوات العملية

#### Step 1: توليد AES Encryption Key
```bash
openssl rand -base64 32 > aes_key.txt
```
**الشرح:**
- `rand`: توليد بيانات عشوائية
- `-base64`: ترميز الناتج بصيغة Base64
- `32`: طول المفتاح (32 bytes = 256 bits)
- حفظ المفتاح في `aes_key.txt`

**التحقق:**
```bash
cat aes_key.txt
```

#### Step 2: إنشاء ملف للتشفير
```bash
echo "This is a test file for AES encryption." > plaintext.txt
```

**التحقق:**
```bash
cat plaintext.txt
```

#### Step 3: تشفير الملف
```bash
openssl enc -aes-256-cbc -salt -in plaintext.txt -out encrypted_file.bin -pass file:aes_key.txt
```
**الشرح:**
- `enc`: تنفيذ encryption/decryption
- `-aes-256-cbc`: AES بمفتاح 256-bit في وضع CBC
- `-salt`: إضافة salt لتعزيز الأمان
- `-pass file:aes_key.txt`: المفتاح من الملف

**التحقق:**
```bash
cat encrypted_file.bin
```
الناتج: بيانات ثنائية غير قابلة للقراءة

#### Step 4: فك التشفير
```bash
openssl enc -aes-256-cbc -d -in encrypted_file.bin -out decrypted_file.txt -pass file:aes_key.txt
```
**الشرح:**
- `-d`: تحديد عملية فك التشفير

**التحقق:**
```bash
cat decrypted_file.txt
```
النتيجة: `"This is a test file for AES encryption."`

### التمارين العملية

#### Exercise 1: تجربة أطوال مفاتيح مختلفة
**الهدف:** فهم تأثير أطوال المفاتيح المختلفة (128, 192, 256 bits)

**المهام:**
1. تشفير ملف بأطوال مفاتيح مختلفة
2. فك التشفير والتحقق من مطابقة الناتج
3. مقارنة الوقت المستغرق

**الأوامر:**
```bash
# استخدام أطوال مختلفة
-aes-128-cbc
-aes-192-cbc
-aes-256-cbc

# قياس الوقت
time openssl enc -aes-128-cbc -in plaintext.txt -out ciphertext_128.enc -k your_password

# التحقق
sha256sum plaintext.txt decrypted.txt
```

#### Exercise 2: تجربة Initialization Vectors (IVs)
**الهدف:** فهم دور IVs في ضمان ciphertexts فريدة لنفس Plaintexts

**المهام:**
1. تشفير نفس الملف عدة مرات بنفس المفتاح لكن IVs مختلفة
2. مقارنة ciphertexts الناتجة
3. فك التشفير والتحقق من ثبات plaintext

**الأوامر:**
```bash
# توليد IVs عشوائية
openssl rand -hex 16

# استخدام IV محدد
-iv <value>

# مقارنة الملفات
diff ciphertext1.enc ciphertext2.enc
```

### المهارات المكتسبة
- توليد AES encryption keys
- تشفير وفك تشفير ملفات بـ AES
- فهم دور Symmetric Encryption في تأمين البيانات
- العمل مع Initialization Vectors
- تحليل تأثير أطوال المفاتيح المختلفة

---

## 4. Lab: Diffie-Hellman Using OpenSSL

### الهدف من المختبر
تعلم تبادل المفاتيح باستخدام Diffie-Hellman (DH) algorithm عبر OpenSSL.

### المتطلبات الأساسية
- معرفة بـ Linux command line
- فهم أساسي لمفاهيم تبادل المفاتيح التشفيرية

### الخطوات العملية

#### Step 1: توليد DH Parameters
```bash
openssl dhparam -out dhparam.pem 2048
```
**ملاحظة:** العملية تستغرق وقتاً، انتظر حتى تكتمل

**الشرح:**
- `dhparam`: توليد معاملات DH
- `-out dhparam.pem`: ملف الخرج
- `2048`: حجم المعاملات بالـ bits

**التحقق:**
```bash
cat dhparam.pem
```

#### Step 2: توليد Private Key للطرف الأول
```bash
openssl genpkey -paramfile dhparam.pem -out private1.pem
```

#### Step 3: استخراج Public Key للطرف الأول
```bash
openssl pkey -in private1.pem -pubout -out public1.pem
```

#### Step 4: توليد مفاتيح للطرف الثاني
```bash
openssl genpkey -paramfile dhparam.pem -out private2.pem
openssl pkey -in private2.pem -pubout -out public2.pem
```

#### Step 5: اشتقاق Shared Secret Key

**Step 5A: الطرف الأول يشتق المفتاح المشترك**
```bash
openssl pkeyutl -derive -inkey private1.pem -peerkey public2.pem -out shared_secret1.bin
```

**Step 5B: الطرف الثاني يشتق المفتاح المشترك**
```bash
openssl pkeyutl -derive -inkey private2.pem -peerkey public1.pem -out shared_secret2.bin
```

**الشرح:**
- `-derive`: اشتقاق مفتاح مشترك
- `-inkey`: ملف المفتاح الخاص
- `-peerkey`: ملف المفتاح العام للطرف الآخر

#### Step 6: التحقق من Shared Secret
```bash
diff shared_secret1.bin shared_secret2.bin
```
إذا كانت الملفات متطابقة، فعملية DH key exchange نجحت!

### التمارين العملية

#### Exercise 1: توليد والتحقق من Shared Secrets
**المهمة:**
1. توليد DH parameter file
2. محاكاة طرفين (Alice و Bob) بتوليد مفاتيحهما
3. اشتقاق المفتاح المشترك للطرفين والتأكد من تطابقهما

**الأوامر:**
```bash
# توليد المعاملات
openssl dhparam -out dhparam.pem 2048

# Alice
openssl genpkey -paramfile dhparam.pem -out alice_private.pem
openssl pkey -in alice_private.pem -pubout -out alice_public.pem

# Bob
openssl genpkey -paramfile dhparam.pem -out bob_private.pem
openssl pkey -in bob_private.pem -pubout -out bob_public.pem

# اشتقاق المفتاح المشترك
openssl pkeyutl -derive -inkey alice_private.pem -peerkey bob_public.pem -out shared_secret_alice.bin
```

#### Exercise 2: تجربة أحجام معاملات DH مختلفة
**الهدف:** استكشاف تأثير أحجام المعاملات على الأمان والأداء

**المهام:**
1. توليد DH parameters بأحجام 1024, 2048, 4096 bits
2. تنفيذ key exchange لكل حجم
3. قياس ومقارنة الوقت المستغرق

**الأوامر:**
```bash
# توليد معاملات بأحجام مختلفة
openssl dhparam -out dh1024.pem 1024
openssl dhparam -out dh2048.pem 2048
openssl dhparam -out dh4096.pem 4096

# قياس الوقت
time openssl genpkey -paramfile dh2048.pem -out private_key.pem

# التحقق
sha256sum <files>
```

#### Exercise 3: محاكاة هجوم Man-in-the-Middle (MITM)
**الهدف:** فهم ثغرات DH key exchange عند عدم المصادقة

**السيناريو:**
- Alice, Bob, Eve (المهاجم)
- Eve تعترض وتتوسط في الاتصال

**الخطوات:**

1. **توليد DH parameters**
```bash
openssl dhparam -out dhparam.pem 2048
```

2. **توليد مفاتيح للأطراف الثلاثة**
```bash
# Alice
openssl genpkey -paramfile dhparam.pem -out alice_private.pem
openssl pkey -in alice_private.pem -pubout -out alice_public.pem

# Bob
openssl genpkey -paramfile dhparam.pem -out bob_private.pem
openssl pkey -in bob_private.pem -pubout -out bob_public.pem

# Eve (المهاجم)
openssl genpkey -paramfile dhparam.pem -out eve_private.pem
openssl pkey -in eve_private.pem -pubout -out eve_public.pem
```

3. **اعتراض واشتقاق shared secrets**
```bash
# Eve تعترض Alice
openssl pkeyutl -derive -inkey eve_private.pem -peerkey alice_public.pem -out shared_secret_eve_alice.bin

# Eve تعترض Bob
openssl pkeyutl -derive -inkey eve_private.pem -peerkey bob_public.pem -out shared_secret_eve_bob.bin
```

4. **التحقق من shared secrets**
```bash
openssl dgst -sha256 shared_secret_eve_alice.bin
openssl dgst -sha256 shared_secret_eve_bob.bin
```

**الملاحظة:**
- المفتاح المشترك بين Eve و Alice يختلف عن المفتاح بين Eve و Bob
- Alice و Bob **ليس لديهما** مفتاح مشترك حقيقي
- Eve يمكنها فك تشفير رسائل Alice وإعادة تشفيرها لـ Bob والعكس
- يوضح أهمية **المصادقة Authentication** في بروتوكولات تبادل المفاتيح

### المهارات المكتسبة
- توليد Diffie-Hellman parameters
- إنشاء وتبادل public keys
- اشتقاق shared secret keys
- فهم كيفية تبادل المفاتيح الآمن عبر قنوات غير آمنة
- إدراك ثغرات DH عند غياب المصادقة
- تطبيق مبادئ أساسية في بروتوكولات الاتصال الآمن

---

## الخلاصة العامة للمختبرات

### المهارات التقنية المكتسبة:
1. **استخدام OpenSSL** لعمليات التشفير المختلفة
2. **Linux Command Line** للتعامل مع الملفات والمفاتيح
3. **Cryptanalysis** باستخدام أدوات الويب
4. **قياس الأداء** باستخدام `time` command

### المفاهيم العملية المطبقة:
- **Symmetric Encryption** (AES)
- **Asymmetric Encryption** (RSA)
- **Key Exchange Protocols** (Diffie-Hellman)
- **Cryptanalysis Techniques** (Brute Force)

### السيناريوهات الأمنية المحاكاة:
- تشفير وفك تشفير البيانات
- تبادل المفاتيح الآمن
- هجمات MITM
- تحليل التشفير الكلاسيكي

### الأدوات والتقنيات:
- **OpenSSL**: الأداة الرئيسية للتشفير
- **cryptii**: لتحليل التشفير الكلاسيكي
- **Linux utilities**: cat, diff, echo, sha256sum, time

هذه المختبرات توفر أساساً عملياً قوياً لفهم وتطبيق مفاهيم التشفير في سيناريوهات واقعية.