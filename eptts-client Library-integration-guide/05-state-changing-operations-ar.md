# الجزء الخامس: العمليات التي تغير حالة العبوة

> يشرح هذا الجزء العمليات التي تنشئ أحداثًا تجارية داخل EPTTS وقد تغير حالة العبوة أو موقعها أو مسارها. يجب التعامل مع هذه العمليات بحذر، لأن نجاح طلب HTTP أو استلام `HTTP 202` لا يعني نجاح العملية نهائيًا. يجب حفظ الرسالة بحالة `Pending` ثم متابعة `StatusQueryIdentifier` حتى ظهور نتيجة نهائية.

---

## 1. العمليات المشمولة في هذا الجزء

تغطي المكتبة العمليات التالية:

```text
Receiving from Branch
Full Pack Dispensing
Partial Dispensing
Dispense Cancel
Return to Branch
Return Cancel
```

والدوال المقابلة لها:

```csharp
ReceiveFromBranchAsync
DispenseFullPackAsync
DispensePartialAsync
CancelDispensingAsync
ReturnToBranchAsync
CancelReturnAsync
```

---

## 2. الفرق بين عملية القراءة وعملية تغيير الحالة

عملية مثل:

```text
VerifyProduct
```

تقرأ بيانات العبوة فقط.

أما عمليات هذا الجزء فقد:

- تغير حالة العبوة.
- تغير موقعها أو الجهة المرتبطة بها.
- تسجل صرفًا أو استلامًا أو مرتجعًا.
- تنشئ رسالة غير متزامنة تحتاج متابعة.

لذلك لا تطبق Retry تلقائيًا على هذه العمليات عند `Timeout` أو انقطاع اتصال غير مؤكد.

---

## 3. التسلسل الإلزامي لأي عملية

استخدم التسلسل التالي:

```text
1. التحقق من إعدادات الاتصال
2. التحقق من مدخلات ERP
3. VerifyProduct عند الحاجة
4. منع وجود عملية Pending مكررة
5. إنشاء Request
6. إرسال العملية مرة واحدة
7. فحص EpttsResult<SubmissionResponse>
8. حفظ معرفات الرسالة وRawRequest وRawResponse
9. حفظ العملية محليًا بحالة Pending
10. تنفيذ GetMessageStatusAsync
11. اعتماد النجاح عند S - Successful فقط
12. تنفيذ VerifyProduct بعد النجاح عند الحاجة
13. تحديث مستند ERP والمخزون وفق نوع العملية
```

---

## 4. Namespaces المطلوبة

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Eptts.Client.Clients;
using Eptts.Client.Configuration;
using Eptts.Client.Exceptions;
using Eptts.Client.Requests;
using Eptts.Client.Responses;
```

---

## 5. قالب موحد لمعالجة Submission

بعد إرسال أي عملية:

```csharp
if (!submission.IsSuccess)
{
    /*
     * فشل تقني أو HTTP.
     * لا تعتبر العملية ناجحة.
     * راجع ErrorCode وErrorMessage وRawResponse.
     */

    return;
}

if (submission.Data == null)
{
    /*
     * استجابة منظمة فارغة.
     * احفظ العملية NeedsReview.
     */

    return;
}

if (!submission.Data.IsAcceptedForProcessing)
{
    /*
     * الرسالة لم تُقبل للمعالجة.
     */

    return;
}

if (!submission.Data.CanQueryFinalStatus ||
    string.IsNullOrWhiteSpace(
        submission.Data.StatusQueryIdentifier))
{
    /*
     * لا يوجد معرف متابعة صالح.
     * احفظ NeedsReview مع البيانات الخام.
     */

    return;
}

string statusQueryIdentifier =
    submission.Data.StatusQueryIdentifier;

/*
 * احفظ العملية Pending فورًا.
 */
```

---

## 6. البيانات المشتركة التي يجب حفظها

بعد قبول أي عملية، احفظ عند توفرها:

```csharp
submission.Data.RequestInstanceIdentifier
submission.Data.MessageId
submission.Data.StatusQueryIdentifier
submission.RawRequest
submission.RawResponse
submission.HttpStatusCode
submission.ErrorCode
submission.ErrorMessage
```

واحفظ أيضًا:

```text
OperationType
ErpDocumentId
PharmacyGln
SGTIN أو SSCC
LocalStatus = Pending
CreatedAt
CreatedBy
```

ثم أضف الحقول الخاصة بكل عملية، مثل `Quantity` أو `ReturnRequestNumber`.

---

# القسم الأول: Receiving from Branch

## 7. الغرض من Receiving

تستخدم عملية `Receiving from Branch` لتسجيل استلام شحنة أو عبوات من فرع مرسل إلى الصيدلية الحالية.

الدالة:

```csharp
ReceiveFromBranchAsync
```

يمكن إرسال:

- `SSCC` لحاوية كاملة.
- قائمة `SGTINs` لعبوات منفردة.

---

## 8. البيانات المطلوبة قبل Receiving

يجب توفير:

```text
SourceGln
SourceSgln
EpcList
EventDateTime
```

ويمكن توفير:

```text
InvoiceNumber
```

### `SourceGln`

GLN الفرع المرسل، وليس `PharmacyGln` الخاص بالصيدلية المستلمة.

### `SourceSgln`

SGLN موقع الفرع المرسل.

### `EpcList`

قائمة تحتوي على `SSCC` أو `SGTINs` المستلمة.

### `InvoiceNumber`

رقم الفاتورة أو مستند الشحن عند توفره.

---

## 9. استلام SSCC كامل

```csharp
ReceivingRequest request =
    new ReceivingRequest
    {
        SourceGln =
            sourceBranchGln,

        SourceSgln =
            sourceBranchSgln,

        EpcList =
            new List<string>
            {
                shipmentSscc
            },

        InvoiceNumber =
            invoiceNumber,

        EventDateTime =
            DateTimeOffset.Now
    };
```

المتغيرات التالية يجب أن يوفرها ERP:

```csharp
sourceBranchGln
sourceBranchSgln
shipmentSscc
invoiceNumber
```

---

## 10. استلام عبوات SGTIN منفردة

```csharp
ReceivingRequest request =
    new ReceivingRequest
    {
        SourceGln =
            sourceBranchGln,

        SourceSgln =
            sourceBranchSgln,

        EpcList =
            new List<string>
            {
                firstSgtin,
                secondSgtin
            },

        EventDateTime =
            DateTimeOffset.Now
    };
```

لا ترسل `SSCC` وكل محتوياته من `SGTINs` في الطلب نفسه إلا إذا كانت المواصفة المعتمدة تطلب ذلك صراحة.

---

## 11. إرسال Receiving

```csharp
EpttsResult<SubmissionResponse> submission;

using (EpttsClient client =
    new EpttsClient(options))
{
    submission =
        await client.ReceiveFromBranchAsync(
            request,
            cancellationToken);
}
```

---

## 12. ما الذي يحفظه ERP في Receiving؟

احفظ بالإضافة إلى الحقول المشتركة:

```text
SourceGln
SourceSgln
SSCC أو قائمة SGTINs
InvoiceNumber
EventDateTime
ErpReceivingDocumentId
```

بعد القبول:

```text
LocalStatus = Pending
```

لا تعتمد الاستلام نهائيًا بسبب `HTTP 202` فقط.

---

## 13. ما بعد نجاح Receiving

بعد `S - Successful`:

1. حدث سجل التكامل إلى `Successful`.
2. طبق اعتماد مستند الاستلام حسب قواعد ERP.
3. حدث المخزون حسب تصميم النظام.
4. نفذ `VerifyProduct` على عبوة اختبار من الشحنة عند الحاجة.

الحالة المتوقعة غالبًا:

```text
Status = active
CurrentGln = PharmacyGln
```

القيمة الفعلية تعتمد على سير العمل والبيئة.

---

## 14. أخطاء Receiving الشائعة

- بيانات الفرع المرسل غير صحيحة.
- `SourceGln` يساوي جهة غير مرسلة.
- `EpcList` فارغة.
- إرسال `GTIN` بدل `SGTIN`.
- إرسال `SSCC` غير صالح.
- استلام شحنة غير موجهة للصيدلية.
- تكرار عملية Receiving على الشحنة نفسها.
- إعادة الإرسال بعد Timeout دون الاستعلام عن الرسالة السابقة.

---

# القسم الثاني: Full Pack Dispensing

## 15. الغرض من Full Pack Dispensing

تستخدم لتسجيل صرف عبوة كاملة.

الدالة:

```csharp
DispenseFullPackAsync
```

تتعامل العملية مع `SGTIN` لعبوة واحدة محددة.

---

## 16. الشروط المبدئية قبل الصرف الكامل

نفذ `VerifyProduct` وتأكد عادة من:

```text
IsSuccess = true
Verified = true
Pack موجودة
Status = active
CurrentGln = PharmacyGln
IsRecalled = false
Alerts لا تمنع العملية
```

مثال:

```csharp
bool canTryDispensing =
    verification.IsSuccess &&
    verification.Data != null &&
    verification.Data.Verified &&
    verification.Data.Pack != null &&
    string.Equals(
        verification.Data.Pack.Status,
        "active",
        StringComparison.OrdinalIgnoreCase) &&
    string.Equals(
        verification.Data.Pack.CurrentGln,
        options.PharmacyGln,
        StringComparison.Ordinal) &&
    !verification.Data.Pack.IsRecalled &&
    (verification.Data.Alerts == null ||
     verification.Data.Alerts.Count == 0);
```

هذا تحقق محلي مبدئي، بينما يظل القرار النهائي لدى EPTTS.

---

## 17. إنشاء FullDispensingRequest

```csharp
FullDispensingRequest request =
    new FullDispensingRequest
    {
        Sgtin =
            sgtin,

        PatientReference =
            patientTransactionReference,

        PrescriptionReference =
            prescriptionReference,

        EventDateTime =
            DateTimeOffset.Now
    };
```

### `PatientReference`

استخدم مرجعًا داخليًا فقط، مثل:

```text
ERP-SALE-1001
```

لا ترسل اسم المريض أو الرقم القومي أو الهاتف أو العنوان.

### `PrescriptionReference`

مرجع داخلي للوصفة عند توفره، مثل:

```text
ERP-RX-1001
```

يمكن تمرير `null` للمراجع الاختيارية إذا لم تكن مطلوبة.

---

## 18. إرسال Full Pack Dispensing

```csharp
EpttsResult<SubmissionResponse> submission;

using (EpttsClient client =
    new EpttsClient(options))
{
    submission =
        await client.DispenseFullPackAsync(
            request,
            cancellationToken);
}
```

بعد قبول الرسالة، احفظها `Pending` ولا تثبت النجاح النهائي بعد.

---

## 19. ما الذي يحفظه ERP في الصرف الكامل؟

```text
SGTIN
ErpSaleDocumentId
PatientReference الداخلي عند السماح بحفظه
PrescriptionReference
EventDateTime
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
RawRequest
RawResponse
LocalStatus = Pending
```

لا تحفظ بيانات مريض شخصية مباشرة في سجل التكامل.

---

## 20. ما بعد نجاح الصرف الكامل

بعد `S - Successful`:

1. حدث العملية إلى `Successful`.
2. طبق اعتماد عملية البيع أو الصرف حسب قواعد ERP.
3. نفذ `VerifyProduct` للتأكد من الحالة النهائية عند الحاجة.

الحالة المتوقعة:

```text
Status = dispensed
```

إذا أعادت EPTTS نجاحًا نهائيًا لكن `VerifyProduct` لم يعرض الحالة المتوقعة، لا تعِد إرسال الصرف. سجل الحالة للمراجعة.

---

## 21. أخطاء الصرف الكامل الشائعة

- العبوة ليست `active`.
- العبوة ليست مرتبطة بالصيدلية الحالية.
- العبوة مستدعاة.
- وجود `Alerts` تمنع العملية.
- `SGTIN` غير صحيح.
- وجود عملية صرف `Pending` سابقة على العبوة.
- إرسال بيانات شخصية مباشرة داخل `PatientReference`.
- اعتبار `HTTP 202` نجاحًا نهائيًا.

---

# القسم الثالث: Partial Dispensing

## 22. الغرض من Partial Dispensing

تستخدم لتسجيل صرف كمية من عبوة تسمح بالصرف الجزئي.

الدالة:

```csharp
DispensePartialAsync
```

---

## 23. الشروط المبدئية قبل الصرف الجزئي

نفذ `VerifyProduct` وتأكد عادة من:

```text
Verified = true
Status = active أو partially_dispensed
CurrentGln = PharmacyGln
IsRecalled = false
Alerts لا تمنع العملية
المنتج يسمح بالصرف الجزئي
Quantity > 0
```

مثال للحالة:

```csharp
bool supportedStatus =
    string.Equals(
        verification.Data.Pack.Status,
        "active",
        StringComparison.OrdinalIgnoreCase) ||
    string.Equals(
        verification.Data.Pack.Status,
        "partially_dispensed",
        StringComparison.OrdinalIgnoreCase);
```

---

## 24. معنى Quantity

`Quantity` هي كمية العملية الحالية فقط.

مثال:

```csharp
Quantity = 1;
```

لا تعني بالضرورة قرصًا واحدًا.

يجب أن يعرف ERP وحدة الصرف المعتمدة من Master Data، مثل:

- قرص.
- شريط.
- Ampoule.
- وحدة بيع جزئي أخرى.

لا تخمّن الوحدة من اسم المنتج.

---

## 25. إنشاء PartialDispensingRequest

```csharp
PartialDispensingRequest request =
    new PartialDispensingRequest
    {
        Sgtin =
            sgtin,

        Quantity =
            quantityToDispense,

        PatientReference =
            patientTransactionReference,

        PrescriptionReference =
            prescriptionReference,

        EventDateTime =
            DateTimeOffset.Now
    };
```

يجب أن تكون:

```csharp
quantityToDispense > 0
```

---

## 26. إرسال Partial Dispensing

```csharp
EpttsResult<SubmissionResponse> submission;

using (EpttsClient client =
    new EpttsClient(options))
{
    submission =
        await client.DispensePartialAsync(
            request,
            cancellationToken);
}
```

بعد قبول الرسالة، احفظ `Quantity` مع العملية `Pending`.

---

## 27. ما الذي يحفظه ERP في الصرف الجزئي؟

```text
SGTIN
Quantity
Unit of Measure عند توفرها في ERP
ErpSaleDocumentId
PatientReference الداخلي
PrescriptionReference
EventDateTime
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
RawRequest
RawResponse
LocalStatus = Pending
```

يجب على ERP الاحتفاظ بالكميات الناجحة محليًا إذا لم توفر EPTTS كمية متبقية منظمة يمكن الاعتماد عليها.

---

## 28. ما بعد نجاح الصرف الجزئي

بعد `S - Successful`:

1. حدث العملية إلى `Successful`.
2. ثبت كمية العملية حسب قواعد ERP.
3. نفذ `VerifyProduct` عند الحاجة.

الحالة المتوقعة غالبًا:

```text
Status = partially_dispensed
```

قد تعتمد الحالة النهائية على قواعد المنتج والكمية.

---

## 29. أخطاء الصرف الجزئي الشائعة

- المنتج لا يسمح بالصرف الجزئي.
- الحالة لا تسمح بالعملية.
- `Quantity` تساوي صفرًا أو قيمة سالبة.
- وحدة الكمية غير صحيحة.
- الكمية أكبر من المتاح.
- العبوة لا تخص الصيدلية.
- العبوة مستدعاة.
- تكرار عملية `Pending` على العبوة نفسها.

---

# القسم الرابع: Dispense Cancel

## 30. الغرض من Dispense Cancel

تستخدم لإلغاء عملية صرف عبوة كاملة سابقة.

الدالة:

```csharp
CancelDispensingAsync
```

لا تستخدم هذا المسار لإلغاء `Partial Dispensing` إلا إذا كانت هناك مواصفة معتمدة تدعم ذلك.

---

## 31. الشروط المبدئية قبل Dispense Cancel

نفذ `VerifyProduct` وتأكد عادة من:

```text
Verified = true
Status = dispensed
CurrentGln = PharmacyGln
```

مثال:

```csharp
bool canTryCancel =
    verification.IsSuccess &&
    verification.Data != null &&
    verification.Data.Verified &&
    verification.Data.Pack != null &&
    string.Equals(
        verification.Data.Pack.Status,
        "dispensed",
        StringComparison.OrdinalIgnoreCase) &&
    string.Equals(
        verification.Data.Pack.CurrentGln,
        options.PharmacyGln,
        StringComparison.Ordinal);
```

---

## 32. إنشاء DispenseCancelRequest

```csharp
DispenseCancelRequest request =
    new DispenseCancelRequest
    {
        Sgtin =
            sgtin,

        EventDateTime =
            DateTimeOffset.Now
    };
```

---

## 33. إرسال Dispense Cancel

```csharp
EpttsResult<SubmissionResponse> submission;

using (EpttsClient client =
    new EpttsClient(options))
{
    submission =
        await client.CancelDispensingAsync(
            request,
            cancellationToken);
}
```

لا تعد العبوة إلى المخزون المتاح بسبب قبول الرسالة فقط.

---

## 34. ما الذي يحفظه ERP في Dispense Cancel؟

```text
SGTIN
OriginalDispensingOperationId إن توفر
ErpCancellationDocumentId
EventDateTime
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
RawRequest
RawResponse
LocalStatus = Pending
```

اربط عملية الإلغاء بعملية الصرف الأصلية داخل ERP متى أمكن.

---

## 35. ما بعد نجاح Dispense Cancel

بعد `S - Successful`:

1. حدث سجل الإلغاء إلى `Successful`.
2. طبق أثر الإلغاء داخل ERP حسب قواعد النظام.
3. نفذ `VerifyProduct`.

الحالة المتوقعة:

```text
Status = active
CurrentGln = PharmacyGln
```

إذا لم تظهر الحالة المتوقعة، سجل الحالة للمراجعة ولا تعِد إرسال الإلغاء.

---

## 36. أخطاء Dispense Cancel الشائعة

- العبوة ليست `dispensed`.
- محاولة إلغاء Partial Dispensing بالمسار الحالي.
- العبوة لا تخص الصيدلية.
- عدم ربط الإلغاء بالصرف الأصلي داخل ERP.
- تكرار إلغاء العملية.
- إعادة الإرسال بعد Timeout دون متابعة الرسالة السابقة.

---

# القسم الخامس: Return to Branch

## 37. الغرض من Return to Branch

تستخدم لإرسال عبوة من الصيدلية إلى فرع كمرتجع.

الدالة:

```csharp
ReturnToBranchAsync
```

---

## 38. الشروط المبدئية قبل Return

نفذ `VerifyProduct` وتأكد عادة من:

```text
Verified = true
Status = active
CurrentGln = PharmacyGln
بيانات الفرع المستقبل صحيحة
ReturnRequestNumber فريد
لا توجد عملية Pending متعارضة
```

راجع أيضًا `Alerts` وسياسة المنتج.

---

## 39. إنشاء ReturnRequestNumber

`ReturnRequestNumber` رقم تجاري ينشئه ERP ويحفظه.

مثال:

```text
RET-UAT-000123
```

يمكن أن يكون رقم مستند المرتجع بشرط أن يكون فريدًا وفق سياسة ERP.

لا تستخدم:

```text
MessageId
RequestInstanceIdentifier
StatusQueryIdentifier
```

مكانه.

---

## 40. إنشاء ReturnRequest

```csharp
ReturnRequest request =
    new ReturnRequest
    {
        Sgtin =
            sgtin,

        DestinationGln =
            destinationBranchGln,

        DestinationSgln =
            destinationBranchSgln,

        ReturnRequestNumber =
            erpReturnRequestNumber,

        EventDateTime =
            DateTimeOffset.Now
    };
```

المتغيرات التالية تأتي من ERP:

```csharp
destinationBranchGln
destinationBranchSgln
erpReturnRequestNumber
```

---

## 41. إرسال Return to Branch

```csharp
EpttsResult<SubmissionResponse> submission;

using (EpttsClient client =
    new EpttsClient(options))
{
    submission =
        await client.ReturnToBranchAsync(
            request,
            cancellationToken);
}
```

---

## 42. ما الذي يحفظه ERP في Return؟

احفظ:

```text
SGTIN
DestinationGln
DestinationSgln
ReturnRequestNumber
ErpReturnDocumentId
EventDateTime
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
RawRequest
RawResponse
LocalStatus = Pending
```

يجب الاحتفاظ بـ `ReturnRequestNumber` الأصلي دون تغيير، لأنه مطلوب عند `Return Cancel`.

---

## 43. ما بعد نجاح Return

بعد `S - Successful`:

1. حدث سجل المرتجع إلى `Successful`.
2. طبق أثر المرتجع في ERP.
3. امنع استخدام العبوة في عملية بيع جديدة حسب قواعد النظام.
4. نفذ `VerifyProduct` عند الحاجة.

قد تصبح الحالة:

```text
returned
```

أو حالة مرتبطة بسير المرتجع حسب البيئة.

لا تفترض قيمة غير موجودة. اعتمد على الاستجابة الفعلية وبيانات UAT.

---

## 44. أخطاء Return الشائعة

- العبوة ليست `active`.
- العبوة ليست لدى الصيدلية الحالية.
- بيانات الفرع المستقبل غير صحيحة.
- `ReturnRequestNumber` فارغ أو مكرر.
- عدم حفظ رقم المرتجع الأصلي.
- وجود عملية `Pending` متعارضة.
- اعتبار قبول الرسالة نجاحًا نهائيًا.

---

# القسم السادس: Return Cancel

## 45. الغرض من Return Cancel

تستخدم لإلغاء مرتجع سابق ما زال قابلًا للإلغاء ولم يكتمل مساره بطريقة تمنع الإلغاء.

الدالة:

```csharp
CancelReturnAsync
```

---

## 46. القاعدة الأساسية لإلغاء المرتجع

استخدم القيم الأصلية نفسها:

```text
نفس SGTIN
نفس DestinationGln
نفس ReturnRequestNumber
```

لا تنشئ `ReturnRequestNumber` جديدًا.

لا تستخدم `MessageId` بدل رقم المرتجع.

---

## 47. تحميل بيانات المرتجع الأصلي

يجب أن يحمل ERP سجل المرتجع الأصلي.

مثال Class تخص ERP وليست جزءًا من المكتبة:

```csharp
public sealed class ErpReturnRecord
{
    public string Sgtin { get; set; } =
        string.Empty;

    public string DestinationGln { get; set; } =
        string.Empty;

    public string ReturnRequestNumber { get; set; } =
        string.Empty;
}
```

ثم:

```csharp
ErpReturnRecord originalReturn =
    LoadOriginalReturn(
        returnDocumentId);
```

الدالة `LoadOriginalReturn` والمتغير `returnDocumentId` أمثلة من ERP.

---

## 48. إنشاء ReturnCancelRequest

```csharp
ReturnCancelRequest request =
    new ReturnCancelRequest
    {
        Sgtin =
            originalReturn.Sgtin,

        DestinationGln =
            originalReturn.DestinationGln,

        ReturnRequestNumber =
            originalReturn.ReturnRequestNumber,

        EventDateTime =
            DateTimeOffset.Now
    };
```

لاحظ أن Request لا ينشئ رقم مرتجع جديدًا.

---

## 49. الخطأ الأخطر في Return Cancel

خطأ:

```csharp
ReturnRequestNumber =
    originalReturn.MessageId;
```

خطأ:

```csharp
ReturnRequestNumber =
    originalReturn.StatusQueryIdentifier;
```

الصحيح:

```csharp
ReturnRequestNumber =
    originalReturn.ReturnRequestNumber;
```

إذا كانت القيمة مختلفة، قد تظهر رسالة مثل:

```text
No PENDING return found
```

---

## 50. إرسال Return Cancel

```csharp
EpttsResult<SubmissionResponse> submission;

using (EpttsClient client =
    new EpttsClient(options))
{
    submission =
        await client.CancelReturnAsync(
            request,
            cancellationToken);
}
```

بعد القبول، احفظ عملية إلغاء مستقلة بحالة `Pending`.

لا تستبدل سجل المرتجع الأصلي بسجل الإلغاء.

---

## 51. ما الذي يحفظه ERP في Return Cancel؟

```text
OriginalReturnId
SGTIN
DestinationGln
ReturnRequestNumber الأصلي
ErpReturnCancelDocumentId
EventDateTime
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
RawRequest
RawResponse
LocalStatus = Pending
```

---

## 52. ما بعد نجاح Return Cancel

بعد `S - Successful`:

1. حدث سجل الإلغاء إلى `Successful`.
2. حدث حالة المرتجع الأصلي داخل ERP.
3. نفذ `VerifyProduct`.

الحالة المتوقعة عادة:

```text
Status = active
CurrentGln = PharmacyGln
```

إذا لم تعد العبوة إلى الحالة المتوقعة، سجل الحالة للمراجعة ولا تعِد الإلغاء.

---

## 53. أخطاء Return Cancel الشائعة

- استخدام `MessageId` بدل `ReturnRequestNumber`.
- استخدام رقم مرتجع جديد.
- استخدام `DestinationGln` مختلف.
- استخدام `SGTIN` مختلف.
- المرتجع لم يعد قابلًا للإلغاء.
- الفرع استلم المرتجع.
- تم إلغاء المرتجع سابقًا.
- محاولة إرسال الإلغاء مرتين.

---

# القسم السابع: متابعة النتيجة النهائية

## 54. الاستعلام مرة واحدة

بعد حفظ `StatusQueryIdentifier`:

```csharp
EpttsResult<MessageStatusResponse> statusResult;

using (EpttsClient client =
    new EpttsClient(options))
{
    statusResult =
        await client.GetMessageStatusAsync(
            statusQueryIdentifier,
            cancellationToken);
}
```

---

## 55. قرار ERP بعد Message Status

```csharp
if (!statusResult.IsSuccess ||
    statusResult.Data == null)
{
    /*
     * لا تعتبر العملية Failed نهائيًا.
     * احتفظ بها Pending أو NeedsReview.
     */

    return;
}

if (statusResult.Data.IsSuccessful)
{
    /*
     * Successful.
     */

    return;
}

if (statusResult.Data.IsApplicationError)
{
    string? errorMessage =
        statusResult.Data.FirstErrorMessage;

    /*
     * Failed.
     */

    return;
}

/*
 * Pending.
 */
```

---

## 56. استخدام WaitForFinalStatusAsync في الاختبارات

```csharp
EpttsResult<MessageStatusResponse> finalResult;

using (EpttsClient client =
    new EpttsClient(options))
{
    finalResult =
        await client.WaitForFinalStatusAsync(
            statusQueryIdentifier,
            pollingInterval:
                TimeSpan.FromSeconds(3),
            maximumWaitTime:
                TimeSpan.FromSeconds(30),
            cancellationToken:
                cancellationToken);
}
```

هذه الطريقة مناسبة للمحاكي وUAT والانتظار القصير.

داخل ERP الحقيقي، استخدم Job للمتابعة طويلة المدى.

---

# القسم الثامن: منع التكرار ونتائج غير مؤكدة

## 57. منع العملية المكررة

قبل الإرسال، افحص وجود عملية محلية:

```text
نفس OperationType
ونفس SGTIN أو SSCC
وLocalStatus = Pending
```

مثال Pseudo-code:

```csharp
bool hasPending =
    repository.HasPendingOperation(
        operationType,
        sgtin);

if (hasPending)
{
    /*
     * امنع إرسال عملية جديدة.
     */

    return;
}
```

`repository` والدالة `HasPendingOperation` يجب أن ينشئهما فريق ERP.

---

## 58. التعامل مع Timeout أثناء الإرسال

إذا حدث:

```text
REQUEST_TIMEOUT
```

لا تفترض أن الرسالة لم تصل.

الإجراء:

```text
احفظ NeedsReview
→ احفظ RawRequest
→ استخرج instanceIdentifier إذا توفر
→ نفذ Message Status
→ لا تعِد الإرسال قبل التأكد
```

ينطبق ذلك على جميع عمليات هذا الجزء.

---

## 59. التعامل مع Cancellation أثناء الإرسال

إلغاء انتظار التطبيق لا يلغي الرسالة داخل EPTTS.

إذا بدأ الإرسال ثم أُلغي محليًا:

```text
لا تعِد العملية فورًا
→ راجع الرسالة السابقة
→ استخدم المعرفات المتاحة
```

---

## 60. تعطيل زر الإرسال

داخل UI:

```text
اضغط المستخدم Send
→ عطّل الزر فورًا
→ نفذ الطلب
→ لا تسمح بنقرة ثانية
```

لكن تعطيل الزر وحده لا يكفي. يجب أيضًا وجود منع تكرار في قاعدة البيانات أو Service، لأن التطبيق قد يعاد تشغيله أو تُفتح أكثر من شاشة.

---

# القسم التاسع: قالب كامل لأي عملية

## 61. قالب موحد

```csharp
try
{
    /*
     * 1. تحقق من العبوة عند الحاجة.
     */

    EpttsResult<VerifyProductResponse> verification =
        await client.VerifyProductAsync(
            sgtin,
            cancellationToken);

    if (!verification.IsSuccess ||
        verification.Data == null ||
        !verification.Data.Verified ||
        verification.Data.Pack == null)
    {
        return;
    }

    /*
     * 2. طبق شروط العملية محليًا.
     */

    /*
     * 3. امنع وجود عملية Pending مكررة.
     */

    /*
     * 4. أنشئ Request المناسب.
     */

    EpttsResult<SubmissionResponse> submission =
        await client.SomeOperationAsync(
            request,
            cancellationToken);

    if (!submission.IsSuccess ||
        submission.Data == null)
    {
        /*
         * احفظ الفشل أو NeedsReview حسب الحالة.
         */

        return;
    }

    if (!submission.Data.CanQueryFinalStatus ||
        string.IsNullOrWhiteSpace(
            submission.Data.StatusQueryIdentifier))
    {
        /*
         * احفظ NeedsReview.
         */

        return;
    }

    string identifier =
        submission.Data.StatusQueryIdentifier;

    /*
     * 5. احفظ العملية Pending فورًا.
     */

    /*
     * 6. تابع النتيجة باستخدام Job أو MsgStatusQuery.
     */
}
catch (EpttsConfigurationException exception)
{
    /* إعدادات غير صحيحة. */
}
catch (EpttsValidationException exception)
{
    /* مدخلات غير صحيحة. */
}
catch (EpttsException exception)
{
    /* خطأ من المكتبة. */
}
catch (Exception exception)
{
    /* خطأ غير متوقع في ERP. */
}
```

`SomeOperationAsync` اسم توضيحي وليس دالة موجودة. استبدله بالدالة الفعلية للعملية.

---

## 62. قائمة اختبار العمليات في Staging

### Receiving

```text
[ ] استلام SSCC صحيح
[ ] استلام قائمة SGTINs
[ ] SourceGln غير صحيح
[ ] EpcList فارغة
[ ] تكرار Receiving
[ ] S - Successful
[ ] E - Application Error
```

### Full Pack Dispensing

```text
[ ] عبوة active ومملوكة للصيدلية
[ ] عبوة dispensed
[ ] عبوة مملوكة لجهة أخرى
[ ] عبوة مستدعاة
[ ] تكرار الصرف
[ ] S - Successful
[ ] VerifyProduct بعد النجاح
```

### Partial Dispensing

```text
[ ] عبوة active
[ ] عبوة partially_dispensed
[ ] Quantity صحيحة
[ ] Quantity = 0
[ ] Quantity سالبة
[ ] منتج لا يسمح بالصرف الجزئي
[ ] S - Successful
```

### Dispense Cancel

```text
[ ] عبوة dispensed
[ ] عبوة active
[ ] عبوة partially_dispensed
[ ] إلغاء مكرر
[ ] S - Successful
[ ] VerifyProduct بعد النجاح
```

### Return

```text
[ ] عبوة active
[ ] DestinationGln صحيح
[ ] ReturnRequestNumber فريد
[ ] ReturnRequestNumber مكرر
[ ] S - Successful
[ ] حفظ رقم المرتجع الأصلي
```

### Return Cancel

```text
[ ] نفس SGTIN
[ ] نفس DestinationGln
[ ] نفس ReturnRequestNumber
[ ] استخدام MessageId بالخطأ
[ ] مرتجع لم يعد Pending
[ ] إلغاء مكرر
[ ] S - Successful
[ ] VerifyProduct بعد النجاح
```

---

## 63. قائمة إتمام الجزء الخامس

```text
[ ] أفهم أن عمليات هذا الجزء تغير حالة أو مسار العبوة
[ ] لا أعتبر HTTP 202 نجاحًا نهائيًا
[ ] أنفذ VerifyProduct قبل العمليات التي تحتاجه
[ ] أمنع عملية Pending مكررة
[ ] أحفظ RequestInstanceIdentifier
[ ] أحفظ MessageId
[ ] أحفظ StatusQueryIdentifier
[ ] أحفظ RawRequest وRawResponse
[ ] أحفظ العملية Pending فور قبولها
[ ] أتابع Message Status
[ ] أعتمد النجاح عند S - Successful فقط
[ ] أسجل E - Application Error كـ Failed
[ ] لا أعيد الإرسال تلقائيًا بعد Timeout
[ ] أستخدم SSCC أو SGTINs بصورة صحيحة في Receiving
[ ] لا أرسل بيانات مريض مباشرة
[ ] أفهم معنى Quantity في Partial Dispensing
[ ] لا أستخدم Dispense Cancel لإلغاء Partial Dispensing دون مواصفة
[ ] أحفظ ReturnRequestNumber الأصلي
[ ] أستخدم القيم الأصلية نفسها في Return Cancel
[ ] لا أستخدم MessageId بدل ReturnRequestNumber
[ ] أنفذ VerifyProduct بعد النجاح عند الحاجة
[ ] أختبر جميع السيناريوهات في Staging
```

---

## 64. خلاصة الجزء الخامس

جميع العمليات تتبع النمط نفسه:

```text
تحقق
→ امنع التكرار
→ أنشئ Request
→ أرسل مرة واحدة
→ احفظ Pending
→ تابع StatusQueryIdentifier
→ Successful أو Failed
→ VerifyProduct بعد النجاح عند الحاجة
→ حدث ERP
```

ولا تطبق أثر النجاح النهائي قبل وصول الرسالة إلى:

```text
S - Successful
```

أما:

```text
HTTP 202 / I001
```

فيعني فقط أن الرسالة قُبلت للمعالجة.
