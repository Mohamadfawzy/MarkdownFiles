# الملحق A: قاموس المصطلحات السريع

> مرجع مختصر للمصطلحات والاختصارات المستخدمة في دليل تكامل `Eptts.Client` مع أنظمة ERP. استخدم هذا الملف عند الحاجة إلى تذكّر معنى مصطلح أو معرفة المعرف الصحيح الذي يجب تمريره إلى إحدى الدوال.

---

## 1. طريقة استخدام القاموس

كل مصطلح يحتوي على:

- تعريف مختصر.
- الاستخدام داخل التكامل.
- ملاحظة أو تحذير مهم عند الحاجة.

للتفاصيل والأمثلة الكاملة، راجع:

```text
02-essential-concepts-ar.md
03-results-and-errors-ar.md
```

---

# المصطلحات العامة

## EPTTS

نظام يستقبل عمليات تتبع العبوات الدوائية ويعالجها.

يستخدم ERP مكتبة `Eptts.Client` لتنفيذ عمليات مثل:

- التحقق من العبوة.
- الاستلام من فرع.
- الصرف الكامل أو الجزئي.
- إلغاء الصرف.
- المرتجع وإلغاء المرتجع.
- متابعة حالة الرسائل.

بعض العمليات تُقبل أولًا ثم تظهر نتيجتها النهائية لاحقًا.

---

## ERP

نظام تخطيط موارد المؤسسة الذي يدمج مكتبة `Eptts.Client`.

ERP مسؤول عن:

- المستندات.
- المخزون.
- قاعدة البيانات.
- المستخدمين والصلاحيات.
- حفظ العمليات المعلقة.
- متابعة النتائج النهائية.

المكتبة لا تعدّل قاعدة بيانات ERP تلقائيًا.

---

## Eptts.Client

مكتبة C# تستخدم لربط ERP بخدمات EPTTS.

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

مثال:

```csharp
using (EpttsClient client =
    new EpttsClient(options))
{
    // Execute an endpoint here.
}
```

إنشاء `EpttsClient` لا يرسل HTTP Request. يبدأ الاتصال عند استدعاء إحدى الدوال.

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

## Endpoint

مسار أو عملية عامة توفرها المكتبة للتعامل مع EPTTS.

أمثلة:

```text
VerifyProductAsync
GetMessageStatusAsync
ReceiveFromBranchAsync
DispenseFullPackAsync
```

---

## API

واجهة برمجية تسمح للتطبيقات بتبادل الطلبات والاستجابات.

تخفي `Eptts.Client` معظم تفاصيل HTTP وJSON عن مطور ERP.

---

## HTTP Request

الطلب الذي ترسله المكتبة إلى EPTTS.

قد يتضمن:

- URL.
- Headers.
- JSON Body.
- بيانات الصيدلية والعملية.

لا ينشئ ERP Headers السرية يدويًا عند استخدام المكتبة.

---

## HTTP Response

الاستجابة التي يعيدها خادم EPTTS للطلب.

تقرأ المكتبة الاستجابة وتحولها إلى `EpttsResult<T>` و`Data` منظمة.

---

## JSON

صيغة نصية تستخدم في تبادل البيانات.

تعرض المكتبة النصوص الخام في:

```text
RawRequest
RawResponse
```

لا يحتاج ERP عادة إلى إنشاء JSON يدويًا.

---

## EPC

اختصار لـ Electronic Product Code.

يستخدم لتعريف عناصر أو مواقع أو وحدات لوجستية في صيغة قابلة للتبادل.

---

## EPC URI

صيغة نصية لمعرف EPC.

أمثلة:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
urn:epc:id:sscc:80026600.000444363
urn:epc:id:sgln:6221388.35823.0
```

يجب تمرير المعرف كاملًا دون حذف البداية.

---

## EPCIS

صيغة أو معيار رسائل يستخدم لتمثيل أحداث التتبع.

تنشئ المكتبة رسائل EPCIS المطلوبة للعمليات. لا يحتاج مطور ERP إلى بنائها يدويًا.

---

# معرفات المنتج والعبوة والشحنة

## GTIN

معرف نوع المنتج التجاري.

يحدد المنتج، لكنه لا يحدد عبوة مادية منفردة.

لا تستخدم `GTIN` وحده عندما تطلب الدالة `SGTIN`.

---

## Serial Number

رقم مسلسل يميز عبوة محددة عن العبوات الأخرى من المنتج نفسه.

يدخل مع هوية المنتج في تكوين `SGTIN`.

---

## SGTIN

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

لا تضع بدلًا منه:

- `GTIN` فقط.
- `SSCC`.
- رقم الصنف الداخلي.
- `ItemID`.
- `ItemRefNo`.

---

## SSCC

معرف حاوية أو شحنة لوجستية.

مثال:

```text
urn:epc:id:sscc:80026600.000444363
```

يستخدم عادة في `Receiving` عند استلام حاوية أو شحنة كاملة.

لا تستخدم `SSCC` داخل عملية صرف عبوة منفردة.

---

## EPC List

قائمة معرفات EPC تمرر إلى بعض العمليات، خصوصًا `Receiving`.

قد تحتوي على:

- `SSCC` لشحنة كاملة.
- قائمة `SGTINs` لعبوات منفردة.

لا ترسل `SSCC` وجميع محتوياته في العملية نفسها إلا إذا كانت المواصفة تطلب ذلك صراحة.

---

## DataMatrix

رمز ثنائي الأبعاد على العبوة قد يحتوي على بيانات مثل:

- `GTIN`.
- الرقم المسلسل.
- التشغيلة.
- تاريخ الانتهاء.

نص DataMatrix الخام ليس بالضرورة `SGTIN` جاهزًا. يجب أن يحلله ERP أو المكوّن المعتمد قبل تمرير المعرف إلى المكتبة.

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

كائن داخل `VerifyProductResponse` يحتوي على بيانات المنتج العامة، مثل الاسم والشكل والتركيز وبيانات الشركة المصنعة عند توفرها.

لا تستبدل Master Data الخاصة بـ ERP تلقائيًا دون سياسة مزامنة معتمدة.

---

# معرفات الجهات والمواقع

## GLN

معرف جهة أو موقع تجاري.

مثال:

```text
6221388358239
```

قد يستخدم كـ:

```text
PharmacyGln
SourceGln
DestinationGln
CurrentGln
```

يجب فهم وظيفة كل قيمة وعدم استبدال إحداها بالأخرى.

---

## SGLN

معرف موقع بصيغة EPC URI.

مثال:

```text
urn:epc:id:sgln:6221388.35823.0
```

قد يستخدم كـ:

```text
PharmacySgln
SourceSgln
DestinationSgln
```

لا تنشئ `SGLN` يدويًا من `GLN` دون قواعد معتمدة.

---

## PharmacyGln

GLN الصيدلية التي ينفذ ERP العملية باسمها.

يُخزن داخل `EpttsOptions`:

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

GLN الجهة أو الفرع المرسل في عملية `Receiving`.

```text
SourceGln
→ الفرع المرسل

PharmacyGln
→ الصيدلية المستلمة
```

---

## SourceSgln

SGLN موقع الفرع المرسل في `Receiving`.

يجب أن يتوافق مع `SourceGln`.

---

## DestinationGln

GLN الفرع المستقبل في عملية `Return to Branch`.

عند `Return Cancel` يجب استخدام `DestinationGln` نفسه الخاص بالمرتجع الأصلي.

---

## DestinationSgln

SGLN موقع الفرع المستقبل في `Return to Branch`.

يجب أن يتوافق مع `DestinationGln`.

---

## CurrentGln

الجهة المرتبطة حاليًا بالعبوة كما تعيدها `VerifyProduct`.

للتأكد مبدئيًا من أن العبوة لدى الصيدلية الحالية، تقارن مع:

```csharp
options.PharmacyGln
```

المقارنة المحلية مساعدة، بينما تطبق EPTTS القواعد النهائية.

---

# إعدادات الاتصال

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

لا تحفظه في:

- Source Code.
- GitHub.
- ملفات Markdown.
- Logs.
- Screenshots.
- RawRequest.
- RawResponse.

استخدم مصدر أسرار آمنًا.

---

## Timeout

مهلة انتظار HTTP Request واحد.

مثال:

```csharp
Timeout =
    TimeSpan.FromSeconds(60);
```

لا يمثل مدة المعالجة النهائية للرسالة بعد قبولها.

---

## Staging

بيئة اختبار أو تكامل تستخدم قبل Production.

يجب استخدام:

- `BaseUrl` خاص بـ Staging.
- `IntegratorKey` خاص بـ Staging.
- صيدلية وعبوات اختبار.

---

## Production

بيئة التشغيل الفعلية.

لا تنتقل إليها قبل إتمام اختبارات Staging وUAT واعتماد خطة النشر والرجوع.

---

# Requests وResponses

## Request

Class تحتوي على مدخلات العملية التي سترسلها المكتبة.

أمثلة:

```text
ReceivingRequest
FullDispensingRequest
PartialDispensingRequest
DispenseCancelRequest
ReturnRequest
ReturnCancelRequest
```

---

## Response

Class منظمة تمثل البيانات التي أعادتها المكتبة.

أمثلة:

```text
VerifyProductResponse
SubmissionResponse
MessageStatusResponse
```

---

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

نوعها يعتمد على `T` في:

```csharp
EpttsResult<T>
```

افحص أنها ليست `null` قبل قراءة خصائصها.

---

## RawRequest

النص الخام المرسل في Body الطلب.

يستخدم للتشخيص والتدقيق، وليس بدل Request Class في منطق العمل.

---

## RawResponse

النص الخام الذي أعاده خادم EPTTS.

يستخدم للدعم والتشخيص. استخدم `Data` في منطق ERP المعتاد.

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

قد تكون:

```text
IsSuccess = true
Verified = false
```

ومعناها أن طلب HTTP نجح، لكن العبوة لم تُتحقق.

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

تنشئ المكتبة معرفًا جديدًا عند بناء رسالة عملية.

يستخدم للتتبع والدعم واستعادة سياق الطلب عند الحاجة.

---

## RequestInstanceIdentifier

الخاصية الموجودة في `SubmissionResponse` التي تمثل معرف الطلب المرتبط بالرسالة المرسلة.

احفظها مع سجل العملية دون تغيير.

---

## MessageId

معرف تعيده EPTTS للرسالة.

يستخدم للتتبع والدعم.

لا تستخدمه بدل:

- `StatusQueryIdentifier`.
- `ReturnRequestNumber`.
- `SGTIN`.

---

## StatusQueryIdentifier

المعرف المستخدم لمتابعة النتيجة النهائية.

يمرر إلى:

```csharp
GetMessageStatusAsync
```

أو:

```csharp
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

عند `Return Cancel` يجب استخدام الرقم الأصلي نفسه.

```text
ReturnRequestNumber ≠ MessageId
ReturnRequestNumber ≠ RequestInstanceIdentifier
ReturnRequestNumber ≠ StatusQueryIdentifier
```

---

## EventDateTime

الوقت الفعلي لحدوث العملية التجارية.

يستخدم عادة كـ `DateTimeOffset`:

```csharp
EventDateTime =
    DateTimeOffset.Now;
```

يجب أن تكون ساعة الجهاز أو الخادم مضبوطة بصورة صحيحة.

---

## PatientReference

مرجع داخلي اختياري يربط عملية الصرف بمعاملة داخل ERP.

مثال:

```text
ERP-SALE-1001
```

لا تضع بيانات مريض شخصية مباشرة.

---

## PrescriptionReference

مرجع داخلي اختياري للوصفة أو الروشتة.

مثال:

```text
ERP-RX-1001
```

لا تضع نص الوصفة الطبية كاملًا.

---

## Quantity

كمية الصرف الجزئي في العملية الحالية.

لا تعني بالضرورة قرصًا واحدًا. يعتمد معناها على وحدة الصرف المعتمدة في Master Data.

---

## InvoiceNumber

رقم فاتورة أو مستند شحن اختياري في عملية `Receiving`.

يمثل مرجع المستند داخل ERP أو الجهة المرسلة حسب تصميم النظام.

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

قد تستخدم `SSCC` أو قائمة `SGTINs`.

---

## Full Pack Dispensing

عملية صرف عبوة كاملة.

الدالة:

```csharp
DispenseFullPackAsync
```

الحالة المبدئية المعتادة:

```text
active
```

---

## Partial Dispensing

عملية صرف كمية جزئية من عبوة تدعم هذا النوع من الصرف.

الدالة:

```csharp
DispensePartialAsync
```

الحالات المبدئية المعتادة:

```text
active
partially_dispensed
```

---

## Dispense Cancel

عملية إلغاء صرف عبوة كاملة سابقة.

الدالة:

```csharp
CancelDispensingAsync
```

الحالة المبدئية المعتادة:

```text
dispensed
```

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

تستخدم القيم الأصلية نفسها:

```text
SGTIN
DestinationGln
ReturnRequestNumber
```

---

## Message Status

عملية قراءة لمعرفة حالة رسالة سبق إرسالها.

تستخدم `StatusQueryIdentifier` ولا تنشئ عملية تجارية جديدة.

---

## GetMessageStatusAsync

تنفذ استعلامًا واحدًا عن حالة الرسالة.

مناسبة لـ Background Job والمتابعة طويلة المدى.

---

## WaitForFinalStatusAsync

تنفذ Polling تلقائيًا حتى تظهر نتيجة نهائية أو تنتهي مدة الانتظار أو يحدث إلغاء.

مناسبة للمحاكي والاختبارات والانتظار القصير.

انتهاء Polling لا يعني فشل العملية الأصلية.

---

# حالات العبوة

## active

العبوة نشطة ويمكن محاولة تنفيذ عملية مناسبة عليها إذا تحققت بقية القواعد.

تستخدم عادة قبل:

- الصرف الكامل.
- أول صرف جزئي.
- المرتجع.

---

## in_transit

العبوة في مسار نقل أو شحنة.

قد تكون الحالة المتوقعة قبل `Receiving` حسب سير العمل.

---

## dispensed

العبوة صُرفت بالكامل.

تكون الحالة المتوقعة قبل `Dispense Cancel`.

---

## partially_dispensed

تم صرف جزء من العبوة.

قد تسمح بصرف جزئي إضافي حسب قواعد المنتج والكمية المتبقية.

---

## returned

العبوة دخلت في مسار مرتجع أو أصبح المرتجع مسجلًا حسب البيئة والعملية.

قد تكون مرتبطة بإمكانية `Return Cancel`.

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

قائمة تنبيهات تعيدها `VerifyProduct`.

يجب مراجعتها قبل إرسال عملية تغير حالة العبوة.

---

# حالات الرسالة والنتيجة

## HTTP 202

يعني أن الرسالة قُبلت للمعالجة.

لا يعني أن العملية نجحت نهائيًا.

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

عندها يمكن تحويل سجل ERP إلى:

```text
Successful
```

وتطبيق أثر النجاح حسب قواعد ERP.

---

## E - Application Error

فشل وظيفي نهائي للعملية الأصلية داخل EPTTS.

قد يكون طلب `Message Status` نفسه ناجحًا تقنيًا.

القرار:

```text
Failed
```

مع حفظ `FirstErrorMessage`.

---

## IsFinal

توضح هل وصلت الرسالة إلى حالة نهائية.

لا تعتمد عليها وحدها إذا كانت المكتبة توفر `IsSuccessful` و`IsApplicationError`.

---

## IsSuccessful

توضح أن العملية الأصلية نجحت نهائيًا.

---

## IsApplicationError

توضح أن العملية الأصلية فشلت وظيفيًا.

---

## FirstErrorMessage

أول رسالة خطأ وظيفي مفيدة من نتيجة معالجة الرسالة.

احفظها واعرض نسخة مناسبة للمستخدم أو الدعم.

---

## LogList

قائمة بسجلات أو رسائل معالجة EPTTS.

تستخدم للتشخيص، ولا تستبدل الخصائص المنظمة الخاصة بالقرار النهائي.

---

# حالات ERP المحلية

## Draft

سجل محلي تم إنشاؤه ولم يبدأ إرساله بعد.

---

## Submitting

بدأت محاولة إرسال العملية ولم تُحسم بعد.

تستخدم لمنع إرسال متزامن من أكثر من شاشة أو Process.

---

## Pending

قُبلت الرسالة ويوجد معرف متابعة، أو ما زالت الرسالة قيد المعالجة.

لا تطبق أثر النجاح النهائي بعد.

---

## Successful

وصلت الرسالة إلى نتيجة نهائية ناجحة.

---

## Failed

وصلت العملية إلى فشل وظيفي نهائي أو رفض نهائي واضح.

لا تستخدمها لمجرد حدوث Timeout غير مؤكد.

---

## NeedsReview

النتيجة غير مؤكدة وتحتاج مراجعة.

أمثلة:

- Timeout أثناء Submission.
- Cancellation أثناء إرسال عملية تغير الحالة.
- قبول غير واضح دون معرف متابعة.
- تعطل التخزين بعد استجابة EPTTS.

---

# التشغيل والمتابعة

## Submission

إرسال العملية الأولي إلى EPTTS.

قد يعيد قبولًا للمعالجة دون نتيجة نهائية.

---

## Polling

تنفيذ استعلامات متكررة بفاصل زمني لمعرفة هل وصلت الرسالة إلى نتيجة نهائية.

يستخدم في `WaitForFinalStatusAsync` أو داخل Background Job.

---

## Polling Interval

المدة بين استعلام حالة والذي يليه.

مثال:

```text
3 seconds
```

لا تستخدم قيمة صغيرة جدًا دون حاجة.

---

## Maximum Wait Time

أقصى مدة ينتظرها استدعاء `WaitForFinalStatusAsync`.

انتهاء المدة لا يعني فشل العملية الأصلية.

---

## Background Job

عملية تعمل دوريًا داخل ERP لقراءة العمليات `Pending` واستدعاء `GetMessageStatusAsync` وتحديث النتيجة.

لا يجب أن يعتمد نجاح التكامل على بقاء شاشة المستخدم مفتوحة.

---

## Repository

طبقة داخل ERP لحفظ وقراءة وتحديث سجلات عمليات EPTTS.

ليست جزءًا من المكتبة، ويجب أن ينفذها فريق ERP.

---

## Idempotency

منع تكرار الأثر التجاري عند إعادة المحاولة أو تكرار الطلب.

في هذا التكامل يجب منع العمليات المتعارضة محليًا باستخدام قاعدة البيانات وحالات `Submitting` و`Pending` و`NeedsReview`.

---

## Retry

إعادة محاولة طلب فشل مؤقتًا.

يمكن استخدام Retry محدود لعمليات القراءة.

لا تعِد عمليات تغيير الحالة تلقائيًا قبل التحقق من نتيجة المحاولة السابقة.

---

## Correlation

ربط سجلات ERP وLogs بمعرفات الرسالة لتتبع العملية كاملة.

استخدم:

```text
OperationId المحلي
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
```

---

## UAT

اختبار قبول المستخدم.

يُنفذ في بيئة اختبار ببيانات معتمدة للتأكد من أن السيناريو التجاري يعمل من البداية حتى النتيجة النهائية.

---

## Smoke Test

اختبار محدود بعد النشر للتأكد من أن التطبيق يبدأ وأن الاتصال الأساسي يعمل دون تنفيذ سيناريوهات واسعة.

---

## Rollback

الرجوع إلى إصدار سابق من التطبيق أو المكتبة عند فشل النشر.

لا تحذف العمليات `Pending` أثناء الرجوع.

---

# الأخطاء والاستثناءات

## EpttsConfigurationException

استثناء يدل على وجود إعدادات غير صحيحة، مثل:

- `BaseUrl` غير صالح.
- `IntegratorKey` فارغ.
- `PharmacyGln` غير صحيح.
- `PharmacySgln` غير صحيح.
- `Timeout` غير صالح.

عادة لا يعني أن طلبًا وصل إلى EPTTS.

---

## EpttsValidationException

استثناء يدل على وجود مدخلات عملية غير صحيحة محليًا، مثل:

- `SGTIN` فارغ.
- `Quantity` غير صالحة.
- قائمة EPC فارغة.
- `StatusQueryIdentifier` فارغ.

---

## EpttsException

نوع عام لأخطاء المكتبة أو التكامل.

ضعه بعد الاستثناءات الأكثر تحديدًا داخل `catch`.

---

## REQUEST_TIMEOUT

انتهت مهلة انتظار HTTP Request.

لا يثبت أن EPTTS لم تستلم عملية تغير الحالة.

عمليات القراءة يمكن إعادة محاولتها وفق سياسة محدودة، بينما عمليات التغيير تحتاج مراجعة الرسالة السابقة أولًا.

---

## REQUEST_CANCELLED

أُلغي انتظار الطلب محليًا.

لا يعني إلغاء العملية داخل EPTTS.

---

## MESSAGE_STATUS_TIMEOUT

انتهت المدة المحددة لانتظار النتيجة النهائية.

العملية الأصلية قد تظل `Pending`.

---

## PACK_NOT_FOUND

العبوة غير موجودة أو المعرف غير صحيح أو البيانات تخص بيئة مختلفة.

راجع `SGTIN` والبيئة و`RawResponse`.

---

## PACK_NOT_OWNED

العبوة ليست مرتبطة بالجهة أو الصيدلية المطلوبة وفق قواعد EPTTS.

راجع `CurrentGln` و`PharmacyGln`.

---

## No PENDING return found

لم تتمكن EPTTS من العثور على مرتجع معلق مطابق.

راجع:

```text
SGTIN الأصلي
DestinationGln الأصلي
ReturnRequestNumber الأصلي
```

ولا تستخدم `MessageId` بدل رقم المرتجع.

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
