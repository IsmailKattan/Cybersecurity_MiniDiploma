<div dir="rtl">

# Network Architecture بنية الشبكة

قبل تأمين أي شبكة، عليك فهم تصميمها الأساسي وكيفية عملها. ستتعرف على بنية الشبكة، وأدوات الشبكات القياسية، والشبكات السحابية، والإطار الأساسي لتنظيم الاتصالات عبر  الشبكة والذي يسمى نموذج TCP/IP. 

يُعد تأمين الشبكات جزءًا أساسيًا من مسؤوليات محلل الأمن، لذا يسعدني مساعدتك في فهم كيفية تأمين شبكة مؤسستك من التهديدات والمخاطر والثغرات الأمنية. هيا بنا!

***

## جدول المحتويات 

## [Network Architecture بنية الشبكة](#network-architecture-بنية-الشبكة)
  - ### [Introduction to Networks مدخل إلى الشبكات](#introduction-to-networks-مدخل-إلى-الشبكات)
    - [What are networks? — ما هي الشبكات؟](#1-what-are-networks--ما-هي-الشبكات)
    - [Network tools — أدوات الشبكات](#2-network-tools--أدوات-الشبكات)
    - [Network components, devices, and diagrams](#3-network-components-devices-and-diagrams)
    - [Cloud networks — شبكات السحابة](#4-cloud-networks--شبكات-السحابة)
    - [Cloud computing and Software-defined networks](#5-cloud-computing-and-software-defined-networks)
  
  - ### [Network Communication تواصل الشبكة](#network-communication)
    - [The TCP/IP Model](#1-the-tcpip-model)
    - [Four Layers of the TCP/IP Model](#2-four-layers-of-the-tcpip-model)
    - [Learn More About the TCP/IP Model](#3-learn-more-about-the-tcpip-model)
    - [The OSI Model](#4-the-osi-model)
  
  - ### [Local and Wide Network Communication تواصل الشبكة المحلية والواسعة](#local-and-wide-network-communication)
    - [IP addresses and network communication](#1-ip-addresses-and-network-communication)
    - [Components of network layer communication](#components-of-network-layer-communication)
    - [Difference between IPv4 and IPv6](#3-difference-between-ipv4-and-ipv6)
  
    

***

## Introduction to Networks مدخل إلى الشبكات 

### 1. What are networks? — ما هي الشبكات؟


هذا الدرس يعرض مفاهيم شبكات أساسية، وكلها مهمة للسيبراني حتى لو كانت بسيطة.

الشبكة "Network" هي مجموعة أجهزة متصلة ببعضها لنقل البيانات.
الأجهزة تتواصل باستخدام عناوين **IP address** و **MAC address** لضمان وصول البيانات للوجهة الصحيحة.

هناك نوعان رئيسيان:

* **LAN الشبكة المحلية**: تغطي مساحة صغيرة (منزل – مدرسة – مكتب).
* **WAN الشبكة الواسعة**: تغطي مناطق كبيرة (مدينة – بلد).
  الإنترنت يُعتبر WAN ضخمة.

الفكرة الأساسية التي تهم الأمن:
كل جهاز متصل يجب أن يُعرّف بعنوان، وكل اتصال بين جهازين يمر عبر بنية يمكن استهدافها أو استغلالها.

---

### 2. Network tools — أدوات الشبكات

هذا الدرس مهم لأنه يحدد الأجهزة الأساسية التي تبنى عليها أي بنية شبكات يمكن تأمينها أو اختراقها.

![network devices](./images/network-devices.png)

**Hub**
جهاز بدائي يُرسل البيانات لجميع المنافذ. غير آمن لأنه يسمح بالتنصت بسهولة (Packet sniffing). استخدامه اليوم نادر.

**Switch**
أذكى من hub. يرسل البيانات فقط للوجهة عبر MAC address table. يوفر أداء وأماناً أفضل.
هذا الجهاز جزء مهم من أي سيناريو سيبراني (VLANs, Port Security).

**Router**
![modems and wireless access points](./images/modems-and-wireless-access-points.png)
يصل بين الشبكات عبر التوجيه باستخدام **IP address**.
يسمح بوصول WAN والإنترنت.
جزء أساسي لأي عملية تحليل ترافق، IDS/IPS، Firewalls.

**Modem**
يربط شبكتك بمزود الخدمة ISP عبر الإنترنت.

![wireless access poits](./images/wireles-access-point.png)
**Virtualization tools**
توفر وظائف الشبكة نفسها بشكل افتراضي داخل السحابة.
هذا مهم جداً لمن يريد الأمن السحابي (Cloud Security).

---

### 3. Network components, devices, and diagrams



هذا القسم مهم لأنه يجمع كل عناصر الشبكات ويعرض علاقة كل عنصر بالأمن.

**Network** = البنية الكاملة لنقاط الاتصال.
**Network devices** = أدوات تتحكم بحركة البيانات.

**التحليل الأمني** يعتمد بشدة على فهم هذه البنية.

أهم الأجهزة:

##### **a. Firewalls الجدران النارية**

جهاز يمنع ويسمح بحركة المرور حسب قواعد محددة.
يقع عادة بين الشبكة الداخلية LAN والإنترنت.
أساسي في أي بيئة سيبرانية.

##### **b. Servers الخوادم**

توفر خدمات مثل:

* DNS
* File server
* Mail server
  العميل Client يرسل طلبات، والخادم يجيب.
  معرفة هذه العلاقة مهمة عند تحليل الهجمات على الخدمات (Service Abuse, Enumeration).

![servers](./images/servers.png)

##### **c. Switches vs hubs**

* تم شرحها سابقاً.
* Switches تستخدم جداول MAC address → مهمة عند تحليل MAC spoofing أو STP attacks.

##### **d. Routers**

توجه البيانات عبر الشبكات.
تفهم IP header وتختار المسار.
مهم لفهم routing attacks – MITM – BGP hijacking (متقدم).

##### **e. Modems & Wireless Access Points**

* **Modem** بوابة الإنترنت.
* **Access Point** يبث Wi-Fi ويستقبل الإشارات اللاسلكية.
  فهم كيفية عمل Wi-Fi أساسي عند دراسة Wireless Security.

##### **f. Network Diagrams**

مهمة جداً لأي Security Analyst.
توضح الأجهزة، الروابط، المسارات.
تساعد على تحديد:

* نقاط الاختراق
* الأماكن التي تحتاج مراقبة
* مصادر التهديدات المحتملة

![network diagrams](./images/network-diagrams.png)

---

### 4. Cloud networks — شبكات السحابة



السحابة أصبحت أهم بيئة حديثة للأمن السيبراني.

**Cloud computing** = استخدام خوادم بعيدة تدار من طرف مزود الخدمة (CSP) مثل Google Cloud, AWS, Azure.

الفوائد:

* تقليل التكلفة
* قابلية التوسع
* سهولة الإدارة وانتشار عالمي

**Cloud networks** = شبكة تعمل بالكامل عبر بنية السحابة، ولا تحتاج مركز بيانات داخلي.
تتيح:

* تشغيل تطبيقات
* تخزين بيانات
* الوصول من أي مكان
* تسجيل الأحداث بسهولة
* تحكم أفضل بالهوية (Identity-based security)

ما يهم المتخصص:
مع السحابة تتغيّر طريقة الحماية من حماية أجهزة محلية إلى حماية موارد موزعة.

---

### 5. Cloud computing and Software-defined networks



هذا القسم مهم جداً للمستقبل المهني، لأنه يشرح جوهر الأمن السحابي.

##### **a. On-premise vs Cloud**

On-premise = بنية مادية داخل الشركة
Cloud = بنية افتراضية عند CSP

##### **b. ثلاث خدمات أساسية من CSP**

* **SaaS:** تطبيق جاهز للاستخدام (Gmail – Google Docs).
* **IaaS:** خوادم افتراضية + تخزين + شبكات (Virtual Machines, Load Balancers).
* **PaaS:** بيئة للمطورين لبناء التطبيقات دون إدارة البنية.

هذه المفاهيم مهمة جداً لفهم الهجمات على السحابة (IAM misconfiguration– public buckets– unsecured VMs).

![iaas paas saas](./images/iass-paas-and-saas.png)

##### **c. Hybrid Cloud**

مزيج من السحابة والبنية الداخلية.
90% من الشركات تعمل بهذه الطريقة.
مهم لفهم انتقال البيانات وتدفقها بين البيئات.

##### **d. Software-defined networks (SDN)**

شبكة تتحكم فيها البرمجيات بدل الأجهزة.
توفر:

* إدارة أسرع
* سياسات أمان مركزية
* مراقبة أفضل
* بنية افتراضية بالكامل

SDN أساس الأمن السحابي الحديث.

##### **e. فوائد السحابة و SDN**

* Reliability
* Cost reduction
* Scalability
* سهولة إضافة Firewalls أو WAF أو IDS/IPS بخطوات قليلة عبر API

---

## Network Communication

### 1. The TCP/IP Model

هذا الدرس يقدّم مراجعة سريعة لمفهوم **TCP/IP** باعتباره النموذج الأساسي للاتصال الشبكي.  
التركيز هنا على نقطتين مهمتين لأي شخص من خلفية علوم الحاسوب ويتخصص في الأمن السيبراني:

##### ما هو TCP؟
TCP أو **Transmission Control Protocol** بروتوكول يعتمد على الاتصال، يضمن:
- إنشاء Connection بين طرفين
- تقسيم البيانات إلى حزم Packets بترتيب صحيح
- إعادة إرسال ما فقد من البيانات
- ضمان وصول البيانات دون أخطاء

##### ما هو IP؟
IP أو **Internet Protocol** مسؤول عن:
- عنونة Addressing الأجهزة عبر **IP address**
- توجيه Routing الحزم بين الشبكات

##### مفهوم المنافذ Ports
يُستخدم لتحديد الخدمة المطلوب الوصول إليها على الجهاز نفسه.  
أمثلة مهمة:
- 25 SMTP
- 443 HTTPS
- 20 FTP data

#### ما هو المهم لشخص في الأمن السيبراني؟
- معرفة TCP مهم في تحليل الـ **packet behavior** أثناء التحقيقات.
- معرفة IP ضروري لفهم التحركات في الشبكات والتحقق من حالات **spoofing**.
- معرفة المنافذ مهمة لرصد **traffic anomalies** وهجمات **port scanning**.


---

### 2. Four Layers of the TCP/IP Model

النموذج يتكون من **4 Layers** فقط، وهي نسخة مبسّطة مقارنة بـ OSI.

##### Network Access Layer
- تشمل الجانب الفيزيائي: كابلات، محولات، NICs
- يتم إنشاء الحزم وتجهيزها للإرسال
- مهم في تحليل مشاكل **LAN attacks** مثل ARP spoofing

##### Internet Layer
- إضافة عناوين IP
- تحديد إن كانت البيانات ستبقى داخل LAN أو تُرسل خارجها
- بروتوكولات مهمة: **IP، ICMP**
- مهم في تحليل:
  - أخطاء routing
  - Ping sweeps
  - ICMP tunneling

##### Transport Layer
- التحكم بحركة البيانات Flow control
- البروتوكولات: **TCP، UDP**
- TCP مفيد للتطبيقات الحساسة والـ forensics
- UDP مهم في تحليل:
  - DNS attacks
  - Streaming abuse
  - DDoS reflection

##### Application Layer
- كيفية تعامل التطبيقات مع البيانات
- بروتوكولات مهمة: **HTTP، FTP، DNS، SMTP**
- أساس تحليل:
  - Web attacks
  - DNS poisoning
  - Email spoofing

![tcp ip model layers](./images/tcp-ip-model.png)
---

### 3. Learn More About the TCP/IP Model

يدخل هذا الدرس في التفاصيل التقنية لكل Layer ويحدد البروتوكولات المستخدمة في كل منها.

##### Network Access Layer (ARP)
- ARP يحوّل IP ↔ MAC  
- مهم جداً في الأمن:
  - لأن ARP spoofing أحد أشهر هجمات الشبكات الداخلية

##### Internet Layer
بروتوكولات مهمة:
- **IP**: التوجيه بين الشبكات
- **ICMP**: فحص الأخطاء والكشف عن فقد الحزم  
  يستخدم بكثرة في:
  - Reconnaissance (مثل ping sweeps)
  - Network troubleshooting

##### Transport Layer
- TCP: ضمان وصول البيانات
- UDP: عدم ضمان وصول البيانات، مناسب للخدمات الزمانية Real-time

##### Application Layer
البروتوكولات الأساسية:
- HTTP
- SMTP
- SSH
- FTP
- DNS

#### ما هو المهم؟
كل محتوى الطبقات الأربع مهم لعمل محلل أمني لأنها تمثل:
- أين قد يحدث الخلل؟
- أي بروتوكول ربما استُخدم في الهجوم؟
- أي Layer يجب فحصها أثناء التحقيق؟

![tcp vs osi](./images/TCP-vs-OSI.png)

---

### 4. The OSI Model

هذا نموذج أكثر تفصيلاً، مفيد للشرح والتواصل بين الخبراء، لكنه أقل استخداماً في الواقع اليومي مقارنة بـ TCP/IP.

#### الطبقات المهمة من منظور محلل أمني:

##### Layer 7: Application
- HTTP/HTTPS, DNS, SMTP  
- أهم Layer للهجمات اليوم:
  - SQL injection
  - XSS
  - DNS spoofing

##### Layer 6: Presentation
- مسؤول عن التشفير Encryption  
- أمثلة: SSL/TLS  
- مهم في:
  - تحليل البنية التحتية الخاصة بالـ HTTPS
  - فحص misconfigurations في certificates

##### Layer 5: Session
- مسؤول عن إنشاء الجلسات Session establishment  
- قد تكون مهمة في:
  - تحليل الـ authentication  
  - فهم آليات session hijacking

##### Layer 4: Transport
- TCP/UDP  
- نفس أهمية Transport layer في TCP/IP

##### Layer 3: Network
- IP Routing  
- مهم في تحليل:
  - هجمات spoofing
  - Traceroute
  - Network segmentation issues

##### Layer 2: Data Link
- Switches, MAC addresses  
- مهم في:
  - ARP spoofing
  - MAC flooding
  - VLAN hopping

##### Layer 1: Physical
- كابلات، إشارة كهربائية  
- نادراً ما يهم إلا عند تحليل انقطاع فعلي أو sniffing على طبقة فيزيائية

![osi model](./images/OSI-Model.png)

---

## Local and Wide Network Communication

---

### 1. IP addresses and network communication

يتناول هذا الدرس مفهوم **عناوين IP** ودورها في تحديد موقع الأجهزة على الشبكة، ويُعد أساسياً لأي شخص في الأمن السيبراني.  
تُستخدم عناوين IP لنقل البيانات بين الأجهزة، مثل عنوان المنزل في البريد. هناك نوعان أساسيان:

- **IPv4**: يتكون من 4 أرقام مفصولة بنقاط، وهو الأقدم والأكثر استخداماً.  
- **IPv6**: يتكون من 32 خانة هكساديسمل، ويحل مشكلة نفاذ عناوين IPv4.

العناوين تنقسم إلى:
- **Public IP**: يقدمه مزود الخدمة ISP، ويظهر للعالم الخارجي.
- **Private IP**: يستخدم للتواصل داخل الشبكة المحلية فقط.

كما يقدم الدرس **MAC address**، وهو معرف فيزيائي ثابت لكل جهاز، ويستخدمه السويتش لتوجيه الإطارات عبر **MAC address table**.

#### لماذا هذا مهم في الأمن السيبراني؟
- فهم الفرق بين **public/private IP** مهم في تحليل هجمات الـ NAT traversal.
- فهم **MAC address** أساسي في تحليل:
  - MAC spoofing
  - ARP attacks
- فهم IPv4 و IPv6 ضروري لمعرفة كيفية تحليل الـ logs والحزم.

---

###  Components of network layer communication  

هذا الدرس يركّز على طبقة الشبكة **Layer 3 – Network Layer** ضمن نموذج **OSI**، وهي الطبقة المسؤولة عن:
- التوجيه Routing
- عنونة IP
- تمرير الحزم بين الشبكات

#### أهم العمليات في Layer 3
##### وظائف جهاز الراوتر
- يقرأ الراوتر عنوان الوجهة داخل **IP header**.
- يستخدم **routing tables** لتحديد المسار التالي.
- يخزن العنوان للمساعدة على التوجيه لاحقاً.

#### أهمية الحزم Packets
- تُسمى **IP packets** في TCP أو **datagrams** في UDP.
- تتضمن عنوان المصدر والوجهة والمعلومات الأساسية لتوجيه الحزمة.

---

#### IP Packet Structure (IPv4)

##### مكونات IPv4 Header (13 حقل مهم)
1. Version  
2. Header Length (IHL)  
3. Type of Service (ToS) – جودة الخدمة  
4. Total Length – الحد الأقصى 65535  
5. Identification – لإعادة تجميع الحزم المجزأة  
6. Flags  
7. Fragment Offset  
8. Time to Live (TTL) – لمنع الدوران اللانهائي  
9. Protocol – مثل TCP أو UDP  
10. Header Checksum – لكشف تلف الهيدر  
11. Source IP  
12. Destination IP  
13. Options – نادراً ما تستخدم

##### Section: Data
- يمكن أن يصل الحجم الإجمالي للحزمة إلى 65535 بايت.

![ip packet header](./images/IP-packet-h.png)
![ip packet header detail](./images/ip-packet-header-detail.png)


#### لماذا هذه التفاصيل مهمة في الأمن السيبراني؟
- فحص TTL يكشف عن **hops** ويستخدم لكشف spoofing.
- فحص Identification يفيد في تحليل **packet fragmentation attacks**.
- Protocol field يساعد في كشف بروتوكول الهجوم (TCP، UDP، ICMP).
- Header checksum يوضح حدوث تلف أو محاولة tampering.

---

### 3. Difference between IPv4 and IPv6

#### ملخص الدرس
يشرح الدرس الفرق الأساسي بين IPv4 و IPv6، مع التركيز على سبب ظهور IPv6 وهو **exhaustion** أي نفاد العناوين.

#### فروقات مهمة:
##### صيغة IPv4
- أرقام عشرية (0–255)
- 32-bit
- 4.3 مليار عنوان

##### صيغة IPv6
- صيغة هكساديسمل
- 128-bit
- عدد ضخم جداً من العناوين
- دعم اختصار الأصفار باستخدام ::

#### اختلافات مهمة في الهيدر:
- IPv6 يلغي:
  - IHL
  - Identification
  - Flags
- IPv6 يضيف:
  - Flow Label: يشير إلى أن الحزمة تحتاج معالجة خاصة

#### أهمية IPv6 أمنياً:
- يقلل من private address collisions
- يسمح بآليات routing أفضل
- يبسط الهيدر مما يحسن الأداء

#### لماذا هذا مهم كمحلل أمني؟
- ستحتاج لتحليل حزم IPv6 في البيئات الحديثة.
- كثير من الهجمات تعتمد على فهم حدود IPv4 مثل NAT.
- معرفة شكل IPv6 أساسي لحل مشاكل misconfiguration في المؤسسات.

![ipv4 and ipv6](./images/IPv4-and-IPv6.png)

---

</div>