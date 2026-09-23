# الملحق A: قاموس المصطلحات السريع

> مرجع مختصر للمصطلحات والاختصارات المتخصصة المستخدمة في دليل تكامل `Eptts.Client` مع أنظمة ERP. يظهر الاسم الكامل باللغة الإنجليزية لكل اختصار كلما أمكن، مع شرح وظيفته في التكامل والتنبيه إلى مواضع الخلط الشائعة.

---

## 1. طريقة استخدام القاموس

كل مصطلح يتضمن:

- الاختصار والاسم الكامل باللغة الإنجليزية، عند توفرهما.
- تعريفًا عربيًا مختصرًا.
- استخدامه داخل تكامل EPTTS.
- تحذيرًا عمليًا عند الحاجة.

لا يتناول هذا القاموس المصطلحات البرمجية البديهية للمطور، ويركز على مصطلحات التتبع والرسائل والنتائج والتشغيل.

للتفاصيل والأمثلة الكاملة، راجع:

```text
02-essential-concepts-ar.md
03-results-and-errors-ar.md
```

---

# المصطلحات العامة

## EPTTS: Egyptian Pharmaceutical Track and Trace System

الاسم الكامل:

```text
Egyptian Pharmaceutical Track and Trace System
```

نظام تتبع وتعقب المنتجات الدوائية الذي يستقبل عمليات العبوات ويعالجها.

يستخدم ERP مكتبة `Eptts.Client` لتنفيذ عمليات التحقق والاستلام والصرف والمرتجع ومتابعة الرسائل.

بعض العمليات تُقبل أولًا ثم تظهر نتيجتها النهائية لاحقًا.

---

## ERP: Enterprise Resource Planning

الاسم الكامل:

```text
Enterprise Resource Planning
```

نظام تخطيط موارد المؤسسة الذي يدمج مكتبة `Eptts.Client`.

ERP مسؤول عن:

- المستندات والمخزون.
- قاعدة البيانات.
- المستخدمين والصلاحيات.
- حفظ العمليات المعلقة.
- متابعة النتائج النهائية.

المكتبة لا تعدّل قاعدة بيانات ERP تلقائيًا.

---

## EPC: Electronic Product Code

الاسم الكامل:

```text
Electronic Product Code
```

رمز إلكتروني يستخدم لتعريف عناصر أو مواقع أو وحدات لوجستية بصورة قابلة للتبادل.

تظهر منه أنواع متخصصة مثل `SGTIN` و`SSCC` و`SGLN`.

---

## EPC URI: Electronic Product Code Uniform Resource Identifier

الاسم الكامل:

```text
Electronic Product Code Uniform Resource Identifier
```

صيغة نصية لتمثيل معرف EPC.

أمثلة:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
urn:epc:id:sscc:80026600.000444363
urn:epc:id:sgln:6221388.35823.0
```

يجب تمرير المعرف كاملًا دون حذف البداية أو تغيير ترتيب الأجزاء.

---

## EPCIS: Electronic Product Code Information Services

الاسم الكامل:

```text
Electronic Product Code Information Services
```

معيار يستخدم لتمثيل أحداث التتبع ومشاركتها.

تنشئ المكتبة رسائل EPCIS المطلوبة للعمليات، ولا يحتاج مطور ERP إلى بنائها يدويًا.

---

## URI: Uniform Resource Identifier

الاسم الكامل:

```text
Uniform Resource Identifier
```

معرف نصي يستخدم لتحديد مورد أو هوية بصورة موحدة.

يظهر في صيغ EPC المستخدمة في هذا الدليل.

---

## URN: Uniform Resource Name

الاسم الكامل:

```text
Uniform Resource Name
```

نوع من URI يُستخدم في بداية معرفات EPC:

```text
urn:epc:id:...
```

---

## HTTP: Hypertext Transfer Protocol

الاسم الكامل:

```text
Hypertext Transfer Protocol
```

البروتوكول الذي تستخدمه المكتبة لإرسال الطلبات إلى EPTTS واستقبال الاستجابات.

الرمز `HTTP 202` يعني قبول الرسالة للمعالجة، وليس نجاح العملية نهائيًا.

---

## DLL: Dynamic-Link Library

الاسم الكامل:

```text
Dynamic-Link Library
```

ملف مكتبة برمجية يضيفه مشروع ERP كمرجع.

ملف المكتبة في هذا التكامل:

```text
Eptts.Client.dll
```

---

## UAT: User Acceptance Testing

الاسم الكامل:

```text
User Acceptance Testing
```

اختبار قبول المستخدم الذي يُنفذ في بيئة اختبار ببيانات معتمدة للتأكد من صلاحية السيناريو التجاري من بدايته حتى النتيجة النهائية.

---

## UTC: Coordinated Universal Time

الاسم الكامل:

```text
Coordinated Universal Time
```

مرجع زمني موحد يفيد في تسجيل أوقات البناء والتشغيل والأحداث بين الأنظمة والبيئات المختلفة.

---

# المكتبة وإعدادات الاتصال

## Eptts.Client

مكتبة C# المستخدمة لربط ERP بخدمات EPTTS.

اسم الملف:

```text
Eptts.Client.dll
```

الـ Namespace الأساسي:

```csharp
Eptts.Client
```

Target Framework:

```text
.NET Standard 2.0
```

---

## EpttsClient

الـ Class الرئيسية المستخدمة لاستدعاء Endpoints.

```csharp
using (EpttsClient client =
    new EpttsClient(options))
{
    // Execute an endpoint here.
}
```

إنشاء `EpttsClient` لا يرسل طلبًا. يبدأ الاتصال عند استدعاء إحدى الدوال.

---

## EpttsOptions

Class تحتوي على إعدادات الاتصال والصيدلية الحالية.

أهم الخصائص:

```text
BaseUrl
IntegratorKey
PharmacyGln
PharmacySgln
Timeout
```

يمكن التحقق منها محليًا باستخدام:

```csharp
options.Validate();
```

---

## BaseUrl

عنوان بيئة EPTTS.

مثال:

```text
https://masar-api.v2.daf-holding.com
```

أدخل عنوان البيئة فقط، ولا تضف مسار Endpoint يدويًا.

---

## IntegratorKey

مفتاح سري يستخدم لمصادقة التكامل.

لا تحفظه في Source Code أو GitHub أو Logs أو Screenshots أو النصوص الخام.

---

## Timeout

مهلة انتظار طلب واحد.

```csharp
Timeout =
    TimeSpan.FromSeconds(60);
```

لا تمثل مدة المعالجة النهائية للرسالة بعد قبولها.

---

## Staging

بيئة اختبار أو تكامل تستخدم قبل Production.

يجب أن يكون لها عنوان ومفتاح وصيدلية وبيانات اختبار مستقلة.

---

## Production

بيئة التشغيل الفعلية.

لا تنتقل إليها قبل إتمام اختبارات Staging وUAT واعتماد خطة النشر والرجوع.

---

# معرفات المنتج والعبوة والشحنة

## GTIN: Global Trade Item Number

الاسم الكامل:

```text
Global Trade Item Number
```

معرف نوع المنتج التجاري.

يحدد المنتج، لكنه لا يحدد عبوة مادية منفردة.

لا تستخدم GTIN وحده عندما تطلب الدالة SGTIN.

---

## Serial Number

الرقم المسلسل الذي يميز عبوة محددة عن غيرها من عبوات المنتج نفسه.

يدخل مع هوية المنتج في تكوين SGTIN.

---

## SGTIN: Serialized Global Trade Item Number

الاسم الكامل:

```text
Serialized Global Trade Item Number
```

معرف عبوة مسلسلة واحدة.

مثال:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
```

يستخدم في:

```text
VerifyProduct
Full Pack Dispensing
Partial Dispensing
Dispense Cancel
Return to Branch
Return Cancel
```

لا تضع بدلًا منه GTIN أو SSCC أو رقم الصنف الداخلي.

---

## SSCC: Serial Shipping Container Code

الاسم الكامل:

```text
Serial Shipping Container Code
```

معرف حاوية أو شحنة لوجستية.

مثال:

```text
urn:epc:id:sscc:80026600.000444363
```

يستخدم عادة في Receiving عند استلام حاوية أو شحنة كاملة.

لا تستخدمه في عملية صرف عبوة منفردة.

---

## EPC List: Electronic Product Code List

الاسم الكامل:

```text
Electronic Product Code List
```

قائمة معرفات EPC تمرر إلى بعض العمليات، خصوصًا Receiving.

قد تحتوي على SSCC لشحنة كاملة أو قائمة SGTINs لعبوات منفردة.

لا ترسل SSCC وجميع محتوياته في العملية نفسها إلا إذا كانت المواصفة تطلب ذلك صراحة.

---

## DataMatrix

رمز ثنائي الأبعاد على العبوة قد يحتوي على GTIN والرقم المسلسل والتشغيلة وتاريخ الانتهاء.

نص DataMatrix الخام ليس بالضرورة SGTIN جاهزًا. يجب تحليله بالمكوّن المعتمد قبل تمرير المعرف للمكتبة.

---

## Pack

كائن داخل `VerifyProductResponse` يحتوي على بيانات العبوة.

قد يشمل:

```text
Sgtin
Gtin
Serial
BatchNumber
ExpiryDate
Status
CurrentGln
IsRecalled
ParentSscc
ManufacturerGln
```

افحص وجود `Pack` قبل قراءة خصائصها.

---

## Product

كائن داخل `VerifyProductResponse` يحتوي على بيانات المنتج العامة عند توفرها.

لا تستبدل Master Data الخاصة بـ ERP تلقائيًا دون سياسة مزامنة معتمدة.

---

# معرفات الجهات والمواقع

## GLN: Global Location Number

الاسم الكامل:

```text
Global Location Number
```

معرف جهة أو موقع تجاري.

قد يظهر كـ:

```text
PharmacyGln
SourceGln
DestinationGln
CurrentGln
```

يجب فهم وظيفة كل قيمة وعدم استبدال إحداها بالأخرى.

---

## SGLN: Serialized Global Location Number

الاسم الكامل:

```text
Serialized Global Location Number
```

معرف موقع بصيغة EPC URI.

قد يظهر كـ:

```text
PharmacySgln
SourceSgln
DestinationSgln
```

لا تنشئ SGLN يدويًا من GLN دون قواعد معتمدة.

---

## PharmacyGln

GLN الصيدلية التي ينفذ ERP العملية باسمها.

```csharp
options.PharmacyGln
```

لا تستخدم GLN الفرع المرسل أو المستقبل مكانه.

---

## PharmacySgln

SGLN موقع الصيدلية الحالية.

يجب أن يمثل الموقع نفسه الخاص بـ `PharmacyGln`.

---

## SourceGln

GLN الجهة أو الفرع المرسل في Receiving.

```text
SourceGln
→ الفرع المرسل

PharmacyGln
→ الصيدلية المستلمة
```

---

## SourceSgln

SGLN موقع الفرع المرسل في Receiving، ويجب أن يتوافق مع `SourceGln`.

---

## DestinationGln

GLN الفرع المستقبل في Return to Branch.

عند Return Cancel يجب استخدام القيمة الأصلية نفسها.

---

## DestinationSgln

SGLN موقع الفرع المستقبل، ويجب أن يتوافق مع `DestinationGln`.

---

## CurrentGln

الجهة المرتبطة حاليًا بالعبوة كما تعيدها `VerifyProduct`.

تقارن مبدئيًا مع:

```csharp
options.PharmacyGln
```

وتظل قواعد الملكية النهائية لدى EPTTS.

---

# النتائج والاستجابات

## EpttsResult<T>

النتيجة الموحدة التي تعيدها دوال المكتبة.

أهم خصائصها:

```text
IsSuccess
HttpStatusCode
ErrorCode
ErrorMessage
Data
RawRequest
RawResponse
```

يجب فحص `IsSuccess` ثم `Data` قبل قراءة النتيجة الوظيفية.

---

## IsSuccess

توضح هل نجح تنفيذ الطلب تقنيًا واستطاعت المكتبة معالجة الاستجابة.

لا تعني وحدها أن العملية التجارية نجحت نهائيًا.

---

## Data

الاستجابة المنظمة الخاصة بالدالة.

افحص أنها ليست `null` قبل قراءة خصائصها.

---

## RawRequest

النص الخام المرسل في جسم الطلب.

يستخدم للتشخيص والتدقيق، وليس بدل Request Class في منطق العمل.

---

## RawResponse

النص الخام الذي أعاده خادم EPTTS.

يستخدم للدعم والتشخيص، بينما يستخدم ERP خاصية `Data` في منطق العمل المعتاد.

---

## VerifyProductResponse

نتيجة منظمة لعملية التحقق من العبوة.

أهم الخصائص:

```text
Verified
Sgtin
VerifiedAt
Pack
Product
Alerts
```

---

## Verified

توضح هل استطاعت EPTTS التحقق من العبوة.

قد تكون النتيجة:

```text
IsSuccess = true
Verified = false
```

أي أن الطلب نجح تقنيًا، لكن العبوة لم تُتحقق.

---

## SubmissionResponse

النتيجة الأولية لعملية تغير حالة العبوة أو مسارها.

أهم الخصائص:

```text
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
IsAcceptedForProcessing
CanQueryFinalStatus
```

لا تمثل النجاح النهائي للعملية.

---

## MessageStatusResponse

نتيجة الاستعلام عن حالة رسالة سبق إرسالها.

أهم الخصائص:

```text
MessageStatus
IsFinal
IsSuccessful
IsApplicationError
FirstErrorMessage
LogList
```

---

# معرفات الرسائل والعمليات

## InstanceIdentifier

معرف يميز رسالة EPCIS محددة.

تنشئ المكتبة معرفًا جديدًا عند بناء رسالة عملية، ويستخدم للتتبع والدعم.

---

## RequestInstanceIdentifier

الخاصية الموجودة في `SubmissionResponse` التي تمثل معرف الطلب المرتبط بالرسالة المرسلة.

احفظها مع سجل العملية دون تغيير.

---

## MessageId

معرف تعيده EPTTS للرسالة ويستخدم للتتبع والدعم.

لا تستخدمه بدل `StatusQueryIdentifier` أو `ReturnRequestNumber` أو `SGTIN`.

---

## StatusQueryIdentifier

المعرف المستخدم لمتابعة النتيجة النهائية.

يمرر إلى:

```csharp
GetMessageStatusAsync
WaitForFinalStatusAsync
```

يجب حفظه فور قبول العملية في قاعدة بيانات ERP.

---

## ReturnRequestNumber

رقم تجاري ينشئه ERP لعملية المرتجع.

مثال:

```text
RET-UAT-000123
```

عند Return Cancel يجب استخدام الرقم الأصلي نفسه.

```text
ReturnRequestNumber ≠ MessageId
ReturnRequestNumber ≠ RequestInstanceIdentifier
ReturnRequestNumber ≠ StatusQueryIdentifier
```

---

## EventDateTime

الوقت الفعلي لحدوث العملية التجارية.

يستخدم عادة كـ `DateTimeOffset`، ويجب أن تكون ساعة الجهاز أو الخادم مضبوطة بصورة صحيحة.

---

## PatientReference

مرجع داخلي اختياري يربط عملية الصرف بمعاملة داخل ERP.

لا تضع بيانات مريض شخصية مباشرة.

---

## PrescriptionReference

مرجع داخلي اختياري للوصفة أو الروشتة.

لا تضع نص الوصفة الطبية كاملًا.

---

## Quantity

كمية الصرف الجزئي في العملية الحالية.

لا تعني بالضرورة قرصًا واحدًا، ويعتمد معناها على وحدة الصرف في Master Data.

---

## InvoiceNumber

رقم فاتورة أو مستند شحن اختياري في Receiving.

---

# العمليات العامة

## VerifyProduct

عملية قراءة للتحقق من العبوة وقراءة حالتها وملكيتها وتنبيهاتها.

الدالة:

```csharp
VerifyProductAsync
```

لا تغير حالة العبوة.

---

## Receiving from Branch

عملية تسجيل استلام شحنة أو عبوات من فرع إلى الصيدلية.

الدالة:

```csharp
ReceiveFromBranchAsync
```

قد تستخدم SSCC أو قائمة SGTINs.

---

## Full Pack Dispensing

عملية صرف عبوة كاملة.

الدالة:

```csharp
DispenseFullPackAsync
```

الحالة المبدئية المعتادة: `active`.

---

## Partial Dispensing

عملية صرف كمية جزئية من عبوة تدعم هذا النوع من الصرف.

الدالة:

```csharp
DispensePartialAsync
```

الحالات المبدئية المعتادة: `active` أو `partially_dispensed`.

---

## Dispense Cancel

عملية إلغاء صرف عبوة كاملة سابقة.

الدالة:

```csharp
CancelDispensingAsync
```

الحالة المبدئية المعتادة: `dispensed`.

لا تقدمها كإلغاء للصرف الجزئي دون مواصفة معتمدة.

---

## Return to Branch

عملية إرسال عبوة من الصيدلية إلى فرع كمرتجع.

الدالة:

```csharp
ReturnToBranchAsync
```

تحتاج `ReturnRequestNumber` ينشئه ERP ويحفظه.

---

## Return Cancel

عملية إلغاء مرتجع سابق ما زال قابلًا للإلغاء.

الدالة:

```csharp
CancelReturnAsync
```

تستخدم القيم الأصلية نفسها: `SGTIN` و`DestinationGln` و`ReturnRequestNumber`.

---

## Message Status

عملية قراءة لمعرفة حالة رسالة سبق إرسالها.

تستخدم `StatusQueryIdentifier` ولا تنشئ عملية تجارية جديدة.

---

## GetMessageStatusAsync

تنفذ استعلامًا واحدًا عن حالة الرسالة، وتناسب Background Job والمتابعة طويلة المدى.

---

## WaitForFinalStatusAsync

تنفذ Polling تلقائيًا حتى تظهر نتيجة نهائية أو تنتهي مدة الانتظار أو يحدث إلغاء.

انتهاء Polling لا يعني فشل العملية الأصلية.

---

# حالات العبوة

## active

العبوة نشطة ويمكن محاولة تنفيذ عملية مناسبة إذا تحققت بقية القواعد.

---

## in_transit

العبوة في مسار نقل أو شحنة، وقد تكون الحالة المتوقعة قبل Receiving حسب سير العمل.

---

## dispensed

العبوة صُرفت بالكامل، وتكون الحالة المتوقعة قبل Dispense Cancel.

---

## partially_dispensed

تم صرف جزء من العبوة، وقد تسمح بصرف جزئي إضافي حسب قواعد المنتج والكمية المتبقية.

---

## returned

العبوة دخلت في مسار مرتجع أو أصبح المرتجع مسجلًا حسب البيئة والعملية.

---

## return_pending

حالة قد تستخدم للدلالة على مرتجع معلق في بعض البيئات أو السيناريوهات.

اعتمد على القيم التي تعيدها البيئة الفعلية.

---

## IsRecalled

خاصية توضح هل العبوة مستدعاة.

لا تنفذ صرفًا عاديًا على عبوة مستدعاة.

---

## Alerts

قائمة تنبيهات تعيدها `VerifyProduct` ويجب مراجعتها قبل إرسال عملية تغير حالة العبوة.

---

# حالات الرسالة والنتيجة

## HTTP 202: Hypertext Transfer Protocol 202 Accepted

الاسم الكامل:

```text
Hypertext Transfer Protocol 202 Accepted
```

يعني أن الرسالة قُبلت للمعالجة، ولا يعني أن العملية نجحت نهائيًا.

القرار:

```text
Pending
→ حفظ StatusQueryIdentifier
→ متابعة Message Status
```

---

## I001

كود معلومات قد يدل على قبول الرسالة للمعالجة حسب الاستجابة.

لا يعادل النجاح النهائي.

---

## S - Successful

نتيجة نهائية ناجحة للعملية الأصلية.

عندها يمكن تحويل سجل ERP إلى `Successful` وتطبيق أثر النجاح حسب قواعد ERP.

---

## E - Application Error

فشل وظيفي نهائي للعملية الأصلية داخل EPTTS.

قد يكون طلب Message Status نفسه ناجحًا تقنيًا.

القرار: `Failed` مع حفظ `FirstErrorMessage`.

---

## IsFinal

توضح هل وصلت الرسالة إلى حالة نهائية.

لا تعتمد عليها وحدها عند توفر `IsSuccessful` و`IsApplicationError`.

---

## IsSuccessful

توضح أن العملية الأصلية نجحت نهائيًا.

---

## IsApplicationError

توضح أن العملية الأصلية فشلت وظيفيًا.

---

## FirstErrorMessage

أول رسالة خطأ وظيفي مفيدة من نتيجة معالجة الرسالة.

---

## LogList

قائمة بسجلات أو رسائل معالجة EPTTS، وتستخدم للتشخيص.

---

# حالات ERP المحلية

## Draft

سجل محلي تم إنشاؤه ولم يبدأ إرساله بعد.

---

## Submitting

بدأت محاولة إرسال العملية ولم تُحسم بعد، وتستخدم لمنع الإرسال المتزامن.

---

## Pending

قُبلت الرسالة ويوجد معرف متابعة، أو ما زالت الرسالة قيد المعالجة.

---

## Successful

وصلت الرسالة إلى نتيجة نهائية ناجحة.

---

## Failed

وصلت العملية إلى فشل وظيفي نهائي أو رفض نهائي واضح.

لا تستخدمها لمجرد حدوث Timeout غير مؤكد.

---

## NeedsReview

النتيجة غير مؤكدة وتحتاج مراجعة، مثل Timeout أثناء Submission أو فقدان معرف المتابعة.

---

# التشغيل والمتابعة

## Submission

إرسال العملية الأولي إلى EPTTS، وقد يعيد قبولًا للمعالجة دون نتيجة نهائية.

---

## Polling

تنفيذ استعلامات متكررة بفاصل زمني لمعرفة هل وصلت الرسالة إلى نتيجة نهائية.

---

## Polling Interval

المدة بين استعلام حالة والذي يليه.

---

## Maximum Wait Time

أقصى مدة ينتظرها `WaitForFinalStatusAsync`.

انتهاء المدة لا يعني فشل العملية الأصلية.

---

## Background Job

عملية تعمل دوريًا داخل ERP لقراءة العمليات `Pending` واستدعاء `GetMessageStatusAsync` وتحديث النتيجة.

---

## Repository

طبقة داخل ERP لحفظ وقراءة وتحديث سجلات عمليات EPTTS.

ليست جزءًا من المكتبة، ويجب أن ينفذها فريق ERP.

---

## Idempotency

منع تكرار الأثر التجاري عند تكرار الطلب أو إعادة المحاولة.

يجب دعمها محليًا باستخدام قاعدة البيانات وحالات العمليات.

---

## Retry

إعادة محاولة طلب فشل بصورة مؤقتة.

يمكن استخدام Retry محدود لعمليات القراءة، ولا تعاد عمليات تغيير الحالة تلقائيًا قبل حسم نتيجة المحاولة السابقة.

---

## Correlation

ربط سجلات ERP وLogs بمعرفات الرسالة لتتبع العملية كاملة.

استخدم `OperationId` المحلي و`RequestInstanceIdentifier` و`MessageId` و`StatusQueryIdentifier`.

---

## Smoke Test

اختبار محدود بعد النشر للتأكد من أن التطبيق يبدأ وأن الاتصال الأساسي يعمل.

---

## Rollback

الرجوع إلى إصدار سابق عند فشل النشر.

لا تحذف العمليات `Pending` أثناء الرجوع.

---

# الأخطاء والاستثناءات

## EpttsConfigurationException

استثناء يدل على وجود إعدادات غير صحيحة، مثل `BaseUrl` أو `PharmacyGln` أو `Timeout`.

عادة لا يعني أن طلبًا وصل إلى EPTTS.

---

## EpttsValidationException

استثناء يدل على مدخلات غير صحيحة محليًا، مثل `SGTIN` فارغ أو `Quantity` غير صالحة.

---

## EpttsException

نوع عام لأخطاء المكتبة أو التكامل.

ضعه بعد الاستثناءات الأكثر تحديدًا داخل `catch`.

---

## REQUEST_TIMEOUT

انتهت مهلة انتظار الطلب.

لا يثبت أن EPTTS لم تستلم عملية تغير الحالة.

---

## REQUEST_CANCELLED

أُلغي انتظار الطلب محليًا.

لا يعني إلغاء العملية داخل EPTTS.

---

## MESSAGE_STATUS_TIMEOUT

انتهت مدة انتظار النتيجة النهائية، وقد تظل العملية الأصلية `Pending`.

---

## PACK_NOT_FOUND

العبوة غير موجودة أو المعرف غير صحيح أو البيانات تخص بيئة مختلفة.

---

## PACK_NOT_OWNED

العبوة ليست مرتبطة بالصيدلية أو الجهة المطلوبة وفق قواعد EPTTS.

راجع `CurrentGln` و`PharmacyGln`.

---

## No PENDING return found

لم تتمكن EPTTS من العثور على مرتجع معلق مطابق.

راجع `SGTIN` و`DestinationGln` و`ReturnRequestNumber` الأصلية.

---

# مرجع اختيار القيمة الصحيحة

## ما المعرف المستخدم للتحقق من عبوة؟

```text
SGTIN
```

## ما المعرف المستخدم لاستلام حاوية كاملة؟

```text
SSCC
```

## ما المعرف المستخدم لمتابعة الرسالة؟

```text
StatusQueryIdentifier
```

## ما الرقم المستخدم لإلغاء المرتجع؟

```text
ReturnRequestNumber الأصلي
```

## ما معرف الصيدلية الحالية؟

```text
PharmacyGln
```

## ما معرف الفرع المرسل في Receiving؟

```text
SourceGln
```

## ما معرف الفرع المستقبل في Return؟

```text
DestinationGln
```

---

# مرجع القرار السريع

```text
IsSuccess = false
→ فشل تقني أو HTTP
→ راجع ErrorCode وErrorMessage

IsSuccess = true وVerified = false
→ الطلب نجح لكن العبوة لم تُتحقق

HTTP 202 أو I001
→ Pending
→ احفظ StatusQueryIdentifier

IsSuccessful = true
→ Successful

IsApplicationError = true
→ Failed
→ احفظ FirstErrorMessage

Timeout أثناء Submission
→ NeedsReview
→ لا تعِد الإرسال مباشرة

Timeout أثناء Polling
→ Pending
→ استعلم لاحقًا

Cancellation أثناء Polling
→ Pending
→ العملية الأصلية لم تُلغَ
```

---

## خلاصة القاموس

المعرفات الأكثر أهمية:

```text
SGTIN
→ عبوة واحدة

SSCC
→ شحنة أو حاوية

PharmacyGln
→ الصيدلية الحالية

SourceGln
→ الفرع المرسل

DestinationGln
→ الفرع المستقبل

StatusQueryIdentifier
→ متابعة النتيجة النهائية

ReturnRequestNumber
→ رقم المرتجع التجاري الذي ينشئه ERP
```

والقاعدة الأساسية للعمليات غير المتزامنة:

```text
Submit
→ Pending
→ Query Status
→ Successful أو Failed
```
