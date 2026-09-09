# دليل استخدام ModernSoft EPTTS Client داخل نظام ERP

> دليل عملي لمطوري ERP لدمج عمليات التتبع الدوائي `EPTTS` باستخدام مكتبة `ModernSoft.Eptts.Client`.

---

## المحتويات

1. [نطاق المكتبة](#1-نطاق-المكتبة)
2. [المتطلبات](#2-المتطلبات)
3. [الملفات المطلوبة](#3-الملفات-المطلوبة)
4. [إضافة المكتبة إلى مشروع ERP](#4-إضافة-المكتبة-إلى-مشروع-erp)
5. [مصطلحات أساسية](#5-مصطلحات-أساسية)
6. [إعداد الاتصال](#6-إعداد-الاتصال)
7. [إنشاء EpttsClient](#7-إنشاء-epttsclient)
8. [فهم EpttsResult](#8-فهم-epttsresult)
9. [معالجة الاستثناءات](#9-معالجة-الاستثناءات)
10. [الترتيب الصحيح لتنفيذ العمليات](#10-الترتيب-الصحيح-لتنفيذ-العمليات)
11. [VerifyProduct](#11-verifyproduct)
12. [MsgStatusQuery](#12-msgstatusquery)
13. [Receiving from Branch](#13-receiving-from-branch)
14. [Full Pack Dispensing](#14-full-pack-dispensing)
15. [Partial Dispensing](#15-partial-dispensing)
16. [Dispense Cancel](#16-dispense-cancel)
17. [Return Request to Branch](#17-return-request-to-branch)
18. [Return Cancel](#18-return-cancel)
19. [حفظ العمليات في قاعدة بيانات ERP](#19-حفظ-العمليات-في-قاعدة-بيانات-erp)
20. [منع تكرار العمليات](#20-منع-تكرار-العمليات)
21. [التعامل مع Timeout وCancellation](#21-التعامل-مع-timeout-وcancellation)
22. [الأخطاء الشائعة](#22-الأخطاء-الشائعة)
23. [قائمة الدوال العامة](#23-قائمة-الدوال-العامة)
24. [قائمة مراجعة قبل الإنتاج](#24-قائمة-مراجعة-قبل-الإنتاج)

---

## 1. نطاق المكتبة

مكتبة `ModernSoft.Eptts.Client` هي طبقة جاهزة بين نظام ERP وواجهات `EPTTS`.

يمرر نظام ERP بيانات واضحة إلى دوال المكتبة، وتتولى المكتبة:

- إنشاء طلبات HTTP.
- إضافة Headers المطلوبة.
- إنشاء JSON المطلوب.
- إنشاء رسائل EPCIS.
- إرسال الطلبات إلى EPTTS.
- تحويل الاستجابات إلى Classes واضحة.
- إعادة نتيجة موحدة من نوع `EpttsResult<T>`.

### العمليات المدعومة

- التحقق من عبوة: `VerifyProductAsync`.
- استلام شحنة من فرع: `ReceiveFromBranchAsync`.
- صرف عبوة كاملة: `DispenseFullPackAsync`.
- صرف كمية جزئية: `DispensePartialAsync`.
- إلغاء صرف عبوة كاملة: `CancelDispensingAsync`.
- إرسال مرتجع إلى فرع: `ReturnToBranchAsync`.
- إلغاء مرتجع معلق: `CancelReturnAsync`.
- متابعة نتيجة رسالة: `GetMessageStatusAsync`.

### ما لا تقوم به المكتبة

المكتبة لا تقوم بما يلي:

- لا تعرض رسائل للمستخدم.
- لا تتصل بقاعدة بيانات ERP.
- لا تنشئ فاتورة أو مرتجعًا داخل ERP.
- لا تخصم أو تضيف كمية إلى المخزون.
- لا تقرر اعتماد مستند ERP.
- لا تحفظ `IntegratorKey`.

نظام ERP مسؤول عن واجهة المستخدم، وحفظ العمليات، وتحديث الفواتير والمخزون بعد ظهور النتيجة النهائية.

---

## 2. المتطلبات

- مشروع ERP يستهدف `.NET Framework 4.8`.
- مكتبة `ModernSoft.Eptts.Client` تستهدف `.NET Standard 2.0`.
- توفر بيانات الاتصال الخاصة بكل صيدلية:
  - `BaseUrl`
  - `IntegratorKey`
  - `PharmacyGln`
  - `PharmacySgln`
- مزامنة وقت جهاز أو خادم ERP مع خدمة الوقت في Windows.
- استخدام بيئة `Staging` أثناء الاختبارات.

---

## 3. الملفات المطلوبة

أضف الملفات التالية إلى مجلد ثابت داخل مشروع ERP، مثل:

```text
Libraries/Eptts/
├── ModernSoft.Eptts.Client.dll
├── ModernSoft.Eptts.Client.xml
└── Newtonsoft.Json.dll
```

وظيفة الملفات:

- `ModernSoft.Eptts.Client.dll`: المكتبة الأساسية.
- `ModernSoft.Eptts.Client.xml`: يعرض شرح الفئات والدوال داخل `IntelliSense`.
- `Newtonsoft.Json.dll`: Dependency تستخدمها المكتبة لمعالجة JSON.

> يجب توزيع ملف XML بجانب ملف DLL حتى تظهر تعليقات الاستخدام لمطوري ERP داخل Visual Studio.

---

## 4. إضافة المكتبة إلى مشروع ERP

داخل Visual Studio:

```text
References
→ Add Reference
→ Browse
→ اختر ModernSoft.Eptts.Client.dll
```

بعد إضافة Reference، استخدم Namespaces التالية حسب الحاجة:

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using ModernSoft.Eptts.Client.Clients;
using ModernSoft.Eptts.Client.Configuration;
using ModernSoft.Eptts.Client.Exceptions;
using ModernSoft.Eptts.Client.Requests;
using ModernSoft.Eptts.Client.Responses;
```

إذا ظهر تعارض في `Newtonsoft.Json`، يجب توحيد الإصدار المستخدم داخل ERP مع الإصدار المصاحب للمكتبة.

---

## 5. مصطلحات أساسية

### SGTIN

معرف عبوة دوائية مسلسل، مثل:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
```

يستخدم في عمليات:

- VerifyProduct.
- Full Dispensing.
- Partial Dispensing.
- Dispense Cancel.
- Return.
- Return Cancel.

لا تضع `GTIN` أو `SSCC` أو `DataMatrix` داخل خاصية `Sgtin`.

### SSCC

معرف حاوية أو شحنة لوجستية، مثل:

```text
urn:epc:id:sscc:80026600.000444363
```

يستخدم عادة في `Receiving from Branch` عند استلام الحاوية كاملة.

### GLN

معرف جهة من 13 رقمًا، مثل:

```text
6221388358239
```

### SGLN

معرف موقع بصيغة EPC، مثل:

```text
urn:epc:id:sgln:6221388.35823.0
```

### InstanceIdentifier

معرف تنشئه المكتبة لكل رسالة EPCIS، ويستخدم لاحقًا لمتابعة حالة الرسالة.

### MessageId

معرف تعيده EPTTS عند قبول الرسالة.

### ReturnRequestNumber

رقم تجاري ينشئه ERP لعملية المرتجع.

```text
ReturnRequestNumber ≠ MessageId ≠ InstanceIdentifier
```

يجب استخدام نفس `ReturnRequestNumber` عند إلغاء المرتجع.

---

## 6. إعداد الاتصال

أنشئ `EpttsOptions` من إعدادات الصيدلية الحالية:

```csharp
var options = new EpttsOptions
{
    BaseUrl = settings.EpttsBaseUrl,
    IntegratorKey = settings.IntegratorKey,
    PharmacyGln = currentBranch.PharmacyGln,
    PharmacySgln = currentBranch.PharmacySgln,
    Timeout = TimeSpan.FromSeconds(60)
};
```

### BaseUrl

ضع عنوان البيئة فقط دون Endpoint:

```text
https://masar-api.v2.daf-holding.com
```

لا تضع:

```text
https://masar-api.v2.daf-holding.com/masar-service/api/v1/VerifyProduct
```

### IntegratorKey

- قيمة سرية خاصة بالبيئة.
- لا تكتبها داخل Source Code.
- لا تعرضها في الشاشة.
- لا تسجلها في Logs.
- لا تضفها إلى `RawRequest` أو `RawResponse`.

### PharmacyGln

GLN الصيدلية التي ينفذ ERP العملية باسمها، ويجب أن يتكون من 13 رقمًا.

### PharmacySgln

SGLN الخاص بموقع الصيدلية.

### Timeout

مهلة طلب HTTP الواحد. لا تمثل مدة معالجة الرسالة داخل EPTTS.

---

## 7. إنشاء EpttsClient

الاستخدام الأساسي:

```csharp
using (var client = new EpttsClient(options))
{
    // استدعاء عمليات EPTTS هنا
}
```

يمكن استخدام Client واحد لعدة عمليات متتالية داخل نفس السياق:

```csharp
using (var client = new EpttsClient(options))
{
    var verification = await client.VerifyProductAsync(sgtin);

    // بعد التحقق، يمكن تنفيذ العملية المناسبة.
}
```

لا تستخدم Client بعد استدعاء `Dispose()`.

---

## 8. فهم EpttsResult

جميع دوال المكتبة تعيد نتيجة C# منظمة:

```csharp
EpttsResult<T>
```

مثال:

```csharp
EpttsResult<VerifyProductResponse> result =
    await client.VerifyProductAsync(sgtin);
```

### الخصائص الأساسية

- `IsSuccess`: نجاح طلب HTTP وتحويل الاستجابة.
- `HttpStatusCode`: كود HTTP، وقد يكون `null` إذا لم يصل Response.
- `ErrorCode`: كود الفشل التقني أو الوظيفي.
- `ErrorMessage`: رسالة الخطأ.
- `Data`: Class تحتوي على نتيجة العملية.
- `RawRequest`: JSON المرسل إلى EPTTS دون Headers السرية.
- `RawResponse`: الاستجابة الأصلية من EPTTS.

### نمط التعامل الأساسي

```csharp
var result = await client.VerifyProductAsync(sgtin);

if (!result.IsSuccess)
{
    ShowError(result.ErrorCode + " - " + result.ErrorMessage);

    SaveTechnicalLog(
        result.HttpStatusCode,
        result.RawRequest,
        result.RawResponse);

    return;
}

var response = result.Data;
```

### هل النتيجة Class أم JSON؟

الاستخدام الطبيعي يعتمد على `Data`، وهي Class منظمة:

```csharp
string status = result.Data.Pack.Status;
```

لا تحلل `RawResponse` يدويًا في منطق العمل العادي. استخدم `RawRequest` و`RawResponse` فقط للتشخيص والتدقيق والدعم الفني.

---

## 9. معالجة الاستثناءات

أخطاء الإعدادات والمدخلات غير الصحيحة تُرمى كاستثناءات قبل إرسال الطلب.

استخدم الترتيب التالي:

```csharp
try
{
    using (var client = new EpttsClient(options))
    {
        // استدعاء المكتبة
    }
}
catch (EpttsConfigurationException ex)
{
    ShowError("خطأ في إعدادات EPTTS: " + ex.Message);
}
catch (EpttsValidationException ex)
{
    ShowError("بيانات العملية غير صحيحة: " + ex.Message);
}
catch (EpttsException ex)
{
    ShowError("خطأ من مكتبة EPTTS: " + ex.Message);
}
catch (Exception ex)
{
    LogUnexpectedError(ex);
    ShowError("حدث خطأ غير متوقع.");
}
```

### الفرق بين Exception وEpttsResult.Failure

- إعداد ناقص أو SGTIN غير صحيح محليًا: `Exception`.
- HTTP 401 أو 403 أو 500 أو Timeout: `EpttsResult<T>` بقيمة `IsSuccess = false`.
- `E - Application Error`: طلب MsgStatusQuery ناجح، لكن العملية التجارية الأصلية فشلت.

---

## 10. الترتيب الصحيح لتنفيذ العمليات

عمليات EPTTS غير المتزامنة تمر بأربع مراحل:

```text
1. VerifyProduct عند الحاجة
2. إرسال العملية
3. حفظ العملية في ERP بحالة Pending
4. تنفيذ MsgStatusQuery حتى الوصول إلى نتيجة نهائية
```

التسلسل العام:

```csharp
// 1. التحقق من العبوة
var verification = await client.VerifyProductAsync(sgtin);

// 2. إرسال العملية
var submission = await client.SomeOperationAsync(request);

// 3. حفظ StatusQueryIdentifier وحالة Pending
SavePendingOperation(submission.Data.StatusQueryIdentifier);

// 4. متابعة النتيجة النهائية
var finalStatus = await client.GetMessageStatusAsync(
    submission.Data.StatusQueryIdentifier);
```

### معنى HTTP 202

```text
HTTP 202 + I001 = تم قبول الرسالة للمعالجة فقط
```

لا يعني:

```text
Receiving نجح
Dispensing نجح
Return نجح
```

النجاح النهائي الوحيد هو:

```text
S - Successful
```

---

## 11. VerifyProduct

عملية قراءة لا تغيّر حالة العبوة.

### الاستخدام

```csharp
var result = await client.VerifyProductAsync(sgtin);

if (!result.IsSuccess)
{
    ShowError(result.ErrorMessage);
    return;
}

if (result.Data == null ||
    !result.Data.Verified ||
    result.Data.Pack == null)
{
    ShowError("تعذر التحقق من العبوة.");
    return;
}

string packStatus = result.Data.Pack.Status;
string currentGln = result.Data.Pack.CurrentGln;
bool isRecalled = result.Data.Pack.IsRecalled;
```

### القيم المهمة

```csharp
result.Data.Verified
result.Data.Sgtin
result.Data.Pack.Status
result.Data.Pack.CurrentGln
result.Data.Pack.IsRecalled
result.Data.Pack.BatchNumber
result.Data.Pack.ExpiryDate
result.Data.Product.Name
result.Data.Alerts
```

### تنبيه

قد تكون:

```csharp
result.IsSuccess == true
```

وفي الوقت نفسه:

```csharp
result.Data.Verified == false
```

هذا يعني أن طلب HTTP نجح، لكن EPTTS لم يتمكن من التحقق من العبوة. افحص القيمتين دائمًا.

### دالة مساعدة مقترحة داخل ERP

```csharp
private static bool IsOwnedByCurrentPharmacy(
    VerifyProductResponse response,
    string pharmacyGln)
{
    return response != null &&
           response.Pack != null &&
           string.Equals(
               response.Pack.CurrentGln,
               pharmacyGln,
               StringComparison.Ordinal);
}
```

---

## 12. MsgStatusQuery

تستخدم لمتابعة النتيجة النهائية للعمليات غير المتزامنة.

### الاستعلام مرة واحدة

```csharp
var result = await client.GetMessageStatusAsync(
    statusQueryIdentifier);

if (!result.IsSuccess)
{
    ShowError(result.ErrorMessage);
    return;
}

if (result.Data == null)
{
    KeepOperationPending();
    return;
}

if (result.Data.IsSuccessful)
{
    MarkOperationSuccessful();
}
else if (result.Data.IsApplicationError)
{
    MarkOperationFailed(
        result.Data.FirstErrorMessage);
}
else
{
    KeepOperationPending();
}
```

### الخصائص المهمة

```csharp
result.Data.MessageStatus
result.Data.IsFinal
result.Data.IsSuccessful
result.Data.IsApplicationError
result.Data.FirstErrorMessage
result.Data.LogList
```

### الانتظار التلقائي

مناسب للاختبارات أو الشاشات التي تسمح بانتظار قصير:

```csharp
var result = await client.WaitForFinalStatusAsync(
    statusQueryIdentifier,
    pollingInterval: TimeSpan.FromSeconds(3),
    maximumWaitTime: TimeSpan.FromSeconds(30));
```

داخل ERP يفضّل استخدام `GetMessageStatusAsync` من خلال Job أو Timer لمعالجة العمليات `Pending`.

---

## 13. Receiving from Branch

تستخدم لاستلام SSCC كامل أو قائمة عبوات منفردة قادمة من فرع.

### الشروط قبل الإرسال

- الشحنة موجهة إلى الصيدلية الحالية.
- الحالة السابقة غالبًا `in_transit`.
- `SourceGln` و`SourceSgln` يخصان الفرع المرسل.
- إذا كانت الشحنة داخل SSCC، أرسل SSCC فقط.
- لا ترسل SSCC ومحتوياته من SGTINs في الطلب نفسه إلا إذا كانت المواصفة تطلب ذلك صراحة.

### استلام SSCC كامل

```csharp
var request = new ReceivingRequest
{
    SourceGln = sourceBranch.Gln,
    SourceSgln = sourceBranch.Sgln,
    EpcList = new List<string>
    {
        shipmentSscc
    },
    InvoiceNumber = invoiceNumber,
    EventDateTime = DateTimeOffset.Now
};

var result = await client.ReceiveFromBranchAsync(request);
```

### استلام عبوات منفردة

```csharp
var request = new ReceivingRequest
{
    SourceGln = sourceBranch.Gln,
    SourceSgln = sourceBranch.Sgln,
    EpcList = new List<string>
    {
        firstSgtin,
        secondSgtin
    },
    EventDateTime = DateTimeOffset.Now
};
```

### بعد إرسال العملية

```csharp
if (!result.IsSuccess ||
    result.Data == null ||
    !result.Data.CanQueryFinalStatus)
{
    HandleSubmissionFailure(result);
    return;
}

SavePendingReceiving(
    result.Data.StatusQueryIdentifier,
    result.RawRequest,
    result.RawResponse);
```

بعد `S - Successful` نفذ `VerifyProduct` على إحدى العبوات. المتوقع:

```text
Status = active
CurrentGln = PharmacyGln
```

---

## 14. Full Pack Dispensing

تستخدم لصرف عبوة كاملة.

### الشروط قبل الإرسال

- `Verified = true`.
- `Pack.Status = active`.
- `Pack.CurrentGln = PharmacyGln`.
- `Pack.IsRecalled = false`.
- المنتج يسمح بالصرف الكامل.
- قناة الصرف مدعومة.

### الاستخدام

```csharp
var request = new FullDispensingRequest
{
    Sgtin = sgtin,
    PatientReference = patientTransactionReference,
    PrescriptionReference = prescriptionReference,
    EventDateTime = DateTimeOffset.Now
};

var result = await client.DispenseFullPackAsync(request);
```

### بيانات المريض

استخدم داخل `PatientReference` مرجعًا داخليًا فقط، مثل:

```text
PATIENT-TRANSACTION-1001
```

لا ترسل:

- اسم المريض.
- الرقم القومي.
- رقم الهاتف.
- العنوان.
- أي بيانات شخصية مباشرة.

بعد `S - Successful` نفذ `VerifyProduct`. المتوقع:

```text
Status = dispensed
```

---

## 15. Partial Dispensing

تستخدم لصرف كمية من عبوة تدعم البيع الجزئي.

### الشروط قبل الإرسال

- الحالة `active` أو `partially_dispensed`.
- العبوة مملوكة للصيدلية.
- العبوة غير مستدعاة.
- المنتج يسمح بالصرف الجزئي.
- الكمية أكبر من صفر.

### الاستخدام

```csharp
var request = new PartialDispensingRequest
{
    Sgtin = sgtin,
    Quantity = quantityToDispense,
    PatientReference = patientTransactionReference,
    PrescriptionReference = prescriptionReference,
    EventDateTime = DateTimeOffset.Now
};

var result = await client.DispensePartialAsync(request);
```

### معنى Quantity

`Quantity` هي كمية العملية الحالية، وليست إجمالي محتوى العبوة.

```csharp
Quantity = 1;
```

قد تعني وحدة صرف واحدة مثل قرص أو شريط أو زجاجة وفق `Master Data` الخاصة بالمنتج. لا تخمّن معنى الوحدة من اسم المنتج.

بعد `S - Successful` نفذ `VerifyProduct`. المتوقع غالبًا:

```text
Status = partially_dispensed
```

يجب على ERP الاحتفاظ بالكميات الناجحة محليًا إذا لم توفر استجابة EPTTS كمية متبقية.

---

## 16. Dispense Cancel

تستخدم لإلغاء عملية `Full Pack Dispensing` لعبوة حالتها `dispensed`.

> لا تستخدم هذه الدالة لإلغاء Partial Dispensing ما لم تعتمد EPTTS مواصفة مستقلة لذلك.

### الشروط قبل الإرسال

- `Verified = true`.
- `Pack.Status = dispensed`.
- العبوة مرتبطة بالصيدلية الحالية.

### الاستخدام

```csharp
var request = new DispenseCancelRequest
{
    Sgtin = sgtin,
    EventDateTime = DateTimeOffset.Now
};

var result = await client.CancelDispensingAsync(request);
```

بعد `S - Successful` نفذ `VerifyProduct`. المتوقع:

```text
Status = active
CurrentGln = PharmacyGln
```

---

## 17. Return Request to Branch

تستخدم لإرسال عبوة من الصيدلية إلى فرع كمرتجع.

### الشروط قبل الإرسال

- العبوة موجودة لدى الصيدلية.
- الحالة `active` حسب السيناريو المدعوم.
- بيانات الفرع المستقبل صحيحة.
- `ReturnRequestNumber` فريد ومحفوظ داخل ERP.

### الاستخدام

```csharp
var request = new ReturnRequest
{
    Sgtin = sgtin,
    DestinationGln = destinationBranch.Gln,
    DestinationSgln = destinationBranch.Sgln,
    ReturnRequestNumber = erpReturnDocumentNumber,
    EventDateTime = DateTimeOffset.Now
};

var result = await client.ReturnToBranchAsync(request);
```

### ما يجب حفظه

احفظ على الأقل:

```csharp
request.Sgtin
request.DestinationGln
request.DestinationSgln
request.ReturnRequestNumber
result.Data.RequestInstanceIdentifier
result.Data.MessageId
result.Data.StatusQueryIdentifier
```

بعد `S - Successful` تصبح العبوة في حالة مرتبطة بالمرتجع، مثل:

```text
returned
```

القيمة الدقيقة تعتمد على البيئة وسير العمل المعتمد.

---

## 18. Return Cancel

تستخدم لإلغاء مرتجع سابق لم يستلمه الفرع بعد.

### القاعدة الأساسية

استخدم نفس القيم الخاصة بالمرتجع الأصلي:

- نفس `SGTIN`.
- نفس `DestinationGln`.
- نفس `ReturnRequestNumber`.

### الاستخدام

```csharp
var originalReturn = LoadOriginalReturn(returnId);

var request = new ReturnCancelRequest
{
    Sgtin = originalReturn.Sgtin,
    DestinationGln = originalReturn.DestinationGln,
    ReturnRequestNumber = originalReturn.ReturnRequestNumber,
    EventDateTime = DateTimeOffset.Now
};

var result = await client.CancelReturnAsync(request);
```

### لا تستخدم MessageId بدل ReturnRequestNumber

خطأ:

```csharp
ReturnRequestNumber = originalReturn.MessageId;
```

صحيح:

```csharp
ReturnRequestNumber = originalReturn.ReturnRequestNumber;
```

إذا لم يكن الرقم مطابقًا للمرتجع الأصلي، قد يعيد EPTTS:

```text
No PENDING return found
```

بعد `S - Successful` نفذ `VerifyProduct`. المتوقع أن تعود العبوة إلى:

```text
Status = active
CurrentGln = PharmacyGln
```

---

## 19. حفظ العمليات في قاعدة بيانات ERP

احفظ سجلًا مستقلًا لكل عملية إرسال.

### الحقول المقترحة

```text
Id
OperationType
ErpDocumentId
Sgtin
Sscc
Quantity
SourceGln
DestinationGln
DestinationSgln
ReturnRequestNumber
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
LocalStatus
HttpStatusCode
ErrorCode
ErrorMessage
RawRequest
RawResponse
CreatedAt
CreatedBy
CompletedAt
```

### الحالات المحلية المقترحة

```text
Draft
Pending
Successful
Failed
NeedsReview
```

استخدم الحالات كما يلي:

- `Draft`: قبل الإرسال.
- `Pending`: بعد `202 / I001` مع معرف قابل للمتابعة.
- `Successful`: بعد `S - Successful` فقط.
- `Failed`: بعد `E - Application Error` أو رفض واضح.
- `NeedsReview`: عند Timeout أو حالة غير مؤكدة.

### مثال حفظ العملية المعلقة

```csharp
private void SavePendingSubmission(
    string operationType,
    string erpDocumentId,
    string sgtin,
    SubmissionResponse response,
    string rawRequest,
    string rawResponse)
{
    repository.Insert(new EpttsOperationLog
    {
        OperationType = operationType,
        ErpDocumentId = erpDocumentId,
        Sgtin = sgtin,
        RequestInstanceIdentifier = response.RequestInstanceIdentifier,
        MessageId = response.MessageId,
        StatusQueryIdentifier = response.StatusQueryIdentifier,
        LocalStatus = "Pending",
        RawRequest = rawRequest,
        RawResponse = rawResponse,
        CreatedAt = DateTime.Now
    });
}
```

---

## 20. منع تكرار العمليات

يجب على ERP منع إنشاء عملية جديدة إذا كانت هناك عملية من نفس النوع على نفس العبوة بحالة `Pending`.

مثال:

```csharp
if (repository.HasPendingOperation(
        operationType: "FullDispensing",
        sgtin: sgtin))
{
    ShowError("توجد عملية EPTTS معلقة على العبوة نفسها.");
    return;
}
```

### قواعد منع التكرار

- عطّل زر الإرسال بعد أول ضغطة.
- احفظ العملية محليًا فور الحصول على معرف المتابعة.
- لا تعِد إرسال العملية بعد Timeout مباشرة.
- لا تنشئ `Return Cancel` مرتين للمرتجع نفسه.
- لا تنشئ صرفًا جديدًا أثناء وجود صرف سابق `Pending` على SGTIN نفسه.
- لا تستخدم `InstanceIdentifier` قديمًا لعملية جديدة.

---

## 21. التعامل مع Timeout وCancellation

### Timeout

إذا أعادت الدالة:

```text
ErrorCode = REQUEST_TIMEOUT
```

فهذا لا يثبت أن EPTTS لم يستلم الرسالة.

الإجراء الصحيح:

1. احفظ العملية بحالة `NeedsReview`.
2. افحص `RawRequest`.
3. استخرج `instanceIdentifier` إذا كان موجودًا.
4. نفذ `GetMessageStatusAsync` على المعرف.
5. لا تعِد الإرسال قبل معرفة حالة الرسالة السابقة.

### CancellationToken

يمكن استخدامه لإلغاء انتظار الشاشة:

```csharp
using (var cancellationSource = new CancellationTokenSource())
{
    var result = await client.VerifyProductAsync(
        sgtin,
        cancellationSource.Token);
}
```

إلغاء الطلب محليًا لا يعني إلغاء العملية داخل EPTTS.

### استخراج InstanceIdentifier من RawRequest عند الحاجة

إذا لم تتوفر قيمة منظمة بسبب Timeout، يمكن لفريق الدعم مراجعة `RawRequest` والبحث عن:

```json
{
  "instanceIdentifier": "32-character-identifier"
}
```

هذه خطوة استثنائية للتعافي والتشخيص، وليست مسار الاستخدام الطبيعي.

---

## 22. الأخطاء الشائعة

### REQUEST_TIMEOUT

انتهت مهلة HTTP، وقد تكون الرسالة وصلت إلى الخادم.

الإجراء: استعلم عن المعرف السابق ولا تعِد الإرسال مباشرة.

### REQUEST_CANCELLED

المستخدم أوقف انتظار البرنامج.

الإجراء: تعامل معها كحالة غير مؤكدة إذا كانت العملية تغيّر حالة العبوة.

### HTTP 401

بيانات المصادقة غير صحيحة أو غير مقبولة.

الإجراء: راجع `IntegratorKey` والبيئة المستخدمة.

### HTTP 403

لا توجد صلاحية على العملية أو الرسالة.

الإجراء: راجع `IntegratorKey` و`PharmacyGln` وملكية الرسالة.

### HTTP 404

المعرف أو Endpoint أو الرسالة غير موجودة.

الإجراء: راجع البيئة والمعرف، ولا تفترض أن العملية فشلت تجاريًا دون مراجعة.

### PACK_NOT_FOUND

العبوة غير موجودة في البيئة الحالية أو أن المعرف غير صحيح.

### PACK_NOT_OWNED

`CurrentGln` لا يطابق الصيدلية الحالية.

### E - Application Error

استعلام الحالة نجح، لكن العملية الأصلية رفضت وظيفيًا.

اقرأ:

```csharp
result.Data.FirstErrorMessage
result.Data.LogList
```

### No PENDING return found

أحد الأسباب المحتملة:

- استخدام `MessageId` بدل `ReturnRequestNumber`.
- استخدام رقم مرتجع مختلف.
- المرتجع لم يعد Pending.
- الفرع استلم المرتجع.
- تم إلغاء المرتجع سابقًا.

### Partial dispensing not allowed

المنتج لا يسمح بالصرف الجزئي أو الكمية أو وحدة الصرف غير مناسبة.

---

## 23. قائمة الدوال العامة

### التحقق من العبوة

```csharp
Task<EpttsResult<VerifyProductResponse>>
    VerifyProductAsync(string productId);
```

```csharp
Task<EpttsResult<VerifyProductResponse>>
    VerifyProductAsync(
        VerifyProductRequest request,
        CancellationToken cancellationToken);
```

### حالة الرسالة

```csharp
Task<EpttsResult<MessageStatusResponse>>
    GetMessageStatusAsync(string instanceIdentifier);
```

```csharp
Task<EpttsResult<MessageStatusResponse>>
    WaitForFinalStatusAsync(
        string instanceIdentifier,
        TimeSpan pollingInterval,
        TimeSpan maximumWaitTime);
```

### الاستلام من فرع

```csharp
Task<EpttsResult<SubmissionResponse>>
    ReceiveFromBranchAsync(ReceivingRequest request);
```

### صرف عبوة كاملة

```csharp
Task<EpttsResult<SubmissionResponse>>
    DispenseFullPackAsync(FullDispensingRequest request);
```

### الصرف الجزئي

```csharp
Task<EpttsResult<SubmissionResponse>>
    DispensePartialAsync(PartialDispensingRequest request);
```

### إلغاء الصرف

```csharp
Task<EpttsResult<SubmissionResponse>>
    CancelDispensingAsync(DispenseCancelRequest request);
```

### إرسال مرتجع

```csharp
Task<EpttsResult<SubmissionResponse>>
    ReturnToBranchAsync(ReturnRequest request);
```

### إلغاء مرتجع

```csharp
Task<EpttsResult<SubmissionResponse>>
    CancelReturnAsync(ReturnCancelRequest request);
```

---

## 24. قائمة مراجعة قبل الإنتاج

- [ ] تم اختبار المكتبة من مشروع `.NET Framework 4.8` مماثل للـ ERP.
- [ ] تم نسخ `ModernSoft.Eptts.Client.dll` وملف XML و`Newtonsoft.Json.dll`.
- [ ] تم ضبط `BaseUrl` لكل بيئة.
- [ ] تم حفظ `IntegratorKey` بطريقة محمية.
- [ ] لا يظهر `IntegratorKey` في Logs أو UI.
- [ ] تم ربط `PharmacyGln` و`PharmacySgln` الصحيحين بكل فرع.
- [ ] يوجد جدول لتسجيل عمليات EPTTS.
- [ ] توجد حالات `Pending`, `Successful`, `Failed`, `NeedsReview`.
- [ ] يوجد Job أو Timer لمتابعة العمليات `Pending`.
- [ ] لا تعتمد أي عملية عند `HTTP 202` فقط.
- [ ] يوجد منع لتكرار العملية على SGTIN أثناء وجود عملية معلقة.
- [ ] تم اختبار `VerifyProduct` قبل وبعد كل عملية.
- [ ] تم اختبار `S - Successful`.
- [ ] تم اختبار `E - Application Error`.
- [ ] تم اختبار HTTP 401 و403 و404.
- [ ] تم اختبار Timeout وإلغاء انتظار الشاشة.
- [ ] تم اختبار Receiving في Staging.
- [ ] تم اختبار Full Pack Dispensing في Staging.
- [ ] تم اختبار Partial Dispensing في Staging.
- [ ] تم اختبار Dispense Cancel في Staging.
- [ ] تم اختبار Return وReturn Cancel في Staging.
- [ ] يتم حفظ `ReturnRequestNumber` الأصلي وعدم استبداله بـ MessageId.
- [ ] لا تُرسل بيانات شخصية مباشرة داخل `PatientReference`.
- [ ] يتم حفظ `RawRequest` و`RawResponse` دون بيانات المصادقة السرية.

---

## قالب موحد لإرسال أي عملية EPCIS

```csharp
try
{
    using (var client = new EpttsClient(options))
    {
        // تحقق من العبوة قبل العملية عند الحاجة.
        var verification = await client.VerifyProductAsync(sgtin);

        if (!verification.IsSuccess ||
            verification.Data == null ||
            !verification.Data.Verified ||
            verification.Data.Pack == null)
        {
            ShowError("تعذر التحقق من العبوة.");
            return;
        }

        // أنشئ Request الخاص بالعملية.
        var submission = await client.SomeOperationAsync(request);

        if (!submission.IsSuccess)
        {
            SaveFailedHttpResult(
                submission.ErrorCode,
                submission.ErrorMessage,
                submission.RawRequest,
                submission.RawResponse);

            return;
        }

        if (submission.Data == null ||
            !submission.Data.CanQueryFinalStatus)
        {
            SaveNeedsReview(
                submission.RawRequest,
                submission.RawResponse);

            return;
        }

        SavePendingOperation(
            submission.Data.RequestInstanceIdentifier,
            submission.Data.MessageId,
            submission.Data.StatusQueryIdentifier,
            submission.RawRequest,
            submission.RawResponse);
    }
}
catch (EpttsConfigurationException ex)
{
    ShowConfigurationError(ex.Message);
}
catch (EpttsValidationException ex)
{
    ShowInputError(ex.Message);
}
catch (EpttsException ex)
{
    ShowIntegrationError(ex.Message);
}
```

---

## قالب Job لمتابعة العمليات المعلقة

```csharp
public async Task ProcessPendingEpttsOperationsAsync()
{
    var pendingOperations = repository.GetPendingOperations();

    foreach (var operation in pendingOperations)
    {
        try
        {
            var options = LoadEpttsOptions(operation.PharmacyId);

            using (var client = new EpttsClient(options))
            {
                var result = await client.GetMessageStatusAsync(
                    operation.StatusQueryIdentifier);

                if (!result.IsSuccess)
                {
                    repository.SaveStatusQueryFailure(
                        operation.Id,
                        result.HttpStatusCode,
                        result.ErrorCode,
                        result.ErrorMessage,
                        result.RawResponse);

                    continue;
                }

                if (result.Data == null || !result.Data.IsFinal)
                {
                    continue;
                }

                if (result.Data.IsSuccessful)
                {
                    repository.MarkSuccessful(
                        operation.Id,
                        result.RawResponse);

                    ApplySuccessfulErpAction(operation);
                }
                else if (result.Data.IsApplicationError)
                {
                    repository.MarkFailed(
                        operation.Id,
                        result.Data.FirstErrorMessage,
                        result.RawResponse);
                }
            }
        }
        catch (Exception ex)
        {
            LogUnexpectedError(operation.Id, ex);
        }
    }
}
```

---

## الدعم والتشخيص

عند إرسال مشكلة إلى فريق دعم المكتبة، أرسل القيم التالية:

```text
OperationType
PharmacyGln
SGTIN أو SSCC
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
HttpStatusCode
ErrorCode
ErrorMessage
RawRequest
RawResponse
وقت حدوث المشكلة
```

لا ترسل:

```text
IntegratorKey
بيانات المريض الشخصية
كلمات المرور
بيانات دخول المستخدم
```

---

## خلاصة الاستخدام

```text
Configure EpttsOptions
→ Create EpttsClient
→ VerifyProduct عند الحاجة
→ Create Request
→ Send Operation
→ Save Pending
→ Query Message Status
→ Accept only S - Successful
→ VerifyProduct بعد العملية
→ Update ERP document and inventory
```

الواجهة العامة التي يحتاجها مطور ERP هي:

```text
EpttsOptions
EpttsClient
Request classes
EpttsResult<T>
Response classes
```

أي تفاصيل أخرى داخل المكتبة لا يحتاج مطور ERP إلى استخدامها أو تعديلها.
