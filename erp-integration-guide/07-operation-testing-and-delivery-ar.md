# الجزء السابع: التشغيل والاختبار والتسليم

> يشرح هذا الجزء كيفية تشغيل تكامل `Eptts.Client` بصورة منظمة، واختباره من أول Build حتى اختبارات UAT، ثم تجهيز حزمة التسليم والنشر في بيئة Production. الهدف هو التأكد من أن فريق ERP لا يكتفي بنجاح الكود، بل يتحقق من دورة حياة العمليات، ومنع التكرار، واستمرار متابعة الرسائل بعد إغلاق التطبيق، وسلامة الإعدادات والأسرار.

---

## 1. الهدف من مرحلة التشغيل والاختبار

قبل نشر التكامل في بيئة Production يجب إثبات ما يلي:

```text
المكتبة تُحمّل بصورة صحيحة
الإعدادات المحلية صحيحة
الاتصال الفعلي يعمل
VerifyProduct يعيد نتائج قابلة للقراءة
عمليات التغيير تُحفظ Pending
Message Status يُحدّث النتيجة النهائية
العمليات المكررة ممنوعة
Timeout وCancellation لا ينتجان إعادة إرسال خطرة
الأسرار لا تظهر في Logs أو UI
يمكن استكمال متابعة Pending بعد إعادة التشغيل
```

نجاح `Build` وحده لا يثبت نجاح التكامل.

ونجاح طلب واحد لا يثبت جاهزية النظام للإنتاج.

---

## 2. مراحل الانتقال إلى Production

استخدم المراحل التالية بالترتيب:

```text
1. Development Verification
2. Local Integration Testing
3. Staging Integration Testing
4. User Acceptance Testing
5. Pre-Production Readiness Review
6. Production Deployment
7. Post-Deployment Monitoring
8. Handover and Support
```

لا تنتقل إلى مرحلة لاحقة قبل إغلاق مشكلات المرحلة السابقة أو قبولها رسميًا.

---

# القسم الأول: التحقق من بيئة التطوير

## 3. التحقق من ملفات المكتبة

تأكد من وجود الملفات التالية داخل مجلد ثابت في مشروع ERP:

```text
Eptts.Client.dll
Eptts.Client.xml
Eptts.Client.pdb
Eptts.Client.deps.json
```

الملف الإلزامي للاستخدام هو:

```text
Eptts.Client.dll
```

والملف المطلوب لظهور التوثيق في `IntelliSense` هو:

```text
Eptts.Client.xml
```

يجب أن تأتي DLL وXML من Build نفسه.

---

## 4. التحقق من Reference

راجع ملف المشروع:

```xml
<ItemGroup>
  <Reference Include="Eptts.Client">
    <HintPath>Libraries\Eptts\Eptts.Client.dll</HintPath>
    <Private>true</Private>
  </Reference>
</ItemGroup>
```

تأكد من:

```text
HintPath صحيح
Copy Local = True
Private = true عند استخدام csproj
```

ثم نفذ:

```text
Build
→ Clean Solution
→ Rebuild Solution
```

---

## 5. التحقق من Dependencies

تأكد من وجود إصدار متوافق من:

```text
Newtonsoft.Json
```

لا تستخدم Reference يدويًا وحزمة NuGet للإصدار نفسه في الوقت نفسه.

بعد Build تحقق من مجلد Output:

```text
Eptts.Client.dll
Eptts.Client.xml
Newtonsoft.Json.dll
```

---

## 6. التحقق من Target Framework

المكتبة تستهدف:

```text
.NET Standard 2.0
```

يجب أن يستهدف مشروع ERP Framework متوافقًا معها.

اختبر داخل Target Framework الفعلي للمشروع، وليس داخل مشروع تجريبي فقط.

إذا كان ERP يستخدم أكثر من تطبيق، مثل:

```text
Desktop Client
Windows Service
Web API
Background Job
```

فيجب اختبار تحميل المكتبة في كل Process سيستخدمها.

---

## 7. تسجيل إصدار المكتبة

قبل بدء الاختبارات، سجل:

```text
اسم الملف
File Version
Assembly Version
حجم الملف
تاريخ الملف
Hash اختياري
إصدار Newtonsoft.Json
Target Framework للمشروع
```

يمكن استخدام PowerShell لقراءة معلومات DLL:

```powershell
(Get-Item ".\Libraries\Eptts\Eptts.Client.dll").VersionInfo
```

ويمكن حساب Hash:

```powershell
Get-FileHash ".\Libraries\Eptts\Eptts.Client.dll" -Algorithm SHA256
```

الـ Hash مفيد للتأكد من أن جميع البيئات تستخدم الملف نفسه.

---

## 8. اختبار XML Documentation

داخل Visual Studio، جرّب:

```csharp
using Eptts.Client.Configuration;

EpttsOptions options =
    new EpttsOptions();

options.Validate();
```

مرر مؤشر الفأرة فوق:

```text
EpttsOptions
Validate
```

يجب أن يظهر الشرح من `Eptts.Client.xml`.

إذا لم يظهر:

```text
تأكد من تطابق اسم DLL وXML
→ احذف bin وobj
→ أعد فتح Visual Studio
→ Rebuild
```

---

# القسم الثاني: اختبار الإعدادات والاتصال

## 9. اختبار الإعدادات المحلية

أنشئ `EpttsOptions` بقيم بيئة Staging:

```csharp
EpttsOptions options =
    new EpttsOptions
    {
        BaseUrl =
            stagingBaseUrl,

        IntegratorKey =
            stagingIntegratorKey,

        PharmacyGln =
            stagingPharmacyGln,

        PharmacySgln =
            stagingPharmacySgln,

        Timeout =
            TimeSpan.FromSeconds(60)
    };

options.Validate();
```

المتغيرات السابقة يجب أن تأتي من مصدر إعدادات آمن.

لا تضع القيم السرية داخل الدليل أو Source Control.

---

## 10. اختبارات Validate المطلوبة

اختبر الحالات التالية:

```text
[ ] جميع القيم صحيحة
[ ] BaseUrl فارغ
[ ] BaseUrl غير صالح
[ ] IntegratorKey فارغ
[ ] PharmacyGln فارغ
[ ] PharmacyGln بطول غير صحيح
[ ] PharmacyGln يحتوي على أحرف
[ ] PharmacySgln فارغ
[ ] PharmacySgln بصيغة غير صحيحة
[ ] Timeout يساوي صفرًا
[ ] Timeout سالب
```

المتوقع في الحالات غير الصحيحة:

```csharp
EpttsConfigurationException
```

لا يرسل `Validate()` أي HTTP Request.

---

## 11. اختبار إنشاء EpttsClient

```csharp
options.Validate();

using (EpttsClient client =
    new EpttsClient(options))
{
    /*
     * Client was created.
     * No HTTP request was sent yet.
     */
}
```

هذا الاختبار يثبت:

- تحميل DLL.
- توفر Dependencies.
- صحة الإعدادات محليًا.
- إمكانية إنشاء Client.

ولا يثبت:

- صحة `IntegratorKey` لدى الخادم.
- توفر الشبكة.
- قبول الصيدلية.
- عمل Endpoint الفعلي.

---

## 12. أول اختبار اتصال فعلي

استخدم `VerifyProductAsync` على `SGTIN` مخصص للاختبار:

```csharp
using (EpttsClient client =
    new EpttsClient(options))
{
    EpttsResult<VerifyProductResponse> result =
        await client.VerifyProductAsync(
            testSgtin,
            cancellationToken);
}
```

سجل النتيجة التالية:

```text
IsSuccess
HttpStatusCode
ErrorCode
ErrorMessage
Verified
Pack.Status
Pack.CurrentGln
Pack.IsRecalled
Alerts
RawRequest
RawResponse
```

لا تسجل `IntegratorKey`.

---

# القسم الثالث: خطة الاختبار

## 13. أنواع الاختبارات المطلوبة

يجب تنفيذ الأنواع التالية:

```text
Build Tests
Unit Tests
Integration Tests
Negative Tests
Concurrency Tests
Recovery Tests
Security Tests
User Acceptance Tests
Deployment Tests
```

---

## 14. Build Tests

اختبر:

```text
[ ] Debug Build
[ ] Release Build
[ ] Clean ثم Rebuild
[ ] Build على جهاز مطور آخر
[ ] Build من CI إن توفر
[ ] تشغيل بدون سورس المكتبة
[ ] نسخ Output إلى مجلد مستقل وتشغيله
```

يجب ألا يعتمد التطبيق على مسار محلي خاص بجهاز المطور.

---

## 15. Unit Tests

ركز على منطق ERP المحلي:

```text
[ ] تحويل إعدادات ERP إلى EpttsOptions
[ ] التحقق من المدخلات المطلوبة
[ ] اختيار نوع العملية الصحيح
[ ] منع العملية المكررة
[ ] تحويل Submission إلى Pending
[ ] تحويل S - Successful إلى Successful
[ ] تحويل E - Application Error إلى Failed
[ ] إبقاء الحالة غير النهائية Pending
[ ] تصنيف Timeout كـ NeedsReview عند الإرسال
[ ] استخدام نفس ReturnRequestNumber في Return Cancel
[ ] عدم تسجيل IntegratorKey
```

يفضل وضع واجهة حول Service التكامل حتى يمكن استخدام Mock أو Fake في Unit Tests.

---

## 16. Integration Tests

تستخدم `Eptts.Client.dll` وتتصل ببيئة Staging فعلية.

اختبر:

```text
[ ] VerifyProduct
[ ] GetMessageStatusAsync
[ ] WaitForFinalStatusAsync
[ ] ReceiveFromBranchAsync
[ ] DispenseFullPackAsync
[ ] DispensePartialAsync
[ ] CancelDispensingAsync
[ ] ReturnToBranchAsync
[ ] CancelReturnAsync
```

يجب استخدام بيانات اختبار معتمدة، وعدم استخدام عبوات Production.

---

## 17. Negative Tests

اختبر المدخلات والنتائج غير الصحيحة عمدًا:

```text
[ ] SGTIN فارغ
[ ] SGTIN غير صالح
[ ] SSCC غير صالح
[ ] GLN غير صحيح
[ ] SGLN غير صحيح
[ ] Quantity = 0
[ ] Quantity سالبة
[ ] ReturnRequestNumber فارغ
[ ] StatusQueryIdentifier فارغ
[ ] IntegratorKey غير صحيح
[ ] عبوة غير موجودة
[ ] عبوة لا تخص الصيدلية
[ ] عملية غير مناسبة لحالة العبوة
```

الهدف هو التأكد من أن النظام:

- يمنع الإرسال المحلي عند الإمكان.
- يعرض رسالة مفهومة.
- يسجل التفاصيل الفنية.
- لا يطبق أثر نجاح غير صحيح.

---

## 18. Concurrency Tests

اختبر:

```text
[ ] مستخدمان يرسلان العملية نفسها
[ ] شاشتان تعملان على SGTIN نفسها
[ ] ضغط زر الإرسال مرتين بسرعة
[ ] Job يعمل أثناء محاولة إرسال جديدة
[ ] Processان يشغلان Job في الوقت نفسه
[ ] إعادة فتح مستند حالته Pending
```

النتيجة المطلوبة:

```text
يُرسل طلب واحد فقط
أو
تُمنع العملية الثانية قبل الإرسال
```

يجب أن يكون المنع داخل قاعدة البيانات أو طبقة Service، وليس داخل UI فقط.

---

## 19. Recovery Tests

اختبر حالات التعطل:

```text
[ ] إغلاق التطبيق بعد حفظ Pending
[ ] إغلاق التطبيق قبل انتهاء Polling
[ ] إعادة تشغيل Background Job
[ ] انقطاع الشبكة أثناء VerifyProduct
[ ] انقطاع الشبكة أثناء Submission
[ ] انقطاع الشبكة أثناء Message Status
[ ] تعطل قاعدة البيانات بعد استجابة EPTTS
[ ] إعادة تشغيل الخادم مع عمليات Pending
```

يجب أن يستطيع النظام استكمال متابعة العمليات المحفوظة بعد إعادة التشغيل.

---

## 20. Security Tests

تحقق من:

```text
[ ] IntegratorKey لا يظهر في UI
[ ] IntegratorKey لا يظهر في Logs
[ ] IntegratorKey لا يظهر في RawRequest
[ ] IntegratorKey لا يظهر في RawResponse
[ ] IntegratorKey غير موجود في Source Control
[ ] إعدادات Production لا تظهر للمستخدم غير المخول
[ ] RawRequest وRawResponse محميان
[ ] PatientReference لا يحتوي على بيانات شخصية مباشرة
[ ] صلاحيات تعديل الإعدادات محدودة
```

---

# القسم الرابع: سيناريوهات UAT الكاملة

## 21. متطلبات UAT

يجب تجهيز بيانات اختبار لكل سيناريو:

```text
بيئة Staging
صيدلية اختبار
فرع مرسل
فرع مستقبل
IntegratorKey للاختبار
عبوات بحالات معروفة
SSCC للاختبار
مستخدمون وصلاحيات مناسبة
مستندات ERP اختبارية
```

لا تستخدم العبوة نفسها في سيناريوهين متعارضين دون إعادة حالتها بطريقة معتمدة.

---

## 22. سيناريو Receiving باستخدام SSCC

### البيانات المطلوبة

```text
SSCC صالح وموجه للصيدلية
SourceGln
SourceSgln
InvoiceNumber اختياري
```

### الخطوات

```text
1. افتح مستند الاستلام
2. حدد الصيدلية الحالية
3. امسح SSCC
4. امنع وجود Receiving Pending للـ SSCC نفسه
5. أرسل ReceiveFromBranchAsync
6. تحقق من HTTP Submission
7. احفظ StatusQueryIdentifier
8. اجعل العملية Pending
9. تابع Message Status
10. انتظر S - Successful
11. حدث مستند الاستلام
12. نفذ VerifyProduct على عبوة من الشحنة عند الحاجة
```

### معايير القبول

```text
[ ] لم يُرسل الطلب مرتين
[ ] تم حفظ جميع المعرفات
[ ] لم يعتمد ERP الاستلام عند HTTP 202 فقط
[ ] تغيرت الحالة إلى Successful بعد النتيجة النهائية
[ ] لم يظهر IntegratorKey
```

---

## 23. سيناريو Receiving باستخدام SGTINs

### البيانات المطلوبة

```text
قائمة SGTINs صحيحة
SourceGln
SourceSgln
```

### الخطوات

أرسل كل `SGTIN` مرة واحدة داخل `EpcList`، ثم اتبع دورة `Pending` وMessage Status نفسها.

### اختبارات إضافية

```text
[ ] قائمة فارغة
[ ] SGTIN مكرر داخل القائمة
[ ] قيمة غير صالحة داخل القائمة
[ ] عبوة غير موجهة للصيدلية
```

---

## 24. سيناريو Full Pack Dispensing

### الحالة قبل العملية

```text
Verified = true
Status = active
CurrentGln = PharmacyGln
IsRecalled = false
Alerts لا تمنع العملية
```

### الخطوات

```text
1. نفذ VerifyProduct
2. تحقق من الحالة والملكية والاستدعاء والتنبيهات
3. امنع وجود عملية متعارضة Pending
4. أنشئ FullDispensingRequest
5. أرسل DispenseFullPackAsync
6. احفظ العملية Pending
7. تابع Message Status
8. عند S - Successful حدث مستند ERP
9. نفذ VerifyProduct
10. تحقق من Status = dispensed
```

### معايير القبول

```text
[ ] العبوة لم تُصرف قبل النجاح النهائي
[ ] تم حفظ StatusQueryIdentifier
[ ] أصبحت العملية Successful
[ ] أظهر VerifyProduct الحالة dispensed
[ ] لم تُرسل بيانات مريض مباشرة
```

---

## 25. سيناريو Dispense Cancel

### الحالة قبل العملية

```text
Verified = true
Status = dispensed
CurrentGln = PharmacyGln
```

### الخطوات

```text
1. نفذ VerifyProduct
2. تأكد من الحالة dispensed
3. اربط الإلغاء بعملية الصرف الأصلية
4. أرسل CancelDispensingAsync
5. احفظ Pending
6. تابع Message Status
7. عند S - Successful حدث ERP
8. نفذ VerifyProduct
9. تحقق من Status = active
```

### معايير القبول

```text
[ ] لا يمكن إلغاء عبوة active
[ ] لا يستخدم المسار لإلغاء Partial Dispensing دون مواصفة
[ ] لا تُعاد العبوة إلى المخزون عند HTTP 202 فقط
[ ] عادت العبوة إلى active بعد النجاح النهائي
```

---

## 26. سيناريو Partial Dispensing

### الحالة قبل العملية

```text
Verified = true
Status = active أو partially_dispensed
CurrentGln = PharmacyGln
IsRecalled = false
المنتج يسمح بالصرف الجزئي
```

### الخطوات

```text
1. نفذ VerifyProduct
2. تحقق من قواعد المنتج
3. أدخل Quantity الحالية
4. تحقق من وحدة القياس والكمية
5. أرسل DispensePartialAsync
6. احفظ Quantity مع العملية Pending
7. تابع Message Status
8. عند S - Successful ثبت الكمية داخل ERP
9. نفذ VerifyProduct عند الحاجة
```

### معايير القبول

```text
[ ] Quantity أكبر من صفر
[ ] تم حفظ Quantity
[ ] لم تعتمد الكمية قبل النجاح النهائي
[ ] لم تُرسل العملية مرتين
```

---

## 27. سيناريو Return to Branch

### الحالة قبل العملية

```text
Verified = true
Status = active
CurrentGln = PharmacyGln
DestinationGln صحيح
DestinationSgln صحيح
ReturnRequestNumber فريد
```

### الخطوات

```text
1. نفذ VerifyProduct
2. أنشئ ReturnRequestNumber داخل ERP
3. احفظ الرقم في مستند المرتجع
4. أنشئ ReturnRequest
5. أرسل ReturnToBranchAsync
6. احفظ Pending والمعرفات
7. تابع Message Status
8. عند S - Successful حدث مستند المرتجع
9. احتفظ بالقيم الأصلية لاستخدام Return Cancel عند الحاجة
```

### معايير القبول

```text
[ ] ReturnRequestNumber فريد ومحفوظ
[ ] لم يستخدم MessageId مكانه
[ ] تم حفظ DestinationGln وDestinationSgln
[ ] لم تعتمد العملية عند HTTP 202 فقط
```

---

## 28. سيناريو Return Cancel

### البيانات المطلوبة

من سجل المرتجع الأصلي:

```text
نفس SGTIN
نفس DestinationGln
نفس ReturnRequestNumber
```

### الخطوات

```text
1. حمّل المرتجع الأصلي من ERP
2. لا تنشئ ReturnRequestNumber جديدًا
3. أنشئ ReturnCancelRequest بالقيم الأصلية
4. أرسل CancelReturnAsync
5. احفظ سجل Return Cancel مستقلًا بحالة Pending
6. تابع Message Status
7. عند S - Successful حدث سجل الإلغاء والمرتجع الأصلي
8. نفذ VerifyProduct
9. تحقق من عودة العبوة إلى active لدى PharmacyGln
```

### معايير القبول

```text
[ ] استُخدم ReturnRequestNumber الأصلي
[ ] استُخدم DestinationGln الأصلي
[ ] لم يُستخدم MessageId أو StatusQueryIdentifier مكان رقم المرتجع
[ ] تم حفظ سجل الإلغاء منفصلًا
[ ] عادت العبوة إلى الحالة المتوقعة بعد النجاح
```

---

## 29. سيناريو Application Error

لكل عملية، يجب اختبار نتيجة:

```text
E - Application Error
```

النتيجة المطلوبة داخل ERP:

```text
Integration Operation = Failed
ERP Document = EPTTS Failed أو الحالة المعتمدة
FinalErrorMessage محفوظة
لا يُطبق أثر النجاح
لا تُعاد العملية تلقائيًا
```

---

## 30. سيناريو Timeout أثناء Submission

النتيجة المطلوبة:

```text
LocalStatus = NeedsReview
RawRequest محفوظ عند توفره
لا يُعاد الإرسال تلقائيًا
العملية المتعارضة ممنوعة
يظهر تنبيه للدعم
تتم محاولة استعادة المعرف أو متابعة الرسالة
```

لا تستخدم:

```text
LocalStatus = Failed
```

فقط لأن مدة الانتظار انتهت.

---

## 31. سيناريو Cancellation

### أثناء VerifyProduct

يمكن إعادة الاستعلام لاحقًا.

### أثناء Submission

تعامل مع النتيجة باعتبارها غير مؤكدة إذا بدأ الإرسال.

### أثناء WaitForFinalStatusAsync

أوقف Polling فقط، واحتفظ بالعملية `Pending`.

---

# القسم الخامس: التحقق من Background Job

## 32. اختبارات Job الأساسية

```text
[ ] يقرأ Pending فقط
[ ] يتجاهل Successful وFailed
[ ] يرفض سجل Pending دون StatusQueryIdentifier أو يحوله إلى NeedsReview
[ ] يستخدم إعدادات الصيدلية الصحيحة
[ ] يحدث LastStatusCheckAt
[ ] يبقي الحالة غير النهائية Pending
[ ] يحول IsSuccessful إلى Successful
[ ] يحول IsApplicationError إلى Failed
[ ] يحفظ FirstErrorMessage
[ ] لا يرسل العملية الأصلية مرة أخرى
```

---

## 33. اختبار إعادة تشغيل Job

1. أنشئ عملية `Pending`.
2. أوقف Job أو التطبيق.
3. أعد تشغيل النظام.
4. شغل Job.
5. تأكد أنه استكمل متابعة العملية نفسها.

يجب ألا يعتمد Job على بيانات موجودة في الذاكرة فقط.

---

## 34. اختبار تداخل Jobs

شغل نسختين من Job في الوقت نفسه.

النتيجة المطلوبة:

```text
السجل يُحجز بواسطة Process واحد
ولا تتم معالجته بالتوازي مرتين
```

استخدم قفل قاعدة بيانات أو `RowVersion` أو آلية موثوقة مناسبة.

---

## 35. اختبار سياسة Polling

تحقق من:

```text
NextStatusCheckAt
LastStatusCheckAt
RetryCount
BatchSize
```

يجب ألا يرسل Job استعلامات متكررة بسرعة غير ضرورية.

---

# القسم السادس: معايير القبول النهائية

## 36. معايير القبول التقنية

```text
[ ] Debug Build ناجح
[ ] Release Build ناجح
[ ] التطبيق يعمل من Output مستقل
[ ] DLL وDependencies موجودة
[ ] XML Documentation تعمل
[ ] لا توجد References لمسارات أجهزة المطورين
[ ] Nullable Warnings المهمة مغلقة
[ ] جميع Exceptions المعروفة معالجة
[ ] CancellationToken مدعوم
[ ] لا توجد أسرار في Source Control
```

---

## 37. معايير القبول الوظيفية

```text
[ ] VerifyProduct يعمل
[ ] جميع العمليات الست تعمل في Staging
[ ] كل Submission تحفظ Pending
[ ] Message Status يحدث النتيجة النهائية
[ ] S - Successful فقط يعتمد النجاح
[ ] E - Application Error يسجل Failed
[ ] Timeout لا يؤدي إلى إعادة إرسال تلقائية
[ ] العمليات المكررة ممنوعة
[ ] Return Cancel يستخدم القيم الأصلية
[ ] VerifyProduct بعد النجاح يعرض الحالة المتوقعة
```

---

## 38. معايير القبول التشغيلية

```text
[ ] Background Job يعمل بعد إعادة التشغيل
[ ] يمكن عرض العمليات Pending
[ ] يمكن عرض NeedsReview
[ ] يوجد تنبيه للعمليات القديمة
[ ] Logs تحتوي على Correlation IDs
[ ] فريق الدعم يعرف مكان RawRequest وRawResponse
[ ] توجد سياسة احتفاظ بالسجلات
[ ] توجد آلية لتدوير IntegratorKey
[ ] توجد خطة Rollback للنشر
```

---

## 39. توقيع UAT

يجب توثيق نتيجة كل سيناريو:

```text
Test Case ID
Operation Type
Input Data
Expected Result
Actual Result
Status
Tester
Test Date
Evidence
Notes
```

يجب أن يوقع أو يعتمد UAT ممثلون عن:

- فريق ERP.
- فريق الأعمال أو الصيدليات.
- فريق التكامل.
- فريق البنية أو التشغيل عند الحاجة.

---

# القسم السابع: تجهيز حزمة التسليم

## 40. هيكل حزمة التسليم المقترح

```text
EpttsClientIntegrationPackage
├── App
│   ├── Eptts.Client.dll
│   ├── Eptts.Client.xml
│   ├── Eptts.Client.pdb
│   ├── Eptts.Client.deps.json
│   └── Newtonsoft.Json.dll أو Dependency Metadata
│
├── Samples
│   ├── 01_Connection
│   ├── 02_ReadEndpoints
│   ├── 03_MessageStatus
│   ├── 04_Operations
│   └── 05_Workflows
│
├── Docs
│   ├── EPTTS_ERP_INTEGRATION_GUIDE_AR.md
│   ├── API_NOTES.md
│   └── CHANGELOG.md
│
├── Tests
│   ├── UAT_Test_Cases.xlsx أو المستند المعتمد
│   └── Test_Evidence
│
├── Release
│   ├── VERSION.txt
│   ├── CHECKSUMS.txt
│   └── RELEASE_NOTES.md
│
└── Support
    └── SUPPORT_REQUEST_TEMPLATE.md
```

لا يلزم أن يحتوي التسليم على سورس المكتبة إذا كان نطاق التسليم هو DLL فقط.

---

## 41. ملف VERSION.txt

اقترح المحتوى التالي:

```text
Product: Eptts.Client
File: Eptts.Client.dll
Version: <version>
Target Framework: netstandard2.0
Build Date: <UTC date>
Newtonsoft.Json Version: <version>
Environment Compatibility: Staging / Production
Documentation Version: <version>
```

استبدل القيم داخل `< >` بالقيم الفعلية.

---

## 42. ملف CHECKSUMS.txt

احسب SHA256 للملفات الأساسية:

```powershell
Get-FileHash ".\Eptts.Client.dll" -Algorithm SHA256
Get-FileHash ".\Eptts.Client.xml" -Algorithm SHA256
```

احفظ القيم في:

```text
CHECKSUMS.txt
```

يساعد ذلك فريق التشغيل على التأكد من سلامة الملفات وعدم اختلافها بين البيئات.

---

## 43. ملف RELEASE_NOTES.md

يجب أن يحتوي على:

```text
الإصدار
تاريخ الإصدار
الدوال المتاحة
التغييرات
الإصلاحات
التغييرات غير المتوافقة
Dependencies
خطوات التحديث
الاختبارات المنفذة
المشكلات المعروفة
```

لا تنشر إصدارًا جديدًا من DLL دون Release Notes.

---

## 44. ملف CHANGELOG.md

سجل التغييرات بين الإصدارات، مثل:

```markdown
## 1.1.0

### Added
- Added a new read endpoint.

### Changed
- Improved validation messages.

### Fixed
- Fixed message-status parsing for an empty log list.
```

يجب أن تعكس الأمثلة أسماء الدوال والتغييرات الفعلية فقط.

---

## 45. وثائق Samples

يجب أن يحتوي مجلد `Samples` على `README.md` يوضح ترتيب القراءة:

```text
1. ConnectionOptionsSample.cs
2. ClientFactorySample.cs
3. VerifyProductSample.cs
4. MessageStatusSample.cs
5. WaitForFinalStatusSample.cs
6. ReceivingSample.cs
7. FullPackDispensingSample.cs
8. PartialDispensingSample.cs
9. DispenseCancelSample.cs
10. ReturnToBranchSample.cs
11. ReturnCancelSample.cs
12. Workflow Samples
```

وضح أن Samples:

- ليست Dependencies إلزامية.
- لا تحتوي على `IntegratorKey` حقيقي.
- لا تعتمد على WPF.
- مناسبة للتعلم والنسخ المنضبط.
- لا تستبدل تصميم التخزين والـ Job داخل ERP.

---

# القسم الثامن: خطة النشر

## 46. قبل النشر

```text
[ ] UAT معتمد
[ ] نسخة DLL النهائية محددة
[ ] Hash محفوظ
[ ] Release Notes جاهزة
[ ] Backup موجود
[ ] إعدادات Production جاهزة
[ ] IntegratorKey مخزن آمنًا
[ ] Pharmacy GLN وSGLN معتمدان
[ ] Job مجدول
[ ] Monitoring جاهز
[ ] Rollback Plan معتمد
[ ] فريق الدعم على علم بموعد النشر
```

---

## 47. نافذة النشر

اختر وقتًا يسمح بـ:

- انخفاض عدد العمليات.
- وجود فريق دعم.
- مراقبة النظام بعد النشر.
- تنفيذ اختبار Smoke Test.
- الرجوع إلى الإصدار السابق عند الحاجة.

تجنب تحديث DLL أثناء وجود عدد كبير من العمليات `Pending` دون خطة واضحة.

---

## 48. خطوات النشر المقترحة

```text
1. إيقاف أو تعليق Job بأمان
2. انتظار اكتمال العمليات المحلية الجارية
3. نسخ Backup من الملفات الحالية
4. نشر Eptts.Client.dll وEptts.Client.xml المتطابقين
5. نشر Dependencies المطلوبة
6. تحديث إعدادات Production دون كشف المفتاح
7. تشغيل Migration لجدول التكامل إن وجدت
8. تشغيل التطبيق أو Service
9. تنفيذ Smoke Test
10. إعادة تشغيل Job
11. مراقبة Logs وPending
12. توثيق نتيجة النشر
```

---

## 49. Smoke Test بعد النشر

نفذ اختبارًا محدودًا وآمنًا:

```text
[ ] التطبيق يبدأ دون خطأ تحميل Assembly
[ ] options.Validate() تعمل
[ ] VerifyProduct يعمل ببيانات اختبار معتمدة
[ ] Message Status يعمل لمعرف مناسب
[ ] Job يعمل ويقرأ Pending
[ ] لا يظهر IntegratorKey في Logs
[ ] لا توجد زيادة غير طبيعية في 401 أو 403 أو Timeout
```

لا تنفذ عملية تغير حالة عبوة في Production كاختبار إلا باستخدام سيناريو وبيانات معتمدة رسميًا.

---

## 50. خطة Rollback

يجب أن تحدد الخطة:

```text
الإصدار السابق من DLL
الإصدار السابق من Dependencies
نسخة إعدادات التطبيق
طريقة إيقاف Job
طريقة إعادة الملفات
كيفية التعامل مع Pending التي أُنشئت أثناء النشر
مالك قرار Rollback
الوقت الأقصى لاتخاذ القرار
```

لا تحذف سجلات العمليات `Pending` أثناء Rollback.

الإصدار السابق يجب أن يستطيع قراءة السجلات القائمة أو يجب توفير إجراء ترحيل واضح.

---

## 51. تحديث قاعدة البيانات

إذا أضاف الإصدار الجديد حقولًا أو جداول:

- اجعل Migration قابلة للتكرار أو محددة الحالة.
- اختبرها على نسخة قريبة من Production.
- أنشئ Backup.
- لا تحذف بيانات تاريخية دون اعتماد.
- افصل تغييرات التوسعة عن الحذف كلما أمكن.

---

# القسم التاسع: المراقبة بعد النشر

## 52. أول ساعة بعد النشر

راقب:

```text
Application startup errors
Assembly load errors
HTTP 401 و403
HTTP 500 و503
Timeouts
عدد Pending
عمر أقدم Pending
Background Job failures
Database errors
Duplicate prevention events
```

---

## 53. أول يوم تشغيل

راجع:

```text
نسبة نجاح VerifyProduct
عدد العمليات المرسلة
عدد Successful
عدد Failed
عدد NeedsReview
متوسط زمن النتيجة النهائية
العمليات التي لم تطابق VerifyProduct بعد النجاح
```

---

## 54. المراقبة المستمرة

أنشئ Dashboard أو تقارير دورية تعرض:

```text
Pending by Age
Failures by ErrorCode
Requests by Pharmacy
Average Processing Time
HTTP Status Distribution
Oldest Pending Operations
NeedsReview Queue
```

---

## 55. حدود التنبيه

تُحدد الأرقام بالتعاون مع فريق التشغيل، لكن يجب وجود تنبيهات عند:

- ارتفاع `Pending` بصورة غير معتادة.
- وجود عمليات أقدم من الحد.
- تكرار `401` أو `403`.
- ارتفاع `Timeout`.
- توقف Job.
- ظهور `NeedsReview` دون معالجة.

---

# القسم العاشر: الدعم الفني

## 56. البيانات المطلوبة لطلب الدعم

عند فتح طلب دعم، أرسل:

```text
EnvironmentName
Library Version
OperationType
OperationId المحلي
PharmacyGln
SGTIN أو SSCC عند السماح
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
HttpStatusCode
ErrorCode
ErrorMessage
Timestamp مع Time Zone
RawRequest بعد المراجعة
RawResponse بعد المراجعة
خطوات إعادة المشكلة
Expected Result
Actual Result
```

---

## 57. بيانات لا تُرسل للدعم دون قناة آمنة

```text
IntegratorKey
Authorization Headers
Passwords
Connection Strings
بيانات مريض شخصية
ملفات قاعدة بيانات كاملة
```

إذا احتاج الدعم بيانات خام، راجعها وأزل أي معلومات حساسة قبل المشاركة.

---

## 58. نموذج طلب دعم

```markdown
# EPTTS Support Request

## Environment
- Environment: Staging
- Eptts.Client Version: <version>
- ERP Version: <version>

## Operation
- Operation Type: <operation>
- Local Operation ID: <id>
- Pharmacy GLN: <gln>
- Request Instance Identifier: <identifier>
- Message ID: <message id>
- Status Query Identifier: <status identifier>

## Result
- HTTP Status Code: <code>
- Error Code: <error code>
- Error Message: <message>
- Timestamp: <timestamp with timezone>

## Description
<steps and observed behavior>

## Attachments
- Sanitized RawRequest
- Sanitized RawResponse
- Relevant logs

## Security
- IntegratorKey is not attached.
- Authorization headers are not attached.
```

---

# القسم الحادي عشر: التسليم والتدريب

## 59. جلسة التسليم

يجب أن تتضمن جلسة التسليم:

```text
شرح الإعدادات
تنفيذ VerifyProduct
إرسال عملية اختبار
قراءة SubmissionResponse
عرض سجل Pending
تشغيل Message Status
شرح Successful وApplication Error
شرح Timeout وNeedsReview
شرح منع التكرار
شرح ReturnRequestNumber
شرح Logs والدعم
```

---

## 60. تدريب فريق ERP

يجب أن يستطيع المطور بعد التدريب:

```text
إضافة DLL إلى مشروع جديد
إنشاء EpttsOptions
تنفيذ VerifyProduct
إرسال كل عملية
حفظ StatusQueryIdentifier
متابعة الرسائل
قراءة الأخطاء
منع التكرار
تحديث DLL
فتح طلب دعم كامل
```

---

## 61. تدريب فريق التشغيل

يجب أن يعرف فريق التشغيل:

```text
مكان DLL والإصدار
مكان الإعدادات
طريقة تدوير IntegratorKey
مكان Logs
مكان العمليات Pending
طريقة تشغيل وإيقاف Job
التنبيهات
Rollback Plan
قنوات الدعم
```

---

## 62. محضر التسليم

يفضل أن يحتوي على:

```text
نطاق التسليم
إصدار المكتبة
إصدار الدليل
الملفات المسلمة
البيئات التي تم اختبارها
نتيجة UAT
المشكلات المعروفة
الميزات المؤجلة
جهات الاتصال
تاريخ التسليم
أسماء الموافقين
```

---

# القسم الثاني عشر: الميزات المؤجلة

## 63. GetInvoices

إذا لم تكن الدالة موجودة في DLL الحالية، سجلها كميزة مؤجلة:

```text
Status: Planned
Not available in current public API
Do not implement direct HTTP calls in ERP
```

تضاف إلى الدليل بعد تنفيذها واختبارها وإصدار DLL جديدة.

---

## 64. Export SSCC Contents

طبق السياسة نفسها:

```text
Status: Planned
Not available in current public API
Do not assume method signature
```

لا تُكتب أمثلة نهائية قبل اعتماد Request وResponse.

---

# القسم الثالث عشر: قائمة التسليم النهائية

## 65. ملفات المكتبة

```text
[ ] Eptts.Client.dll
[ ] Eptts.Client.xml
[ ] Eptts.Client.pdb عند الحاجة
[ ] Eptts.Client.deps.json عند الحاجة
[ ] Newtonsoft.Json Dependency محددة
```

---

## 66. الوثائق

```text
[ ] الدليل الكامل بصيغة Markdown
[ ] README للـ Samples
[ ] RELEASE_NOTES.md
[ ] CHANGELOG.md
[ ] VERSION.txt
[ ] CHECKSUMS.txt
[ ] Support Request Template
```

---

## 67. الاختبارات

```text
[ ] Unit Tests ناجحة
[ ] Integration Tests ناجحة
[ ] Negative Tests منفذة
[ ] Concurrency Tests منفذة
[ ] Recovery Tests منفذة
[ ] Security Tests منفذة
[ ] UAT معتمد
[ ] Smoke Test موثق
```

---

## 68. التشغيل

```text
[ ] إعدادات Staging منفصلة
[ ] إعدادات Production منفصلة
[ ] IntegratorKey محمي
[ ] Background Job مجدول
[ ] Monitoring جاهز
[ ] Alerts جاهزة
[ ] Pending Queue قابلة للمراجعة
[ ] NeedsReview Queue قابلة للمراجعة
[ ] Backup جاهز
[ ] Rollback Plan معتمد
```

---

## 69. الدعم

```text
[ ] جهات الاتصال معروفة
[ ] نموذج الدعم جاهز
[ ] فريق الدعم يستطيع تتبع المعرفات
[ ] لا تُشارك الأسرار
[ ] سياسة الاحتفاظ بالـ Logs معتمدة
[ ] المشكلات المعروفة موثقة
```

---

## 70. خلاصة الجزء السابع

الانتقال الصحيح إلى Production يجب أن يتبع هذا المسار:

```text
Build ناجح
→ Unit Tests
→ Integration Tests في Staging
→ Negative وConcurrency وRecovery Tests
→ UAT كامل
→ تجهيز Release Package
→ Pre-Production Review
→ نشر منظم
→ Smoke Test
→ مراقبة Pending وErrors
→ تسليم وتدريب ودعم
```

ولا يعتبر التكامل جاهزًا قبل تحقق الشروط التالية:

```text
كل عملية تغير حالة تُحفظ Pending
كل Pending يمكن استكمالها بعد إعادة التشغيل
كل نتيجة نهائية تُحوّل إلى Successful أو Failed
Timeout لا يؤدي إلى إعادة إرسال تلقائية
العمليات المكررة ممنوعة
IntegratorKey محمي
Return Cancel يستخدم ReturnRequestNumber الأصلي
الإصدار والملفات والـ Hash موثقة
خطة Rollback والدعم جاهزة
```
