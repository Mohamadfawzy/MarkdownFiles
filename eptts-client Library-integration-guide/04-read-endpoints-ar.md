# الجزء الرابع: Endpoints القراءة

> يشرح هذا الجزء Endpoints التي تقرأ بيانات من EPTTS دون إنشاء عملية جديدة تغير حالة العبوة. يبدأ الجزء بـ `VerifyProductAsync`، ثم يشرح متابعة الرسائل باستخدام `GetMessageStatusAsync` و`WaitForFinalStatusAsync`. كما يوضح حالة `GetInvoices` و`Export SSCC Contents` دون افتراض دوال غير موجودة في النسخة الحالية من المكتبة.

---

## 1. ما المقصود بـ Endpoints القراءة؟

Endpoints القراءة تستعلم عن بيانات موجودة في EPTTS دون إنشاء حدث تجاري جديد مثل الصرف أو الاستلام أو المرتجع.

تشمل الدوال المتاحة حاليًا:

```text
VerifyProductAsync
GetMessageStatusAsync
WaitForFinalStatusAsync
```

يستخدم ERP هذه الدوال من أجل:

- التحقق من عبوة قبل تنفيذ عملية عليها.
- قراءة حالة العبوة وملكيتها الحالية.
- معرفة هل العبوة مستدعاة.
- قراءة تنبيهات العبوة.
- متابعة النتيجة النهائية لعملية سبق إرسالها.

### هل يمكن إعادة محاولة عملية القراءة؟

بما أن عملية القراءة لا تغير حالة العبوة، يمكن إعادة المحاولة عند فشل اتصال مؤقت وفق سياسة ERP.

لكن يجب أن تكون إعادة المحاولة:

- محدودة العدد.
- متباعدة زمنيًا.
- قابلة للإلغاء.
- مسجلة لأغراض الدعم عند استمرار الفشل.

---

## 2. Namespaces المطلوبة

أضف Namespaces التالية بحسب المثال المستخدم:

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Eptts.Client.Clients;
using Eptts.Client.Configuration;
using Eptts.Client.Exceptions;
using Eptts.Client.Responses;
```

لا تحتاج إلى `Eptts.Client.Requests` عند استخدام Overload البسيط من `VerifyProductAsync` الذي يستقبل `string` مباشرة.

---

# القسم الأول: VerifyProduct

## 3. الغرض من `VerifyProductAsync`

تستخدم `VerifyProductAsync` للتحقق من عبوة دوائية وقراءة بياناتها الحالية من EPTTS.

العملية:

```text
قراءة فقط
```

ولا تقوم بـ:

- صرف العبوة.
- إلغاء صرفها.
- نقل ملكيتها.
- استلامها.
- إرسالها كمرتجع.
- إلغاء مرتجع.

يجب استخدام `VerifyProduct` قبل العمليات التي تتطلب معرفة حالة العبوة الحالية.

---

## 4. المدخل الأساسي: Product ID أو SGTIN

الاستخدام المعتاد يعتمد على `SGTIN` كامل:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
```

مثال:

```csharp
string sgtin =
    "urn:epc:id:sgtin:629000999.0001.SBX104047";
```

لا تمرر:

- `GTIN` فقط.
- `SSCC` عندما تريد التحقق من عبوة منفردة.
- رقم الصنف الداخلي في ERP.
- `ItemID`.
- `ItemRefNo`.
- نصًا فارغًا.

استخدم المعرف الذي تقبله EPTTS وفق بيانات العبوة والبيئة المستخدمة.

---

## 5. أبسط استدعاء لـ `VerifyProductAsync`

يفترض المثال التالي أن `options` تم إنشاؤها والتحقق منها كما ورد في الجزء الأول:

```csharp
using (EpttsClient client =
    new EpttsClient(options))
{
    EpttsResult<VerifyProductResponse> result =
        await client.VerifyProductAsync(
            sgtin,
            cancellationToken);
}
```

المتغيران التاليان يجب أن يوفرهما تطبيق ERP:

```csharp
sgtin
cancellationToken
```

أما `options` فهي إعدادات الاتصال المحملة من إعدادات الصيدلية الحالية.

---

## 6. مثال كامل قابل للاستخدام

```csharp
public static async Task<
    EpttsResult<VerifyProductResponse>>
    VerifyPackAsync(
        EpttsOptions options,
        string sgtin,
        CancellationToken cancellationToken)
{
    if (options == null)
    {
        throw new ArgumentNullException(
            nameof(options));
    }

    if (string.IsNullOrWhiteSpace(sgtin))
    {
        throw new ArgumentException(
            "SGTIN is required.",
            nameof(sgtin));
    }

    using (EpttsClient client =
        new EpttsClient(options))
    {
        return await client.VerifyProductAsync(
            sgtin.Trim(),
            cancellationToken);
    }
}
```

هذه الدالة:

- تتحقق من وجود الإعدادات.
- تتحقق من وجود `SGTIN`.
- تنشئ `EpttsClient`.
- تنفذ الطلب.
- تعيد النتيجة إلى ERP دون عرض رسائل UI.

---

## 7. القراءة الصحيحة لنتيجة VerifyProduct

استخدم الترتيب التالي:

```csharp
EpttsResult<VerifyProductResponse> result =
    await VerifyPackAsync(
        options,
        sgtin,
        cancellationToken);

if (!result.IsSuccess)
{
    /*
     * فشل تقني أو HTTP أو Timeout أو Cancellation.
     * لا تكمل إلى عملية تغير حالة العبوة.
     */

    return;
}

if (result.Data == null)
{
    /*
     * الاستجابة المنظمة فارغة.
     * سجل الحالة للمراجعة.
     */

    return;
}

if (!result.Data.Verified)
{
    /*
     * طلب HTTP نجح، لكن العبوة لم يتم التحقق منها.
     */

    return;
}

if (result.Data.Pack == null)
{
    /*
     * لا توجد بيانات Pack كافية لاتخاذ القرار.
     */

    return;
}

string packStatus =
    result.Data.Pack.Status;

string currentGln =
    result.Data.Pack.CurrentGln;

bool isRecalled =
    result.Data.Pack.IsRecalled;
```

---

## 8. الفرق بين `IsSuccess` و`Verified`

### `result.IsSuccess`

توضح هل نجح طلب HTTP واستطاعت المكتبة معالجة الاستجابة.

### `result.Data.Verified`

توضح هل استطاعت EPTTS التحقق من العبوة المطلوبة.

قد تكون النتيجة:

```text
IsSuccess = true
Verified = false
```

ومعناها:

```text
الطلب نجح تقنيًا
لكن العبوة لم يتم التحقق منها
```

لذلك لا يكفي:

```csharp
if (result.IsSuccess)
{
    // لا تفترض أن العبوة صالحة للعملية.
}
```

يجب أيضًا فحص:

```csharp
result.Data != null
result.Data.Verified
result.Data.Pack != null
```

---

## 9. الخصائص المهمة في `VerifyProductResponse`

بعد التأكد من وجود `result.Data`، يمكن قراءة خصائص مثل:

```csharp
result.Data.Verified
result.Data.Sgtin
result.Data.VerifiedAt
result.Data.Pack
result.Data.Product
result.Data.Alerts
```

قد تختلف القيم المتاحة حسب الاستجابة الفعلية والبيانات الموجودة في البيئة.

---

## 10. بيانات Pack

بعد التأكد من:

```csharp
result.Data.Pack != null
```

يمكن قراءة:

```csharp
string? returnedSgtin =
    result.Data.Pack.Sgtin;

string? gtin =
    result.Data.Pack.Gtin;

string? serial =
    result.Data.Pack.Serial;

string? batchNumber =
    result.Data.Pack.BatchNumber;

var expiryDate =
    result.Data.Pack.ExpiryDate;

string? status =
    result.Data.Pack.Status;

string? currentGln =
    result.Data.Pack.CurrentGln;

bool isRecalled =
    result.Data.Pack.IsRecalled;

string? parentSscc =
    result.Data.Pack.ParentSscc;

string? manufacturerGln =
    result.Data.Pack.ManufacturerGln;
```

لا تفترض أن جميع القيم غير فارغة. افحص القيم الاختيارية قبل استخدامها أو عرضها.

---

## 11. بيانات Product

بعد التأكد من:

```csharp
result.Data.Product != null
```

يمكن قراءة بيانات مثل:

```csharp
string? productName =
    result.Data.Product.Name;

string? dosageForm =
    result.Data.Product.DosageForm;

string? strength =
    result.Data.Product.Strength;

string? manufacturerName =
    result.Data.Product.ManufacturerName;
```

استخدم هذه البيانات للعرض أو الربط والمراجعة، لكن لا تستبدل Master Data الخاصة بـ ERP تلقائيًا دون سياسة مزامنة واضحة.

---

## 12. قراءة Alerts

قد تعيد الاستجابة قائمة تنبيهات:

```csharp
if (result.Data.Alerts != null &&
    result.Data.Alerts.Count > 0)
{
    foreach (string alert in result.Data.Alerts)
    {
        /*
         * اعرض التنبيه أو سجله وفق سياسة ERP.
         */
    }
}
```

لا تتجاهل `Alerts` قبل تنفيذ عملية أخرى على العبوة.

يمكن استخدام دالة مساعدة:

```csharp
private static bool HasAlerts(
    VerifyProductResponse response)
{
    return response.Alerts != null &&
           response.Alerts.Count > 0;
}
```

---

## 13. التحقق من ملكية العبوة للصيدلية الحالية

يمكن مقارنة `CurrentGln` مع `PharmacyGln`:

```csharp
private static bool IsOwnedByCurrentPharmacy(
    VerifyProductResponse response,
    string pharmacyGln)
{
    return response.Pack != null &&
           string.Equals(
               response.Pack.CurrentGln,
               pharmacyGln,
               StringComparison.Ordinal);
}
```

الاستخدام:

```csharp
bool ownedByCurrentPharmacy =
    IsOwnedByCurrentPharmacy(
        result.Data,
        options.PharmacyGln);
```

هذه مقارنة محلية مساعدة.

EPTTS يطبق قواعد الملكية والصلاحية النهائية عند معالجة العملية التجارية.

---

## 14. قرار مبدئي قبل Full Pack Dispensing

مثال قرار مساعد:

```csharp
bool canTryFullDispensing =
    result.IsSuccess &&
    result.Data != null &&
    result.Data.Verified &&
    result.Data.Pack != null &&
    string.Equals(
        result.Data.Pack.Status,
        "active",
        StringComparison.OrdinalIgnoreCase) &&
    string.Equals(
        result.Data.Pack.CurrentGln,
        options.PharmacyGln,
        StringComparison.Ordinal) &&
    !result.Data.Pack.IsRecalled &&
    (result.Data.Alerts == null ||
     result.Data.Alerts.Count == 0);
```

إذا كانت القيمة `false`، فلا ترسل عملية الصرف الكامل قبل مراجعة السبب.

إذا كانت `true`، فهذا يسمح بمحاولة العملية محليًا، لكنه لا يضمن قبول EPTTS النهائي.

---

## 15. قرار مبدئي قبل Partial Dispensing

```csharp
bool supportedStatus =
    string.Equals(
        result.Data.Pack.Status,
        "active",
        StringComparison.OrdinalIgnoreCase) ||
    string.Equals(
        result.Data.Pack.Status,
        "partially_dispensed",
        StringComparison.OrdinalIgnoreCase);

bool canTryPartialDispensing =
    result.IsSuccess &&
    result.Data != null &&
    result.Data.Verified &&
    result.Data.Pack != null &&
    supportedStatus &&
    string.Equals(
        result.Data.Pack.CurrentGln,
        options.PharmacyGln,
        StringComparison.Ordinal) &&
    !result.Data.Pack.IsRecalled &&
    (result.Data.Alerts == null ||
     result.Data.Alerts.Count == 0);
```

يجب أيضًا أن يعرف ERP أن المنتج يسمح بالصرف الجزئي، وأن `Quantity` تتوافق مع وحدة الصرف المعتمدة.

---

## 16. قرار مبدئي قبل Dispense Cancel

```csharp
bool canTryDispenseCancel =
    result.IsSuccess &&
    result.Data != null &&
    result.Data.Verified &&
    result.Data.Pack != null &&
    string.Equals(
        result.Data.Pack.Status,
        "dispensed",
        StringComparison.OrdinalIgnoreCase) &&
    string.Equals(
        result.Data.Pack.CurrentGln,
        options.PharmacyGln,
        StringComparison.Ordinal);
```

لا تستخدم مسار إلغاء الصرف الكامل لعبوة حالتها:

```text
partially_dispensed
```

إلا إذا كانت هناك مواصفة معتمدة تدعم ذلك.

---

## 17. قرار مبدئي قبل Return to Branch

```csharp
bool canTryReturn =
    result.IsSuccess &&
    result.Data != null &&
    result.Data.Verified &&
    result.Data.Pack != null &&
    string.Equals(
        result.Data.Pack.Status,
        "active",
        StringComparison.OrdinalIgnoreCase) &&
    string.Equals(
        result.Data.Pack.CurrentGln,
        options.PharmacyGln,
        StringComparison.Ordinal);
```

راجع أيضًا:

- التنبيهات.
- سياسة المنتج.
- بيانات الفرع المستقبل.
- عدم وجود عملية `Pending` سابقة على العبوة.

---

## 18. استخدام `CancellationToken` مع VerifyProduct

```csharp
using (CancellationTokenSource source =
    new CancellationTokenSource())
{
    EpttsResult<VerifyProductResponse> result =
        await client.VerifyProductAsync(
            sgtin,
            source.Token);
}
```

يمكن للواجهة استدعاء:

```csharp
source.Cancel();
```

لإيقاف انتظار المستخدم.

بما أن `VerifyProduct` عملية قراءة، يمكن إعادة الاستعلام لاحقًا عند الحاجة.

---

## 19. أخطاء VerifyProduct الشائعة

### SGTIN فارغ أو غير صحيح

قد ينتج `EpttsValidationException` قبل إرسال الطلب.

الإجراء:

```text
صحح المدخل
→ لا ترسل الطلب حتى يصبح المعرف صالحًا
```

### HTTP 401 أو 403

راجع:

- `IntegratorKey`.
- البيئة.
- `PharmacyGln`.
- الصلاحيات.

### `PACK_NOT_FOUND`

قد يعني:

- العبوة غير موجودة في البيئة الحالية.
- المعرف غير صحيح.
- استخدام بيانات إنتاج داخل Staging أو العكس.

### `Verified = false`

يعني أن طلب HTTP نجح، لكن العبوة لم يتم التحقق منها.

لا تكمل إلى عملية تغير حالة العبوة.

### `REQUEST_TIMEOUT`

يمكن إعادة محاولة عملية القراءة وفق سياسة محدودة، لأن `VerifyProduct` لا تغير حالة العبوة.

---

## 20. ما الذي يسجله ERP عند VerifyProduct؟

لا يلزم دائمًا إنشاء سجل تكامل دائم لكل استعلام قراءة، لكن يوصى بتسجيل الحد الأدنى عند الفشل أو لأغراض التدقيق:

```text
PharmacyGln
SGTIN
HttpStatusCode
ErrorCode
ErrorMessage
RawRequest
RawResponse
RequestedAt
RequestedBy
```

لا تسجل `IntegratorKey` أو Headers المصادقة.

---

# القسم الثاني: GetMessageStatusAsync

## 21. الغرض من `GetMessageStatusAsync`

تستخدم `GetMessageStatusAsync` للاستعلام مرة واحدة عن حالة رسالة سبق إرسالها.

يجب أن يكون ERP قد حصل مسبقًا على:

```csharp
StatusQueryIdentifier
```

من `SubmissionResponse` الخاصة بعملية مثل:

- `Receiving`.
- `Full Pack Dispensing`.
- `Partial Dispensing`.
- `Dispense Cancel`.
- `Return`.
- `Return Cancel`.

هذه الدالة لا تنشئ عملية تجارية جديدة.

هي تستعلم عن نتيجة رسالة موجودة.

---

## 22. الاستعلام مرة واحدة

```csharp
EpttsResult<MessageStatusResponse> result =
    await client.GetMessageStatusAsync(
        statusQueryIdentifier,
        cancellationToken);
```

المتغير:

```csharp
statusQueryIdentifier
```

يجب أن يكون القيمة التي أعادتها:

```csharp
submission.Data.StatusQueryIdentifier
```

لا تستخدم `MessageId` أو `ReturnRequestNumber` مكانه.

---

## 23. مثال كامل قابل للاستخدام

```csharp
public static async Task<
    EpttsResult<MessageStatusResponse>>
    QueryMessageStatusAsync(
        EpttsOptions options,
        string statusQueryIdentifier,
        CancellationToken cancellationToken)
{
    if (options == null)
    {
        throw new ArgumentNullException(
            nameof(options));
    }

    if (string.IsNullOrWhiteSpace(
            statusQueryIdentifier))
    {
        throw new ArgumentException(
            "StatusQueryIdentifier is required.",
            nameof(statusQueryIdentifier));
    }

    using (EpttsClient client =
        new EpttsClient(options))
    {
        return await client.GetMessageStatusAsync(
            statusQueryIdentifier.Trim(),
            cancellationToken);
    }
}
```

---

## 24. معالجة نتيجة Message Status

```csharp
EpttsResult<MessageStatusResponse> result =
    await QueryMessageStatusAsync(
        options,
        statusQueryIdentifier,
        cancellationToken);

if (!result.IsSuccess)
{
    /*
     * فشل استعلام الحالة نفسه.
     * لا تعتبر العملية الأصلية Failed نهائيًا.
     */

    return;
}

if (result.Data == null)
{
    /*
     * لا توجد بيانات منظمة.
     * احتفظ بالعملية Pending أو NeedsReview.
     */

    return;
}

if (result.Data.IsSuccessful)
{
    /*
     * العملية الأصلية نجحت نهائيًا.
     * حدث سجل ERP إلى Successful.
     */

    return;
}

if (result.Data.IsApplicationError)
{
    string? errorMessage =
        result.Data.FirstErrorMessage;

    /*
     * العملية الأصلية فشلت وظيفيًا.
     * حدث سجل ERP إلى Failed.
     */

    return;
}

/*
 * الحالة غير نهائية.
 * احتفظ بالعملية Pending.
 */
```

---

## 25. قراءة Processing Log

إذا كانت `LogList` متاحة:

```csharp
if (result.Data.LogList != null)
{
    foreach (var logItem in result.Data.LogList)
    {
        string? type =
            logItem.Type;

        string? message =
            logItem.Message;

        /*
         * اعرض أو سجل هذه التفاصيل للدعم.
         */
    }
}
```

لا تعتمد على `LogList` وحدها لتحديد النجاح.

استخدم الخصائص المنظمة:

```csharp
IsFinal
IsSuccessful
IsApplicationError
```

---

## 26. متى يستخدم ERP الاستعلام مرة واحدة؟

يفضل استخدام `GetMessageStatusAsync` داخل ERP الحقيقي عندما:

- تُحفظ العمليات المعلقة في قاعدة البيانات.
- يوجد `Job` أو `Timer` يعمل دوريًا.
- لا تريد إبقاء شاشة المستخدم في حالة انتظار.
- تريد التحكم في عدد الاستعلامات وتوقيتها.
- تحتاج إلى استكمال المتابعة بعد إعادة تشغيل التطبيق أو الخادم.

التسلسل الموصى به:

```text
إرسال العملية
→ حفظ Pending في قاعدة البيانات
→ إنهاء طلب المستخدم
→ Job يقرأ العمليات Pending
→ GetMessageStatusAsync
→ تحديث الحالة
```

---

## 27. نموذج Job توضيحي

الكود التالي `Pseudo-code` توضيحي، لأن Repository تابع لنظام ERP وليس للمكتبة:

```csharp
public async Task ProcessPendingOperationsAsync(
    CancellationToken cancellationToken)
{
    IReadOnlyCollection<ErpEpttsOperation> operations =
        repository.GetPendingOperations();

    foreach (ErpEpttsOperation operation in operations)
    {
        EpttsOptions options =
            LoadOptionsForPharmacy(
                operation.PharmacyId);

        using (EpttsClient client =
            new EpttsClient(options))
        {
            EpttsResult<MessageStatusResponse> result =
                await client.GetMessageStatusAsync(
                    operation.StatusQueryIdentifier,
                    cancellationToken);

            if (!result.IsSuccess ||
                result.Data == null)
            {
                repository.SaveStatusQueryFailure(
                    operation.Id,
                    result.ErrorCode,
                    result.ErrorMessage,
                    result.RawResponse);

                continue;
            }

            if (result.Data.IsSuccessful)
            {
                repository.MarkSuccessful(
                    operation.Id,
                    result.RawResponse);

                continue;
            }

            if (result.Data.IsApplicationError)
            {
                repository.MarkFailed(
                    operation.Id,
                    result.Data.FirstErrorMessage,
                    result.RawResponse);

                continue;
            }

            repository.KeepPending(
                operation.Id,
                result.Data.MessageStatus,
                result.RawResponse);
        }
    }
}
```

الأسماء التالية أمثلة من ERP وليست جزءًا من المكتبة:

```text
ErpEpttsOperation
repository
LoadOptionsForPharmacy
GetPendingOperations
MarkSuccessful
MarkFailed
KeepPending
```

---

## 28. أخطاء Message Status الشائعة

### Identifier فارغ أو غير صحيح

قد ينتج `EpttsValidationException` محليًا.

### HTTP 401 أو 403

راجع أن الجهة المستعلمة مخولة بقراءة الرسالة، وأن البيئة والمفتاح والصيدلية صحيحة.

### HTTP 404

قد يعني أن المعرف غير موجود في البيئة الحالية.

راجع:

- `StatusQueryIdentifier`.
- البيئة.
- عدم استخدام `MessageId` بدل المعرف الصحيح.

### Timeout

فشل استعلام الحالة لا يعني فشل العملية الأصلية.

احتفظ بالعملية `Pending` وأعد الاستعلام لاحقًا.

### Cancellation

إلغاء استعلام الحالة يوقف الانتظار المحلي فقط.

لا يلغي العملية الأصلية.

---

# القسم الثالث: WaitForFinalStatusAsync

## 29. الغرض من `WaitForFinalStatusAsync`

تستخدم `WaitForFinalStatusAsync` لتنفيذ Polling تلقائي حتى:

- تصل الرسالة إلى نتيجة نهائية.
- تنتهي مدة الانتظار القصوى.
- يتم إلغاء الانتظار محليًا.
- يحدث فشل يمنع الاستمرار.

هذه الدالة مناسبة لـ:

- `ErpSimulator`.
- الاختبارات.
- UAT.
- شاشة تسمح بانتظار قصير.

داخل ERP الحقيقي، يفضل غالبًا استخدام `GetMessageStatusAsync` من خلال Job.

---

## 30. أبسط استدعاء للانتظار التلقائي

```csharp
EpttsResult<MessageStatusResponse> result =
    await client.WaitForFinalStatusAsync(
        statusQueryIdentifier,
        pollingInterval:
            TimeSpan.FromSeconds(3),
        maximumWaitTime:
            TimeSpan.FromSeconds(30),
        cancellationToken:
            cancellationToken);
```

### `pollingInterval`

المدة بين كل استعلام والذي يليه.

### `maximumWaitTime`

أقصى مدة ينتظرها التطبيق داخل هذا الاستدعاء.

### `cancellationToken`

يسمح للتطبيق أو المستخدم بإيقاف الانتظار محليًا.

---

## 31. قواعد قيم Polling

يجب أن تكون:

```csharp
pollingInterval > TimeSpan.Zero
```

و:

```csharp
maximumWaitTime > TimeSpan.Zero
```

ويفضل أن تكون:

```csharp
pollingInterval <= maximumWaitTime
```

مثال مناسب للاختبار:

```csharp
TimeSpan pollingInterval =
    TimeSpan.FromSeconds(3);

TimeSpan maximumWaitTime =
    TimeSpan.FromSeconds(30);
```

لا تستخدم Polling سريعًا جدًا دون حاجة، لأنه يزيد عدد الطلبات على الخدمة.

---

## 32. مثال كامل قابل للاستخدام

```csharp
public static async Task<
    EpttsResult<MessageStatusResponse>>
    WaitForFinalResultAsync(
        EpttsOptions options,
        string statusQueryIdentifier,
        TimeSpan pollingInterval,
        TimeSpan maximumWaitTime,
        CancellationToken cancellationToken)
{
    if (options == null)
    {
        throw new ArgumentNullException(
            nameof(options));
    }

    if (string.IsNullOrWhiteSpace(
            statusQueryIdentifier))
    {
        throw new ArgumentException(
            "StatusQueryIdentifier is required.",
            nameof(statusQueryIdentifier));
    }

    if (pollingInterval <= TimeSpan.Zero)
    {
        throw new ArgumentOutOfRangeException(
            nameof(pollingInterval));
    }

    if (maximumWaitTime <= TimeSpan.Zero)
    {
        throw new ArgumentOutOfRangeException(
            nameof(maximumWaitTime));
    }

    if (pollingInterval > maximumWaitTime)
    {
        throw new ArgumentException(
            "PollingInterval must not be greater " +
            "than MaximumWaitTime.");
    }

    using (EpttsClient client =
        new EpttsClient(options))
    {
        return await client.WaitForFinalStatusAsync(
            statusQueryIdentifier.Trim(),
            pollingInterval,
            maximumWaitTime,
            cancellationToken);
    }
}
```

---

## 33. قراءة نتيجة الانتظار التلقائي

استخدم نفس قواعد `GetMessageStatusAsync`:

```csharp
if (!result.IsSuccess)
{
    /*
     * راجع ErrorCode وErrorMessage.
     * لا تعتبر العملية الأصلية Failed تلقائيًا.
     */

    return;
}

if (result.Data == null)
{
    /*
     * النتيجة غير مؤكدة.
     */

    return;
}

if (result.Data.IsSuccessful)
{
    /* Successful */
    return;
}

if (result.Data.IsApplicationError)
{
    /* Failed */
    return;
}

/* Pending */
```

---

## 34. انتهاء `maximumWaitTime`

إذا انتهت مدة الانتظار قبل ظهور نتيجة نهائية، فلا تعتبر العملية فاشلة.

معنى ذلك:

```text
توقف التطبيق عن Polling بعد المدة المحددة
```

الإجراء الصحيح:

```text
احتفظ بالعملية Pending
→ احتفظ بـ StatusQueryIdentifier
→ أعد GetMessageStatusAsync لاحقًا
```

لا تعِد إرسال العملية الأصلية بسبب انتهاء مدة Polling.

---

## 35. إلغاء الانتظار

يمكن إلغاء الانتظار:

```csharp
source.Cancel();
```

لكن الإلغاء يعني:

```text
إيقاف انتظار التطبيق
```

ولا يعني:

```text
إلغاء عملية Receiving أو Dispensing أو Return
```

احتفظ بمعرف الرسالة واستعلم عنها لاحقًا.

---

## 36. اختيار الدالة المناسبة

استخدم `GetMessageStatusAsync` عندما:

- تعمل داخل Job.
- تريد استعلامًا واحدًا.
- تحفظ العمليات في قاعدة البيانات.
- تحتاج إلى متابعة طويلة المدى.
- لا تريد إبقاء المستخدم منتظرًا.

استخدم `WaitForFinalStatusAsync` عندما:

- تختبر المكتبة.
- تنفذ UAT.
- تنتظر مدة قصيرة داخل شاشة تجربة.
- تريد Polling جاهزًا داخل استدعاء واحد.

---

# القسم الرابع: GetInvoices

## 37. حالة GetInvoices في النسخة الحالية

وفق واجهة المكتبة المستخدمة في هذا الدليل، لم يتم اعتماد دالة عامة نهائية لـ `GetInvoices` ضمن قائمة الدوال الحالية.

لذلك لا يجب كتابة أو نسخ كود يفترض وجود دالة مثل:

```csharp
client.GetInvoicesAsync(...)
```

ما لم تظهر الدالة فعليًا داخل `IntelliSense` في نسخة `Eptts.Client.dll` المستخدمة.

لا تنشئ HTTP Request يدويًا من مشروع ERP لتجاوز المكتبة، لأن ذلك يسبب مسارين مختلفين للتكامل ويزيد مخاطر اختلاف Headers والتحقق ومعالجة الأخطاء.

---

## 38. متى يضاف GetInvoices إلى الدليل؟

يضاف القسم التنفيذي بعد اكتمال الخطوات التالية:

```text
1. اعتماد مواصفة Endpoint
2. تحديد Request fields
3. تحديد Response structure
4. تنفيذ الدالة داخل Eptts.Client
5. كتابة Unit Tests
6. استخراج DLL جديدة
7. اختبار الدالة من ErpSimulator
8. توثيق التوقيع الفعلي
```

بعدها يجب أن يحتوي القسم على:

- اسم الدالة الفعلي.
- Namespace الخاص بـ Request وResponse.
- الفلاتر المتاحة.
- Pagination إن وجدت.
- مثال كامل.
- طريقة قراءة النتيجة.
- الأخطاء المتوقعة.

حتى ذلك الوقت، اعتبر `GetInvoices`:

```text
ميزة مخططة وليست جزءًا من واجهة الاستخدام المعتمدة
```

---

# القسم الخامس: Export SSCC Contents

## 39. حالة Export SSCC Contents في النسخة الحالية

وفق واجهة المكتبة المستخدمة في هذا الدليل، لم يتم اعتماد دالة عامة نهائية لتصدير محتويات `SSCC` ضمن قائمة الدوال الحالية.

لا تفترض توقيعًا مثل:

```csharp
client.ExportSsccContentsAsync(...)
```

ولا تعتمد على اسم مؤقت قبل ظهوره فعليًا في `IntelliSense` وXML Documentation.

---

## 40. ما الذي يجب تثبيته قبل توثيق Export SSCC؟

يجب تحديد:

- هل المدخل `SSCC` واحد فقط.
- هل النتيجة قائمة `SGTINs` أم ملف أم استجابة منظمة.
- صيغة التصدير.
- Pagination إن وجدت.
- حدود عدد العناصر.
- طريقة التعامل مع SSCC فارغ أو غير موجود.
- هل Endpoint قراءة فقط.
- شكل Request وResponse الفعليين.

بعد تنفيذ الدالة واختبارها، يضاف مثال مطابق تمامًا للـ DLL.

حتى ذلك الوقت، لا تنسخ كودًا تجريبيًا أو تنشئ HTTP Client منفصلًا داخل ERP.

---

# القسم السادس: نمط الاستخدام داخل ERP

## 41. خدمة موحدة لعمليات القراءة

يمكن لفريق ERP إنشاء Service خاصة به تجمع عمليات القراءة.

المثال التالي كود ERP اختياري، وليس Class توفرها المكتبة:

```csharp
public sealed class ErpEpttsReadService
{
    private readonly EpttsOptions _options;

    public ErpEpttsReadService(
        EpttsOptions options)
    {
        _options =
            options ??
            throw new ArgumentNullException(
                nameof(options));
    }

    public async Task<
        EpttsResult<VerifyProductResponse>>
        VerifyProductAsync(
            string sgtin,
            CancellationToken cancellationToken)
    {
        using (EpttsClient client =
            new EpttsClient(_options))
        {
            return await client.VerifyProductAsync(
                sgtin,
                cancellationToken);
        }
    }

    public async Task<
        EpttsResult<MessageStatusResponse>>
        GetMessageStatusAsync(
            string statusQueryIdentifier,
            CancellationToken cancellationToken)
    {
        using (EpttsClient client =
            new EpttsClient(_options))
        {
            return await client.GetMessageStatusAsync(
                statusQueryIdentifier,
                cancellationToken);
        }
    }
}
```

ميزة هذا الأسلوب:

- يمنع تكرار إنشاء `EpttsOptions` داخل كل شاشة.
- يبقي كود المكتبة بعيدًا عن عناصر UI.
- يسهل الاختبار.
- يسمح بتوحيد التسجيل ومعالجة الأخطاء.

---

## 42. عدم ربط كود المكتبة مباشرة بالواجهة

لا تجعل منطق التكامل يعتمد على:

```text
TextBox
PasswordBox
MessageBox
Window
UserControl
```

الأفضل:

```text
واجهة المستخدم
→ تجمع المدخلات
→ تستدعي Service
→ Service تستخدم Eptts.Client
→ تعيد EpttsResult<T>
→ الواجهة تعرض النتيجة
```

هذا يجعل الكود قابلًا للاستخدام من:

- WPF.
- Windows Forms.
- API.
- Windows Service.
- Job.
- Unit Tests.

---

## 43. التسجيل الآمن لعمليات القراءة

عند التسجيل، استخدم:

```text
Endpoint Name
PharmacyGln
SGTIN أو StatusQueryIdentifier
HttpStatusCode
ErrorCode
ErrorMessage
RequestedAt
Elapsed Time
RawRequest عند الحاجة
RawResponse عند الحاجة
```

لا تسجل:

```text
IntegratorKey
Authorization Headers
كلمات المرور
بيانات شخصية مباشرة
```

---

## 44. قائمة اختبار VerifyProduct

اختبر الحالات التالية في Staging:

```text
[ ] SGTIN صحيح وموجود
[ ] SGTIN صحيح وغير موجود
[ ] SGTIN فارغ
[ ] صيغة SGTIN غير صحيحة
[ ] عبوة active
[ ] عبوة dispensed
[ ] عبوة partially_dispensed
[ ] عبوة returned أو في مسار مرتجع
[ ] عبوة غير مملوكة للصيدلية
[ ] عبوة مستدعاة إن توفرت بيانات اختبار
[ ] نتيجة تحتوي على Alerts
[ ] IntegratorKey غير صحيح
[ ] PharmacyGln غير صحيح
[ ] HTTP Timeout
[ ] CancellationToken
```

تحقق دائمًا من:

```text
IsSuccess
Verified
Pack
Status
CurrentGln
IsRecalled
Alerts
RawRequest
RawResponse
```

---

## 45. قائمة اختبار Message Status

اختبر:

```text
[ ] StatusQueryIdentifier صحيح لرسالة Pending
[ ] رسالة وصلت إلى S - Successful
[ ] رسالة وصلت إلى E - Application Error
[ ] رسالة ما زالت غير نهائية
[ ] Identifier فارغ
[ ] Identifier غير موجود
[ ] Identifier من بيئة مختلفة
[ ] HTTP 401
[ ] HTTP 403
[ ] HTTP 404
[ ] Timeout
[ ] Cancellation
[ ] WaitForFinalStatusAsync بمدة قصيرة
[ ] WaitForFinalStatusAsync حتى نتيجة نهائية
```

---

## 46. أخطاء شائعة في Endpoints القراءة

### الاعتماد على `IsSuccess` فقط في VerifyProduct

الصحيح هو فحص:

```csharp
IsSuccess
Data
Verified
Pack
```

### استخدام MessageId مع GetMessageStatus

استخدم:

```csharp
StatusQueryIdentifier
```

الذي أعادته `SubmissionResponse`.

### اعتبار Timeout فشلًا نهائيًا للعملية الأصلية

فشل استعلام الحالة لا يغير نتيجة الرسالة الأصلية.

### انتظار المستخدم مدة طويلة

داخل ERP الحقيقي، استخدم Job للعمليات طويلة المدى بدل إبقاء الشاشة مفتوحة.

### تحليل RawResponse بدل Data

استخدم Classes المنظمة، واترك النص الخام للدعم والتشخيص.

### إنشاء GetInvoices يدويًا خارج المكتبة

انتظر تنفيذ الدالة المعتمدة داخل `Eptts.Client` حتى يبقى التكامل موحدًا.

---

## 47. قائمة إتمام الجزء الرابع

قبل الانتقال إلى العمليات التي تغير حالة العبوة، تأكد من:

```text
[ ] أفهم أن VerifyProduct عملية قراءة
[ ] أستخدم SGTIN الصحيح
[ ] أفحص IsSuccess
[ ] أفحص Data
[ ] أفحص Verified
[ ] أفحص Pack
[ ] أقرأ Status وCurrentGln وIsRecalled
[ ] أراجع Alerts
[ ] لا أعتمد على Status وحدها
[ ] أفهم التحقق من ملكية العبوة
[ ] أعرف كيفية استخدام CancellationToken
[ ] أعرف ما يجب تسجيله عند فشل VerifyProduct
[ ] أستخدم StatusQueryIdentifier مع GetMessageStatusAsync
[ ] لا أستخدم MessageId مكان StatusQueryIdentifier
[ ] أعرف الفرق بين IsSuccessful وIsApplicationError
[ ] أبقي الحالة غير النهائية Pending
[ ] أعرف متى أستخدم GetMessageStatusAsync
[ ] أعرف متى أستخدم WaitForFinalStatusAsync
[ ] أعرف أن انتهاء maximumWaitTime لا يعني فشل العملية
[ ] لا أعتبر Cancellation إلغاءً للعملية الأصلية
[ ] لا أستخدم GetInvoices قبل إضافتها للمكتبة
[ ] لا أستخدم Export SSCC قبل إضافتها للمكتبة
[ ] لا أسجل IntegratorKey
```

---

## 48. خلاصة الجزء الرابع

التسلسل الصحيح لاستخدام Endpoints القراءة:

```text
إنشاء EpttsOptions
→ Validate
→ إنشاء EpttsClient
→ تنفيذ عملية القراءة
→ فحص IsSuccess
→ فحص Data
→ قراءة النتيجة الوظيفية
→ تسجيل الفشل عند الحاجة
```

مع `VerifyProduct`:

```text
IsSuccess
→ Data
→ Verified
→ Pack
→ Status وCurrentGln وIsRecalled وAlerts
```

مع Message Status:

```text
StatusQueryIdentifier
→ GetMessageStatusAsync أو WaitForFinalStatusAsync
→ IsSuccessful أو IsApplicationError أو Pending
```

ولا تستخدم `GetInvoices` أو `Export SSCC Contents` قبل أن تظهر الدوال المعتمدة فعليًا في نسخة `Eptts.Client.dll` المستخدمة.
