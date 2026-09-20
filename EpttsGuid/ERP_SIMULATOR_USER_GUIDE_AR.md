# دليل استخدام ErpSimulator

> يشرح هذا الدليل كيفية تثبيت وتشغيل وتجربة برنامج `ErpSimulator` خطوة بخطوة. البرنامج أداة تعليم واختبار تساعد مستخدم ERP ومطور التكامل على تجربة مكتبة `Eptts.Client` وفهم دورة حياة عمليات EPTTS دون الحاجة إلى قراءة كود واجهات WPF.

---

## 1. ما هو ErpSimulator؟

`ErpSimulator` هو تطبيق مكتبي تجريبي يعرض طريقة استخدام مكتبة:

```text
Eptts.Client.dll
```

ويسمح للمستخدم بتنفيذ أو تجربة العمليات التالية:

```text
1. حفظ إعدادات الاتصال للجلسة الحالية
2. التحقق من عبوة باستخدام VerifyProduct
3. إرسال عمليات EPTTS
4. متابعة النتيجة النهائية للرسائل
```

البرنامج ليس نظام ERP كاملًا، ولا يعد بديلًا عن قاعدة بيانات ERP أو شاشاته أو قواعد المخزون.

الغرض من البرنامج هو:

- تجربة الاتصال ببيئة EPTTS.
- فهم المدخلات المطلوبة لكل عملية.
- مشاهدة `Structured Result`.
- مشاهدة `Raw Request` و`Raw Response`.
- التمييز بين قبول الرسالة والنجاح النهائي.
- مساعدة فريق ERP على فهم طريقة استخدام المكتبة.
- تنفيذ اختبارات Staging وUAT بصورة منظمة.

---

## 2. ما الذي لا يفعله البرنامج؟

لا يقوم `ErpSimulator` تلقائيًا بـ:

- تحديث مخزون ERP.
- إنشاء أو اعتماد فواتير ERP.
- حفظ العمليات في قاعدة بيانات دائمة.
- منع التكرار بعد إغلاق التطبيق.
- إدارة المستخدمين والصلاحيات.
- تخزين الإعدادات بعد إغلاق البرنامج.
- استبدال نظام ERP الفعلي.

الإعدادات والنتائج الموجودة في البرنامج مخصصة لجلسة التشغيل الحالية ما لم تتم إضافة تخزين دائم لاحقًا.

---

## 3. تحذير مهم قبل الاستخدام

عمليات التبويب الثالث قد تغير حالة العبوة أو مسارها داخل EPTTS.

تشمل هذه العمليات:

```text
Receiving from Branch
Full Pack Dispensing
Partial Dispensing
Dispense Cancel
Return to Branch
Return Cancel
```

استخدم البرنامج أولًا في:

```text
Staging / UAT
```

ولا تستخدم بيانات Production إلا بعد اعتماد رسمي وخطة اختبار واضحة.

لا تضغط زر الإرسال أكثر من مرة لنفس العملية.

---

## 4. المتطلبات

لتشغيل البرنامج تحتاج إلى:

- نظام Windows متوافق مع إصدار التطبيق.
- ملفات تشغيل `ErpSimulator` كاملة.
- ملف `Eptts.Client.dll` المتوافق.
- ملف `Eptts.Client.xml` لظهور التوثيق عند التطوير.
- `Newtonsoft.Json.dll` أو الحزمة المطلوبة ضمن ملفات التشغيل.
- اتصال بالشبكة يسمح بالوصول إلى EPTTS.
- بيانات بيئة الاختبار.
- `IntegratorKey` صالح.
- `PharmacyGln` و`PharmacySgln` للصيدلية التجريبية.
- بيانات عبوات وشحنات مخصصة للاختبار.

---

## 5. ملفات التشغيل المتوقعة

قد تكون حزمة التشغيل بالشكل التالي:

```text
ErpSimulator
├── ErpSimulator.exe
├── ErpSimulator.dll
├── ErpSimulator.deps.json
├── ErpSimulator.runtimeconfig.json
├── Eptts.Client.dll
├── Eptts.Client.xml
├── Newtonsoft.Json.dll
└── ملفات تشغيل أخرى حسب إصدار .NET
```

لا تحذف أي ملف من مجلد التشغيل قبل التأكد من أنه غير مطلوب.

إذا ظهر خطأ تحميل Assembly، تحقق من وجود:

```text
Eptts.Client.dll
Newtonsoft.Json.dll
```

بجوار ملفات التطبيق.

---

## 6. تشغيل البرنامج

شغّل:

```text
ErpSimulator.exe
```

عند فتح البرنامج تظهر أربع تبويبات رئيسية:

```text
1. Connection
2. Read Endpoints
3. Operations
4. Message Status
```

ابدأ دائمًا من التبويب الأول.

---

# التبويب الأول: Connection

## 7. الغرض من التبويب

يستخدم تبويب `Connection` لإدخال إعدادات EPTTS والتحقق منها محليًا ثم حفظها داخل جلسة البرنامج الحالية.

لا يرسل حفظ الإعدادات وحده عملية تجارية إلى EPTTS.

---

## 8. الحقول المطلوبة

### BaseUrl

عنوان بيئة EPTTS.

مثال:

```text
https://masar-api.v2.daf-holding.com
```

أدخل عنوان البيئة فقط، ولا تضف مسار Endpoint مثل `VerifyProduct`.

### IntegratorKey

المفتاح السري الخاص بالتكامل.

لا تشارك المفتاح في:

- Screenshots.
- ملفات الدعم العادية.
- البريد غير الآمن.
- Logs.
- GitHub.

### Pharmacy GLN

معرف الصيدلية الحالية.

مثال:

```text
6221388358239
```

### Pharmacy SGLN

معرف موقع الصيدلية بصيغة EPC URI.

مثال:

```text
urn:epc:id:sgln:6221388.35823.0
```

### Timeout

مهلة انتظار HTTP Request الواحد بالثواني.

قيمة اختبار شائعة:

```text
60
```

---

## 9. التحقق من الإعدادات

اضغط زر التحقق من الإعدادات إن كان متاحًا.

يتحقق البرنامج محليًا من:

- وجود `BaseUrl`.
- صحة تنسيق URL.
- وجود `IntegratorKey`.
- صحة `PharmacyGln`.
- صحة `PharmacySgln`.
- أن `Timeout` أكبر من صفر.

التحقق المحلي لا يثبت أن:

- المفتاح فعال لدى الخادم.
- الصيدلية مخولة.
- الشبكة متاحة.
- EPTTS تعمل حاليًا.

أول اتصال فعلي يتم عند تشغيل Endpoint مثل `VerifyProduct`.

---

## 10. حفظ الإعدادات للجلسة

اضغط:

```text
Save for Session
```

بعد الحفظ يجب أن تتغير حالة البرنامج إلى رسالة قريبة من:

```text
Settings are saved for this session
```

تظل الإعدادات محفوظة داخل الذاكرة حتى إغلاق التطبيق.

بعد إغلاق البرنامج يجب عادة إدخال الإعدادات مرة أخرى، ما لم تتم إضافة آلية تخزين دائمة في إصدار لاحق.

---

## 11. أخطاء Connection الشائعة

### BaseUrl غير صحيح

تأكد من:

- وجود `https://`.
- عدم إضافة مسار Endpoint.
- عدم وجود مسافات زائدة.

### PharmacyGln غير صحيح

تأكد من أن القيمة رقمية وبالتنسيق المعتمد.

### PharmacySgln غير صحيح

تأكد من أن القيمة تبدأ مثلًا بـ:

```text
urn:epc:id:sgln:
```

### IntegratorKey فارغ

لا يمكن استخدام بقية التبويبات قبل إدخال المفتاح وحفظ الإعدادات.

---

# التبويب الثاني: Read Endpoints

## 12. الغرض من التبويب

يستخدم تبويب `Read Endpoints` للاستعلام عن EPTTS دون إرسال عملية تغير حالة العبوة.

الميزة المتاحة حاليًا:

```text
VerifyProduct
```

قد تظهر ميزات أخرى معطلة مثل:

```text
GetInvoices
Export SSCC Contents
```

لا يمكن استخدامها حتى تُضاف إلى نسخة المكتبة والبرنامج.

---

## 13. ما هو SGTIN؟

`SGTIN` هو المعرف المسلسل لعبوة واحدة محددة.

مثال:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
```

لا تستخدم بدلًا منه:

- رقم الصنف داخل ERP.
- `ItemID`.
- `ItemRefNo`.
- `GTIN` فقط.
- `SSCC`.

---

## 14. تجربة VerifyProduct

1. احفظ إعدادات الاتصال في التبويب الأول.
2. افتح:

```text
2. Read Endpoints
```

3. اختر:

```text
VerifyProduct
```

4. أدخل `SGTIN` كاملًا.
5. اضغط:

```text
Verify Product
```

6. انتظر اكتمال الطلب.

---

## 15. نتيجة VerifyProduct

يعرض البرنامج ثلاث مناطق للنتيجة:

```text
Structured Result
Raw Request
Raw Response
```

### Structured Result

يعرض نتيجة منظمة ومقروءة، وقد تشمل:

```text
HTTP Operation Success
HTTP Status Code
Error Code
Error Message
Verified
Normalized SGTIN
Pack Status
Current GLN
Is Recalled
Alerts
Product Information
```

### Raw Request

يعرض محتوى الطلب الذي أرسلته المكتبة.

### Raw Response

يعرض النص الأصلي الذي أعاده EPTTS، مع تنسيق JSON عند إمكان ذلك.

---

## 16. الفرق بين IsSuccess وVerified

قد تكون النتيجة:

```text
IsSuccess = True
Verified = False
```

المعنى:

```text
طلب HTTP نجح
لكن العبوة لم يتم التحقق منها
```

لا يكفي أن تكون:

```text
HTTP Operation Success = True
```

يجب أيضًا فحص:

```text
Verified
Pack Status
Current GLN
Is Recalled
Alerts
```

---

## 17. خصائص العبوة المهمة

### Status

الحالة الحالية للعبوة، مثل:

```text
active
dispensed
partially_dispensed
returned
in_transit
```

### CurrentGln

الجهة المرتبطة حاليًا بالعبوة.

قارنها مع `PharmacyGln` لمعرفة هل العبوة مرتبطة بالصيدلية الحالية.

### IsRecalled

توضح هل العبوة مستدعاة.

لا تنفذ صرفًا عاديًا على عبوة مستدعاة.

### Alerts

تنبيهات مهمة يجب مراجعتها قبل إرسال عملية جديدة.

---

## 18. أزرار Read Endpoints

### Verify Product

يرسل طلب التحقق.

### Cancel Request

يلغي انتظار الطلب المحلي.

إلغاء الانتظار لا يغير حالة العبوة.

### Clear Result

يمسح مناطق النتائج الثلاث دون تغيير أي بيانات في EPTTS.

---

## 19. اختبارات VerifyProduct المقترحة

```text
[ ] عبوة موجودة
[ ] عبوة غير موجودة
[ ] SGTIN فارغ
[ ] SGTIN بصيغة غير صحيحة
[ ] عبوة active
[ ] عبوة dispensed
[ ] عبوة partially_dispensed
[ ] عبوة غير مرتبطة بالصيدلية الحالية
[ ] IntegratorKey غير صحيح
[ ] Timeout
[ ] Cancel Request
```

---

# التبويب الثالث: Operations

## 20. تحذير التبويب

عمليات هذا التبويب قد تغير حالة العبوة داخل EPTTS.

استخدم بيانات اختبار مخصصة، ولا تضغط زر الإرسال أكثر من مرة.

العمليات المتاحة:

```text
Receiving from Branch
Full Pack Dispensing
Partial Dispensing
Dispense Cancel
Return to Branch
Return Cancel
```

---

## 21. التسلسل الصحيح لأي عملية

اتبع هذا التسلسل:

```text
اختيار العملية
→ إدخال البيانات
→ Verify Before Operation أو Review Input
→ مراجعة النتيجة
→ Submit Operation مرة واحدة
→ حفظ StatusQueryIdentifier
→ متابعة Message Status
→ اعتماد النجاح عند S - Successful فقط
```

---

## 22. معنى Verify Before Operation

للعمليات على عبوة محددة، ينفذ البرنامج `VerifyProduct` أولًا ويراجع مبدئيًا:

- وجود العبوة.
- الحالة الحالية.
- `CurrentGln`.
- `IsRecalled`.
- `Alerts`.

إذا تغير `SGTIN` أو العملية بعد التحقق، يجب إجراء التحقق مرة أخرى.

التحقق المحلي لا يضمن قبول العملية النهائي، لكنه يمنع أخطاء واضحة قبل الإرسال.

---

## 23. Receiving from Branch

تستخدم لاستلام شحنة أو عبوات من فرع.

الحقول:

```text
Source Branch GLN
Source Branch SGLN
EPC List
Invoice Number
```

داخل `EPC List` أدخل قيمة واحدة في كل سطر.

يمكن أن تكون:

- `SSCC` لشحنة كاملة.
- `SGTINs` لعبوات منفردة.

لا تدخل `SSCC` وكل محتوياته في الطلب نفسه دون متطلب واضح.

### خطوات التجربة

1. اختر `Receiving from Branch`.
2. أدخل بيانات الفرع المرسل.
3. أدخل `SSCC` أو `SGTINs`.
4. أدخل رقم الفاتورة إن وجد.
5. اضغط `Review Receiving Input`.
6. راجع المدخلات.
7. اضغط `Submit Receiving` مرة واحدة.
8. انتقل إلى `Message Status`.

---

## 24. Full Pack Dispensing

تستخدم لصرف عبوة كاملة.

الحالة المبدئية المتوقعة عادة:

```text
Status = active
CurrentGln = PharmacyGln
IsRecalled = false
```

الحقول:

```text
Pack SGTIN
Patient Reference
Prescription Reference
```

استخدم مراجع داخلية فقط، ولا تدخل اسم المريض أو الرقم القومي أو الهاتف.

### خطوات التجربة

1. اختر `Full Pack Dispensing`.
2. أدخل `SGTIN` لعبوة `active`.
3. أدخل مراجع اختبار داخلية.
4. اضغط `Verify Before Operation`.
5. تأكد أن العبوة جاهزة.
6. اضغط `Dispense Full Pack` مرة واحدة.
7. تابع `StatusQueryIdentifier`.
8. بعد `S - Successful` نفذ `VerifyProduct` مجددًا.
9. الحالة المتوقعة:

```text
dispensed
```

---

## 25. Partial Dispensing

تستخدم لصرف كمية من عبوة تسمح بالصرف الجزئي.

الحالات المبدئية المستخدمة عادة:

```text
active
partially_dispensed
```

الحقول:

```text
Pack SGTIN
Quantity to Dispense
Patient Reference
Prescription Reference
```

### معنى Quantity

`Quantity` هي كمية العملية الحالية وفق وحدة الصرف المعتمدة للمنتج.

لا تعني بالضرورة قرصًا واحدًا.

### خطوات التجربة

1. اختر `Partial Dispensing`.
2. أدخل `SGTIN` مناسبًا.
3. أدخل كمية أكبر من صفر.
4. اضغط `Verify Before Operation`.
5. اضغط `Dispense Partial` مرة واحدة.
6. تابع الرسالة حتى النتيجة النهائية.

---

## 26. Dispense Cancel

تستخدم لإلغاء صرف عبوة كاملة سابقة.

الحالة المبدئية المتوقعة:

```text
dispensed
```

لا تستخدم العملية لإلغاء `Partial Dispensing` إلا إذا كانت هناك مواصفة معتمدة لذلك.

### خطوات التجربة

1. استخدم عبوة صُرفت بالكامل بنجاح.
2. اختر `Dispense Cancel`.
3. أدخل `SGTIN`.
4. اضغط `Verify Before Operation`.
5. تأكد من أن الحالة `dispensed`.
6. اضغط `Cancel Dispensing` مرة واحدة.
7. تابع الرسالة.
8. بعد النجاح نفذ `VerifyProduct`.
9. الحالة المتوقعة عادة:

```text
active
```

---

## 27. Return to Branch

تستخدم لإرسال عبوة من الصيدلية إلى فرع كمرتجع.

الحقول:

```text
Pack SGTIN
Destination Branch GLN
Destination Branch SGLN
Return Request Number
```

### ReturnRequestNumber

رقم تجاري فريد لعملية المرتجع.

مثال تجريبي:

```text
RET-UAT-000123
```

يجب حفظ الرقم لأنه مطلوب عند إلغاء المرتجع.

### خطوات التجربة

1. اختر `Return to Branch`.
2. أدخل عبوة `active` مرتبطة بالصيدلية.
3. أدخل بيانات الفرع المستقبل.
4. أدخل `ReturnRequestNumber` فريدًا.
5. اضغط `Verify Before Operation`.
6. اضغط `Return to Branch` مرة واحدة.
7. احفظ رقم المرتجع ومعرف المتابعة.
8. تابع الرسالة حتى النتيجة النهائية.

---

## 28. Return Cancel

تستخدم لإلغاء مرتجع سابق ما زال قابلًا للإلغاء.

استخدم القيم الأصلية نفسها:

```text
نفس SGTIN
نفس DestinationGln
نفس ReturnRequestNumber
```

لا تستخدم:

```text
MessageId
RequestInstanceIdentifier
StatusQueryIdentifier
```

مكان `ReturnRequestNumber`.

### خطوات التجربة

1. احصل على بيانات المرتجع الأصلي.
2. اختر `Return Cancel`.
3. أدخل `SGTIN` الأصلي.
4. أدخل `DestinationGln` الأصلي.
5. أدخل `ReturnRequestNumber` الأصلي.
6. اضغط `Verify Before Operation`.
7. اضغط `Cancel Return` مرة واحدة.
8. تابع الرسالة حتى النجاح أو الفشل النهائي.
9. بعد النجاح نفذ `VerifyProduct`.

الحالة المتوقعة عادة:

```text
active
```

---

## 29. أزرار Operations

### Verify Before Operation

يتحقق من العبوة قبل العملية.

في `Receiving` قد يتغير النص إلى مراجعة المدخلات لأن العملية قد تعتمد على `SSCC` أو قائمة عبوات.

### Submit Operation

يرسل العملية المختارة.

لا تضغطه أكثر من مرة.

### Cancel Request

يوقف انتظار HTTP محليًا.

لا يثبت أن EPTTS لم تستلم العملية.

### Copy Status Identifier

ينسخ `StatusQueryIdentifier` الناتج من العملية.

### Clear Result

يمسح العرض الحالي فقط.

لا يلغي أي عملية داخل EPTTS.

---

## 30. نتيجة Operations

تعرض المناطق التالية:

```text
Structured Result
Raw Request
Raw Response
```

قد تتضمن النتيجة المنظمة:

```text
HTTP Operation Success
HTTP Status Code
Status Type
Body Code
Message ID
Request Instance Identifier
Status Query Identifier
Accepted For Processing
Can Query Final Status
```

---

## 31. معنى HTTP 202 وI001

إذا ظهرت نتيجة مثل:

```text
HTTP 202
I001
Accepted For Processing = True
```

فهذا يعني:

```text
تم قبول الرسالة للمعالجة
```

ولا يعني:

```text
العملية نجحت نهائيًا
```

الخطوة التالية دائمًا:

```text
Message Status
```

---

## 32. التعامل مع Timeout أو Cancel أثناء الإرسال

إذا انتهت المهلة أو ضغط المستخدم `Cancel Request` بعد بدء إرسال عملية:

- لا تعِد إرسال العملية مباشرة.
- راجع `Raw Request` و`Raw Response`.
- احتفظ بأي معرف عاد من المكتبة.
- استعلم عن حالة الرسالة عند توفر المعرف.
- استخدم عبوة اختبار أخرى إذا تعذر حسم نتيجة المحاولة السابقة.

---

# التبويب الرابع: Message Status

## 33. الغرض من التبويب

يستخدم تبويب `Message Status` لمعرفة النتيجة النهائية لعملية سبق إرسالها.

المدخل الأساسي:

```text
StatusQueryIdentifier
```

قد ينتقل المعرف تلقائيًا من تبويب `Operations` إلى هذا التبويب.

يمكن أيضًا نسخه ولصقه يدويًا.

---

## 34. Query Once

ينفذ استعلامًا واحدًا عن حالة الرسالة.

استخدمه عندما تريد:

- تحديث الحالة يدويًا.
- اختبار معرف معين.
- معرفة هل الرسالة ما زالت `Pending`.

---

## 35. Wait for Final Status

ينفذ Polling تلقائيًا حتى:

- تظهر نتيجة نهائية.
- تنتهي مدة الانتظار القصوى.
- يتم إلغاء الانتظار.
- يحدث فشل يمنع الاستمرار.

الحقول:

```text
Polling Interval
Maximum Wait Time
```

قيم تجريبية مناسبة:

```text
Polling Interval = 3 seconds
Maximum Wait Time = 30 seconds
```

انتهاء المدة لا يعني فشل العملية الأصلية.

---

## 36. نتيجة Message Status

تعرض:

```text
Structured Result
Processing Log
Raw Request
Raw Response
```

الخصائص المهمة:

```text
Message Status
Is Final
Is Successful
Is Application Error
First Error Message
```

---

## 37. S - Successful

عند ظهور:

```text
S - Successful
```

فهذا يعني أن العملية الأصلية نجحت نهائيًا.

يمكن بعد ذلك:

- تنفيذ `VerifyProduct` للتأكد من حالة العبوة.
- تحديث نتيجة سيناريو الاختبار.
- اعتماد نجاح العملية داخل ERP الحقيقي وفق تصميمه.

---

## 38. E - Application Error

عند ظهور:

```text
E - Application Error
```

فإن استعلام الحالة نفسه نجح، لكن العملية الأصلية فشلت وظيفيًا.

راجع:

```text
First Error Message
Processing Log
Raw Response
```

لا تعِد العملية تلقائيًا قبل فهم سبب الخطأ.

---

## 39. الحالة غير النهائية

إذا لم تكن الحالة `Successful` أو `Application Error`، فقد تكون الرسالة ما زالت قيد المعالجة.

الإجراء:

```text
انتظر
→ Query Once لاحقًا
أو
Wait for Final Status
```

لا تعِد إرسال العملية الأصلية.

---

## 40. Cancel Request في Message Status

يلغي الاستعلام أو Polling المحلي فقط.

لا يلغي:

- عملية الاستلام.
- عملية الصرف.
- المرتجع.
- إلغاء المرتجع.

احتفظ بـ `StatusQueryIdentifier` وأعد الاستعلام لاحقًا.

---

# سيناريو تجربة كامل

## 41. السيناريو الأول: VerifyProduct فقط

```text
1. افتح Connection
2. أدخل الإعدادات
3. Save for Session
4. افتح Read Endpoints
5. أدخل SGTIN
6. Verify Product
7. راجع Structured Result
8. راجع Raw Request وRaw Response
```

هذا هو أول اختبار يجب تنفيذه.

---

## 42. السيناريو الثاني: Full Dispensing ثم المتابعة

استخدم عبوة مخصصة للاختبار وحالتها `active`.

```text
1. تحقق من العبوة في Read Endpoints
2. افتح Operations
3. اختر Full Pack Dispensing
4. أدخل SGTIN
5. أدخل مراجع اختبار داخلية
6. Verify Before Operation
7. راجع أن العبوة جاهزة
8. Submit مرة واحدة
9. انتقل إلى Message Status
10. Wait for Final Status
11. انتظر S - Successful أو E - Application Error
12. عند النجاح نفذ VerifyProduct
13. تحقق من الحالة dispensed
```

---

## 43. السيناريو الثالث: Dispense Cancel

بعد نجاح السيناريو السابق:

```text
1. اختر Dispense Cancel
2. استخدم SGTIN نفسها
3. Verify Before Operation
4. تأكد من status = dispensed
5. Submit مرة واحدة
6. تابع Message Status
7. عند النجاح نفذ VerifyProduct
8. تحقق من status = active
```

---

## 44. السيناريو الرابع: Return ثم Return Cancel

```text
1. استخدم عبوة active
2. اختر Return to Branch
3. أدخل DestinationGln وDestinationSgln
4. أنشئ ReturnRequestNumber فريدًا
5. احفظ ReturnRequestNumber خارج البرنامج مؤقتًا
6. Verify Before Operation
7. Submit مرة واحدة
8. تابع Message Status حتى النجاح
9. اختر Return Cancel عند الحاجة
10. استخدم نفس SGTIN
11. استخدم نفس DestinationGln
12. استخدم نفس ReturnRequestNumber
13. Submit مرة واحدة
14. تابع Message Status
15. عند النجاح نفذ VerifyProduct
```

---

# الألوان والحالات المرئية

## 45. معنى ألوان المؤشرات

قد يستخدم البرنامج الألوان التالية:

```text
رمادي
→ Ready أو لم يبدأ طلب

أزرق
→ الطلب قيد التنفيذ

أخضر
→ نتيجة ناجحة أو جاهزية صحيحة

برتقالي
→ تحذير أو Pending أو Timeout أو Cancellation

أحمر
→ فشل أو Application Error
```

اللون مساعد بصري فقط. اقرأ النص التفصيلي دائمًا.

---

# استكشاف الأخطاء

## 46. البرنامج لا يبدأ

تحقق من:

- وجود ملفات التشغيل كاملة.
- وجود Runtime المناسب.
- وجود `Eptts.Client.dll`.
- وجود `Newtonsoft.Json.dll`.
- عدم حظر Windows للملفات المنسوخة.

راجع Windows Event Viewer أو رسالة الخطأ عند الحاجة.

---

## 47. تظهر رسالة Settings are required

افتح التبويب الأول واضغط:

```text
Save for Session
```

بعد إدخال جميع القيم المطلوبة.

---

## 48. HTTP 401

راجع:

- `IntegratorKey`.
- البيئة.
- عدم وجود مسافات إضافية.
- عدم استخدام مفتاح Staging في Production أو العكس.

لا ترسل المفتاح في Screenshot.

---

## 49. HTTP 403

راجع:

- صلاحيات المفتاح.
- `PharmacyGln`.
- الصيدلية المستخدمة.
- ملكية الرسالة أو العبوة.

---

## 50. HTTP 404

راجع:

- `BaseUrl`.
- `StatusQueryIdentifier`.
- البيئة.
- عدم استخدام `MessageId` بدل معرف المتابعة.

---

## 51. VerifyProduct نجح لكن Verified = False

المعنى أن الاتصال نجح، لكن العبوة لم يتم التحقق منها.

راجع:

- `SGTIN`.
- بيئة الاختبار.
- وجود العبوة في البيئة.
- `ErrorCode` و`ErrorMessage`.
- `Raw Response`.

---

## 52. لا يمكن الضغط على Submit Operation

قد يكون السبب:

- لم تحفظ إعدادات الاتصال.
- لم تدخل الحقول المطلوبة.
- لم تنفذ `Verify Before Operation`.
- العبوة ليست جاهزة للعملية.
- تغير `SGTIN` بعد التحقق.
- العملية المختارة لا تسمح بحالة العبوة.

---

## 53. Wait for Final Status انتهى دون نتيجة

هذا لا يعني فشل العملية.

احتفظ بالمعرف واضغط `Query Once` لاحقًا أو أعد `Wait for Final Status`.

لا تعِد إرسال العملية الأصلية.

---

## 54. Return Cancel يعيد No PENDING return found

راجع أنك استخدمت:

```text
نفس SGTIN
نفس DestinationGln
نفس ReturnRequestNumber
```

لا تستخدم `MessageId` أو `StatusQueryIdentifier` بدل `ReturnRequestNumber`.

وقد يكون المرتجع لم يعد قابلًا للإلغاء.

---

## 55. Raw Request أو Raw Response فارغة

قد يحدث ذلك إذا:

- فشل التحقق المحلي قبل إرسال HTTP.
- لم يُرسل الطلب.
- حدث استثناء قبل إنشاء النص الخام.
- لم يعد الخادم Body.

راجع `Structured Result` و`Error Message`.

---

# الأمان والخصوصية

## 56. حماية IntegratorKey

لا تعرض المفتاح في:

- Screenshots.
- فيديوهات التدريب.
- ملفات الدعم العامة.
- GitHub.
- البريد غير الآمن.

إذا احتجت إلى مشاركة Screenshot، أخفِ حقل المفتاح بالكامل.

---

## 57. بيانات المرضى

استخدم مراجع داخلية تجريبية مثل:

```text
UAT-PATIENT-001
UAT-RX-001
```

لا تستخدم بيانات أشخاص حقيقيين في بيئة الاختبار.

---

## 58. Raw Request وRaw Response

قد تحتوي النصوص الخام على معرفات عبوات ومستندات ومراجع داخلية.

شاركها فقط مع فريق مخول، وبعد مراجعتها وإزالة أي بيانات حساسة.

---

# تجهيز طلب دعم

## 59. المعلومات المطلوبة

عند حدوث مشكلة، جهز:

```text
إصدار ErpSimulator
إصدار Eptts.Client.dll
اسم البيئة
اسم العملية
وقت المشكلة مع Time Zone
PharmacyGln
SGTIN أو SSCC عند السماح
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
HttpStatusCode
ErrorCode
ErrorMessage
RawRequest بعد المراجعة
RawResponse بعد المراجعة
خطوات إعادة المشكلة
```

لا ترسل:

```text
IntegratorKey
Authorization Headers
كلمات مرور
بيانات مريض حقيقية
```

---

# قائمة اختبار المستخدم

## 60. Connection

```text
[ ] البرنامج يعمل
[ ] BaseUrl تم إدخاله
[ ] IntegratorKey تم إدخاله بأمان
[ ] PharmacyGln صحيح
[ ] PharmacySgln صحيح
[ ] Timeout موجب
[ ] Save for Session نجح
```

---

## 61. Read Endpoints

```text
[ ] VerifyProduct يعمل
[ ] Structured Result يظهر
[ ] Raw Request يظهر
[ ] Raw Response يظهر
[ ] الفرق بين IsSuccess وVerified مفهوم
[ ] Clear Result يعمل
[ ] Cancel Request يعمل
```

---

## 62. Operations

```text
[ ] العمليات الست تظهر
[ ] الحقول تتغير حسب العملية
[ ] Verify Before Operation يعمل
[ ] Submit لا يعمل قبل التحقق عند الحاجة
[ ] العملية تُرسل مرة واحدة فقط
[ ] StatusQueryIdentifier يظهر
[ ] Copy Status Identifier يعمل
[ ] HTTP 202 لا يُعتبر نجاحًا نهائيًا
```

---

## 63. Message Status

```text
[ ] Identifier ينتقل أو يُلصق بنجاح
[ ] Query Once يعمل
[ ] Wait for Final Status يعمل
[ ] Processing Log يظهر
[ ] S - Successful مفهوم
[ ] E - Application Error مفهوم
[ ] Timeout يبقي العملية غير محسومة
[ ] Cancel Request لا يلغي العملية الأصلية
```

---

# حدود الإصدار الحالي

## 64. التخزين المؤقت

النسخة الحالية لا تحتوي بالضرورة على سجل عمليات دائم في قاعدة بيانات.

لذلك:

- احفظ `StatusQueryIdentifier` خارجيًا أثناء الاختبارات المهمة.
- لا تعتمد على استمرار البيانات بعد إغلاق التطبيق.
- لا تستخدم البرنامج كبديل لسجل ERP الإنتاجي.

---

## 65. الميزات غير المتاحة حاليًا

قد تكون الميزات التالية مخططة فقط:

```text
GetInvoices
Export SSCC Contents
```

إذا ظهرت معطلة، فهذا سلوك متوقع.

لا توجد مشكلة في الاتصال لمجرد أن الخيار معطل.

---

# مسار التجربة الموصى به

## 66. للمستخدم الجديد

نفذ بالترتيب:

```text
1. اقرأ هذا الدليل
2. استخدم بيئة Staging
3. احفظ إعدادات Connection
4. اختبر VerifyProduct فقط
5. اختبر Message Status بمعرف سابق إن توفر
6. اختبر Receiving ببيانات UAT
7. اختبر Full Dispensing
8. اختبر Dispense Cancel
9. اختبر Partial Dispensing
10. اختبر Return
11. اختبر Return Cancel
12. احتفظ بالأدلة والنتائج
```

لا تبدأ بـ `Return Cancel` أو `Dispense Cancel` دون نجاح العملية الأصلية أولًا.

---

## 67. خلاصة الدليل

يعمل `ErpSimulator` عبر أربع خطوات رئيسية:

```text
Connection
→ حفظ الإعدادات

Read Endpoints
→ VerifyProduct وقراءة حالة العبوة

Operations
→ إرسال عملية واحدة والحصول على StatusQueryIdentifier

Message Status
→ معرفة النجاح أو الفشل النهائي
```

القواعد الأهم أثناء الاستخدام:

```text
استخدم Staging أولًا
لا تشارك IntegratorKey
لا تضغط Submit مرتين
HTTP 202 ليس نجاحًا نهائيًا
احتفظ بـ StatusQueryIdentifier
اعتمد النجاح عند S - Successful فقط
Cancellation لا يلغي العملية الأصلية
Timeout لا يثبت أن الرسالة لم تصل
Return Cancel يستخدم ReturnRequestNumber الأصلي
نفذ VerifyProduct بعد النجاح عند الحاجة
```
