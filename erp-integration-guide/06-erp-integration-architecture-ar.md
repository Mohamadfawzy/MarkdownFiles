# الجزء السادس: التكامل الصحيح داخل ERP

> يشرح هذا الجزء كيفية تحويل أمثلة استخدام `Eptts.Client` إلى تكامل إنتاجي داخل نظام ERP. لا يكتفي التكامل الصحيح باستدعاء Endpoint وعرض النتيجة، بل يحتاج إلى تخزين العمليات، ومنع التكرار، ومتابعة الرسائل المعلقة، وحماية الأسرار، وربط نجاح EPTTS بمستندات ERP والمخزون بطريقة آمنة وقابلة للتدقيق.

---

## 1. الهدف من هذا الجزء

بعد الأجزاء السابقة أصبح مطور ERP يعرف كيفية:

- إنشاء `EpttsOptions`.
- إنشاء `EpttsClient`.
- تنفيذ `VerifyProductAsync`.
- إرسال عمليات الاستلام والصرف والمرتجع.
- قراءة `SubmissionResponse`.
- متابعة `StatusQueryIdentifier`.
- قراءة النتيجة النهائية من `MessageStatusResponse`.

لكن استدعاء الدوال وحده لا يكفي لتكامل إنتاجي.

يجب أن يضمن ERP ما يلي:

```text
عدم ضياع العمليات المعلقة
عدم إرسال العملية مرتين
إمكانية استكمال المتابعة بعد إغلاق التطبيق
إمكانية معرفة ما أُرسل وما أُعيد
عدم كشف IntegratorKey
ربط كل رسالة بمستند ERP
اعتماد النجاح عند النتيجة النهائية فقط
```

---

## 2. المبادئ الأساسية للتكامل

التكامل الصحيح يعتمد على المبادئ التالية:

### المبدأ الأول: فصل الإرسال عن النتيجة النهائية

```text
إرسال العملية
≠
نجاح العملية
```

قد تقبل EPTTS الرسالة أولًا ثم تعالجها لاحقًا.

لذلك يجب الفصل بين:

```text
Submission
```

و:

```text
Final Message Status
```

### المبدأ الثاني: حفظ العملية قبل فقدان السياق

عند الحصول على `StatusQueryIdentifier` يجب حفظ العملية فورًا في قاعدة البيانات.

لا تعتمد على:

- متغير داخل الشاشة.
- `TextBox`.
- Clipboard.
- ذاكرة التطبيق.
- استمرار تشغيل جهاز المستخدم.

### المبدأ الثالث: عدم تكرار عمليات تغير الحالة

عند `Timeout` أو انقطاع الاتصال، لا تفترض أن الرسالة لم تصل.

تحقق من العملية السابقة قبل إرسال عملية جديدة.

### المبدأ الرابع: EPTTS لا تدير مستندات ERP

المكتبة ترسل وتقرأ بيانات EPTTS، لكنها لا:

- تعتمد فاتورة بيع.
- تزيد أو تخفض المخزون.
- تنشئ إذن مرتجع.
- تنشئ قيودًا محاسبية.
- تحفظ سجل التكامل في قاعدة بيانات ERP.

كل هذه مسؤولية ERP.

---

## 3. البنية المقترحة داخل ERP

يفضل تقسيم تكامل EPTTS إلى طبقات واضحة:

```text
ERP User Interface
        ↓
Application / Business Service
        ↓
EPTTS Integration Service
        ↓
Eptts.Client
        ↓
EPTTS API
```

وبالتوازي:

```text
EPTTS Integration Service
        ↓
Integration Repository
        ↓
ERP Database
```

ثم:

```text
Background Job
        ↓
Integration Repository
        ↓
GetMessageStatusAsync
        ↓
Update Local Status
```

---

## 4. مسؤولية كل طبقة

### واجهة المستخدم

مسؤولة عن:

- جمع المدخلات.
- عرض حالة العملية.
- تعطيل زر الإرسال أثناء التنفيذ.
- عرض رسالة مفهومة للمستخدم.

لا يجب أن تحتوي على منطق إنشاء Requests بالتفصيل أو منطق متابعة العمليات الطويلة.

### Application Service

مسؤولة عن:

- تطبيق قواعد ERP.
- تحميل المستند والصيدلية والمستخدم الحالي.
- التحقق من صلاحية المستند.
- منع وجود عملية معلقة متعارضة.
- استدعاء خدمة EPTTS.
- حفظ نتيجة التكامل.
- ربط النتيجة بمستند ERP.

### EPTTS Integration Service

مسؤولة عن:

- تحميل أو استقبال `EpttsOptions`.
- استخدام `EpttsClient`.
- إنشاء Request المناسب.
- إرسال العملية.
- إعادة `EpttsResult<T>` أو نتيجة تطبيقية منظمة.

### Repository

مسؤول عن:

- حفظ العمليات.
- قراءة العمليات `Pending`.
- منع التكرار.
- تحديث الحالة النهائية.
- حفظ تفاصيل الأخطاء والتدقيق.

### Background Job

مسؤول عن:

- قراءة العمليات المعلقة.
- تنفيذ `GetMessageStatusAsync`.
- تحديث `Pending` إلى `Successful` أو `Failed`.
- تطبيق سياسة Retry للاستعلام فقط.
- رفع العمليات القديمة أو غير القابلة للحسم إلى `NeedsReview`.

---

## 5. هيكل ملفات مقترح

هذه بنية مقترحة وليست إلزامية:

```text
EpttsIntegration
├── Configuration
│   ├── ErpEpttsSettings.cs
│   └── IEpttsSettingsProvider.cs
│
├── Services
│   ├── IEpttsIntegrationService.cs
│   ├── EpttsIntegrationService.cs
│   ├── IEpttsOperationService.cs
│   └── EpttsOperationService.cs
│
├── Persistence
│   ├── IEpttsOperationRepository.cs
│   └── EpttsOperationRepository.cs
│
├── Jobs
│   └── EpttsMessageStatusJob.cs
│
├── Models
│   ├── EpttsOperationRecord.cs
│   ├── EpttsLocalStatus.cs
│   └── EpttsOperationType.cs
│
└── Security
    └── IntegratorKeyProtector.cs
```

يمكن لفريق ERP تكييف الأسماء مع البنية الموجودة لديه.

---

## 6. نموذج إعدادات ERP

يمكن إنشاء Class محلية داخل ERP:

```csharp
public sealed class ErpEpttsSettings
{
    public string BaseUrl { get; set; } =
        string.Empty;

    public string IntegratorKey { get; set; } =
        string.Empty;

    public string PharmacyGln { get; set; } =
        string.Empty;

    public string PharmacySgln { get; set; } =
        string.Empty;

    public int TimeoutSeconds { get; set; } =
        60;
}
```

هذه Class ليست جزءًا من `Eptts.Client`.

يجب أن يوفر ERP طريقة لتحميلها للصيدلية الحالية.

---

## 7. Settings Provider مقترح

```csharp
public interface IEpttsSettingsProvider
{
    ErpEpttsSettings GetForPharmacy(
        int pharmacyId);
}
```

ثم يحول ERP الإعدادات إلى `EpttsOptions`:

```csharp
private static EpttsOptions CreateOptions(
    ErpEpttsSettings settings)
{
    if (settings == null)
    {
        throw new ArgumentNullException(
            nameof(settings));
    }

    EpttsOptions options =
        new EpttsOptions
        {
            BaseUrl =
                settings.BaseUrl,

            IntegratorKey =
                settings.IntegratorKey,

            PharmacyGln =
                settings.PharmacyGln,

            PharmacySgln =
                settings.PharmacySgln,

            Timeout =
                TimeSpan.FromSeconds(
                    settings.TimeoutSeconds)
        };

    options.Validate();

    return options;
}
```

---

## 8. تعدد الصيدليات والفروع

إذا كان ERP يخدم أكثر من صيدلية، فلا تستخدم إعدادًا عالميًا واحدًا دون ربطه بالصيدلية الحالية.

يجب أن يحتوي كل سجل إعدادات على الأقل على:

```text
PharmacyId
BaseUrl
IntegratorKey أو مرجع السر
PharmacyGln
PharmacySgln
IsActive
EnvironmentName
```

عند تنفيذ عملية:

```text
حدد الصيدلية من مستند ERP
→ حمّل إعداداتها
→ أنشئ EpttsOptions
→ نفذ العملية باسم الصيدلية نفسها
```

لا تعتمد على الصيدلية التي اختارها المستخدم في الشاشة فقط إذا كان المستند مربوطًا بصيدلية مختلفة.

---

## 9. بيئات Staging وProduction

يجب الفصل بين البيئات بوضوح.

مثال:

```text
EnvironmentName = Staging
BaseUrl = عنوان Staging
IntegratorKey = مفتاح Staging

EnvironmentName = Production
BaseUrl = عنوان Production
IntegratorKey = مفتاح Production
```

لا تستخدم:

- مفتاح Production مع Staging.
- بيانات عبوات Production داخل Staging.
- قاعدة عمليات مشتركة دون تمييز البيئة.

أضف `EnvironmentName` إلى سجل التكامل حتى يمكن معرفة البيئة التي استقبلت الرسالة.

---

# القسم الأول: نموذج بيانات العمليات

## 10. لماذا نحتاج جدول تكامل؟

يجب أن يستطيع ERP الإجابة عن الأسئلة التالية:

- ما العملية التي أُرسلت؟
- من أرسلها؟
- متى أُرسلت؟
- لأي صيدلية؟
- على أي `SGTIN` أو `SSCC`؟
- ما `StatusQueryIdentifier`؟
- هل العملية ما زالت `Pending`؟
- ما النتيجة النهائية؟
- ما نص الطلب والاستجابة؟
- هل توجد محاولة مكررة؟

لذلك يجب إنشاء جدول دائم، وليس سجلًا مؤقتًا داخل الذاكرة.

---

## 11. الحقول المقترحة للجدول

الاسم المقترح:

```text
EpttsOperation
```

أو أي اسم يتوافق مع معايير ERP.

الحقول المقترحة:

```text
Id
OperationType
EnvironmentName
PharmacyId
PharmacyGln
ErpDocumentType
ErpDocumentId
ErpDocumentNumber
Sgtin
Sscc
Quantity
UnitOfMeasure
SourceGln
SourceSgln
DestinationGln
DestinationSgln
ReturnRequestNumber
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
LocalStatus
FinalMessageStatus
HttpStatusCode
ErrorCode
ErrorMessage
FinalErrorMessage
RequestJson
ResponseJson
StatusRequestJson
StatusResponseJson
CreatedAt
CreatedBy
SubmittedAt
LastStatusCheckAt
CompletedAt
RetryCount
RowVersion
```

ليس مطلوبًا استخدام كل حقل لكل عملية.

احتفظ بالقيم غير المستخدمة كـ `NULL` حسب تصميم قاعدة البيانات.

---

## 12. نموذج Class لسجل العملية

```csharp
public sealed class EpttsOperationRecord
{
    public long Id { get; set; }

    public string OperationType { get; set; } =
        string.Empty;

    public string EnvironmentName { get; set; } =
        string.Empty;

    public int PharmacyId { get; set; }

    public string PharmacyGln { get; set; } =
        string.Empty;

    public string? ErpDocumentType { get; set; }

    public long? ErpDocumentId { get; set; }

    public string? ErpDocumentNumber { get; set; }

    public string? Sgtin { get; set; }

    public string? Sscc { get; set; }

    public decimal? Quantity { get; set; }

    public string? SourceGln { get; set; }

    public string? DestinationGln { get; set; }

    public string? ReturnRequestNumber { get; set; }

    public string? RequestInstanceIdentifier { get; set; }

    public string? MessageId { get; set; }

    public string? StatusQueryIdentifier { get; set; }

    public string LocalStatus { get; set; } =
        EpttsLocalStatus.Draft;

    public string? FinalMessageStatus { get; set; }

    public int? HttpStatusCode { get; set; }

    public string? ErrorCode { get; set; }

    public string? ErrorMessage { get; set; }

    public string? FinalErrorMessage { get; set; }

    public string? RequestJson { get; set; }

    public string? ResponseJson { get; set; }

    public DateTimeOffset CreatedAt { get; set; }

    public DateTimeOffset? LastStatusCheckAt { get; set; }

    public DateTimeOffset? CompletedAt { get; set; }

    public int RetryCount { get; set; }
}
```

هذه Class مقترحة داخل ERP وليست جزءًا من المكتبة.

---

## 13. الحالات المحلية المقترحة

```csharp
public static class EpttsLocalStatus
{
    public const string Draft =
        "Draft";

    public const string Submitting =
        "Submitting";

    public const string Pending =
        "Pending";

    public const string Successful =
        "Successful";

    public const string Failed =
        "Failed";

    public const string NeedsReview =
        "NeedsReview";
}
```

### `Draft`

السجل محلي ولم يبدأ إرساله بعد.

### `Submitting`

بدأت محاولة الإرسال ولم تُحسم بعد.

هذه الحالة مفيدة لمنع الإرسال المتزامن من أكثر من شاشة أو Process.

### `Pending`

قُبلت الرسالة ويوجد `StatusQueryIdentifier` صالح.

### `Successful`

وصلت الرسالة إلى نتيجة نهائية ناجحة.

### `Failed`

وصلت الرسالة إلى `Application Error` نهائي أو رفض نهائي واضح.

### `NeedsReview`

النتيجة غير مؤكدة وتحتاج تدخلًا أو متابعة خاصة.

---

## 14. دورة حياة السجل

```text
Draft
→ Submitting
→ Pending
→ Successful
```

أو:

```text
Draft
→ Submitting
→ Pending
→ Failed
```

أو عند نتيجة غير مؤكدة:

```text
Draft
→ Submitting
→ NeedsReview
```

وقد يعود `NeedsReview` إلى `Pending` إذا تم العثور على معرف متابعة صحيح.

---

## 15. أنواع العمليات المحلية

```csharp
public static class EpttsOperationType
{
    public const string Receiving =
        "Receiving";

    public const string FullDispensing =
        "FullDispensing";

    public const string PartialDispensing =
        "PartialDispensing";

    public const string DispenseCancel =
        "DispenseCancel";

    public const string ReturnToBranch =
        "ReturnToBranch";

    public const string ReturnCancel =
        "ReturnCancel";
}
```

استخدم قيمًا ثابتة بدل نصوص مختلفة في كل شاشة.

---

# القسم الثاني: Repository وحفظ العمليات

## 16. واجهة Repository مقترحة

```csharp
public interface IEpttsOperationRepository
{
    long InsertDraft(
        EpttsOperationRecord operation);

    bool TryMarkSubmitting(
        long operationId);

    void MarkPending(
        long operationId,
        string? requestInstanceIdentifier,
        string? messageId,
        string statusQueryIdentifier,
        int? httpStatusCode,
        string? requestJson,
        string? responseJson);

    void MarkSuccessful(
        long operationId,
        string? finalMessageStatus,
        string? statusResponseJson);

    void MarkFailed(
        long operationId,
        string? finalMessageStatus,
        string? finalErrorMessage,
        string? statusResponseJson);

    void MarkNeedsReview(
        long operationId,
        string? errorCode,
        string? errorMessage,
        string? requestJson,
        string? responseJson);

    bool HasConflictingPendingOperation(
        int pharmacyId,
        string operationType,
        string? sgtin,
        string? sscc);

    IReadOnlyCollection<EpttsOperationRecord>
        GetPendingOperations(
            int batchSize);
}
```

هذه الواجهة مثال تصميمي ويجب تنفيذها بتقنية التخزين المستخدمة داخل ERP.

---

## 17. أهمية `TryMarkSubmitting`

تعطيل زر الإرسال في UI لا يمنع:

- فتح شاشتين.
- ضغط مستخدمين مختلفين.
- إرسال Process آخر.
- إعادة تشغيل التطبيق.

لذلك يجب حجز العملية في قاعدة البيانات قبل الاتصال بالخدمة.

مثال منطقي:

```csharp
bool acquired =
    repository.TryMarkSubmitting(
        operationId);

if (!acquired)
{
    throw new InvalidOperationException(
        "The operation is already being submitted.");
}
```

يجب تنفيذ هذا التغيير بصورة ذرية داخل قاعدة البيانات.

---

## 18. المعاملة المحلية وحدودها

لا توجد Transaction واحدة تشمل قاعدة بيانات ERP وخدمة EPTTS معًا.

لذلك لا تنفذ هذا الافتراض:

```text
Begin SQL Transaction
→ Send EPTTS HTTP Request
→ Commit both systems together
```

الاتصال الخارجي لا يشارك تلقائيًا في SQL Transaction.

الأسلوب العملي:

```text
1. أنشئ سجلًا محليًا Draft
2. غيّره ذريًا إلى Submitting
3. أرسل الطلب
4. احفظ Pending أو NeedsReview
5. تابع النتيجة لاحقًا
```

هذا نمط تكامل غير متزامن يعتمد على المزامنة النهائية، وليس Transaction موزعة.

---

## 19. الحفظ فورًا بعد Submission

بمجرد استلام `StatusQueryIdentifier`:

```csharp
repository.MarkPending(
    operationId:
        operationId,

    requestInstanceIdentifier:
        submission.Data.RequestInstanceIdentifier,

    messageId:
        submission.Data.MessageId,

    statusQueryIdentifier:
        submission.Data.StatusQueryIdentifier,

    httpStatusCode:
        ConvertHttpStatusCode(
            submission.HttpStatusCode),

    requestJson:
        submission.RawRequest,

    responseJson:
        submission.RawResponse);
```

الدالة `ConvertHttpStatusCode` اختيارية وتعتمد على نوع الخاصية الفعلي داخل المكتبة.

لا تؤجل الحفظ حتى إغلاق الشاشة.

---

# القسم الثالث: منع التكرار وIdempotency المحلية

## 20. لماذا يحدث التكرار؟

قد يحدث التكرار بسبب:

- ضغط زر الإرسال مرتين.
- بطء الشبكة.
- `Timeout` بعد وصول الرسالة.
- إعادة المحاولة التلقائية غير المنضبطة.
- تشغيل أكثر من Job.
- فتح العملية من جهازين.
- عدم حفظ حالة `Submitting` أو `Pending`.

---

## 21. قاعدة منع التكرار الأساسية

قبل عملية تغير حالة عبوة، امنع وجود سجل:

```text
نفس PharmacyId
ونفس SGTIN أو SSCC
وعملية متعارضة
وحالته Submitting أو Pending أو NeedsReview
```

لا تقصر الفحص على نفس `OperationType` فقط.

مثال:

```text
FullDispensing Pending
```

يجب أن يمنع غالبًا:

- FullDispensing جديدة.
- PartialDispensing.
- ReturnToBranch.

حتى تُحسم العملية السابقة.

---

## 22. مصفوفة تعارض مبسطة

يمكن لفريق ERP تعريف قواعد مثل:

```text
Receiving Pending على SSCC
→ يمنع Receiving أخرى على SSCC نفسه

FullDispensing Pending على SGTIN
→ يمنع الصرف والمرتجع والإلغاء الجديد

PartialDispensing Pending على SGTIN
→ يمنع PartialDispensing أخرى حتى ظهور النتيجة

ReturnToBranch Pending على SGTIN
→ يمنع البيع أو مرتجع جديد

ReturnCancel Pending
→ يمنع إلغاء آخر أو عملية جديدة على العبوة
```

يجب اعتماد المصفوفة النهائية مع فريق الأعمال وEPTTS.

---

## 23. Unique Index مقترح

يمكن إنشاء مفتاح فريد مشروط حسب إمكانات قاعدة البيانات.

الفكرة:

```text
EnvironmentName
PharmacyId
OperationKey
ActiveIntegrationStatus
```

حيث `OperationKey` قد يكون:

```text
SGTIN
أو SSCC
أو ReturnRequestNumber
```

لا تعتمد على الفحص البرمجي وحده إذا كان أكثر من Process قادرًا على الإرسال.

---

## 24. Retry لعمليات القراءة مقابل عمليات التغيير

### عمليات القراءة

يمكن إعادة محاولة:

```text
VerifyProduct
GetMessageStatus
```

عند أخطاء اتصال مؤقتة وفق سياسة محدودة.

### عمليات تغيير الحالة

لا تعِد إرسال:

```text
Receiving
Dispensing
Dispense Cancel
Return
Return Cancel
```

عند نتيجة غير مؤكدة قبل مراجعة الرسالة السابقة.

---

# القسم الرابع: خدمة إرسال العمليات

## 25. واجهة خدمة مقترحة

```csharp
public interface IEpttsOperationService
{
    Task<SubmissionResult> SubmitFullDispensingAsync(
        FullDispensingCommand command,
        CancellationToken cancellationToken);

    Task<SubmissionResult> SubmitReturnAsync(
        ReturnCommand command,
        CancellationToken cancellationToken);
}
```

`FullDispensingCommand` و`ReturnCommand` و`SubmissionResult` Classes محلية ينشئها ERP.

---

## 26. نتيجة تطبيقية موحدة

```csharp
public sealed class SubmissionResult
{
    public bool Accepted { get; init; }

    public bool NeedsReview { get; init; }

    public long OperationId { get; init; }

    public string? StatusQueryIdentifier { get; init; }

    public string? ErrorCode { get; init; }

    public string? ErrorMessage { get; init; }
}
```

هذه النتيجة تمنع UI من الاعتماد مباشرة على جميع تفاصيل المكتبة.

---

## 27. مثال خدمة صرف كامل

```csharp
public async Task<SubmissionResult>
    SubmitFullDispensingAsync(
        FullDispensingCommand command,
        CancellationToken cancellationToken)
{
    if (command == null)
    {
        throw new ArgumentNullException(
            nameof(command));
    }

    if (repository.HasConflictingPendingOperation(
            command.PharmacyId,
            EpttsOperationType.FullDispensing,
            command.Sgtin,
            null))
    {
        return new SubmissionResult
        {
            ErrorCode =
                "LOCAL_PENDING_OPERATION",

            ErrorMessage =
                "A pending EPTTS operation already exists " +
                "for this pack."
        };
    }

    EpttsOperationRecord operation =
        new EpttsOperationRecord
        {
            OperationType =
                EpttsOperationType.FullDispensing,

            EnvironmentName =
                command.EnvironmentName,

            PharmacyId =
                command.PharmacyId,

            PharmacyGln =
                command.PharmacyGln,

            ErpDocumentId =
                command.ErpDocumentId,

            Sgtin =
                command.Sgtin,

            LocalStatus =
                EpttsLocalStatus.Draft,

            CreatedAt =
                DateTimeOffset.Now
        };

    long operationId =
        repository.InsertDraft(
            operation);

    if (!repository.TryMarkSubmitting(operationId))
    {
        return new SubmissionResult
        {
            OperationId =
                operationId,

            ErrorCode =
                "LOCAL_SUBMISSION_LOCK_FAILED",

            ErrorMessage =
                "The operation could not be reserved for submission."
        };
    }

    ErpEpttsSettings settings =
        settingsProvider.GetForPharmacy(
            command.PharmacyId);

    EpttsOptions options =
        CreateOptions(settings);

    FullDispensingRequest request =
        new FullDispensingRequest
        {
            Sgtin =
                command.Sgtin,

            PatientReference =
                command.PatientReference,

            PrescriptionReference =
                command.PrescriptionReference,

            EventDateTime =
                command.EventDateTime
        };

    try
    {
        EpttsResult<SubmissionResponse> submission;

        using (EpttsClient client =
            new EpttsClient(options))
        {
            submission =
                await client.DispenseFullPackAsync(
                    request,
                    cancellationToken);
        }

        if (!submission.IsSuccess ||
            submission.Data == null)
        {
            repository.MarkNeedsReview(
                operationId,
                submission.ErrorCode,
                submission.ErrorMessage,
                submission.RawRequest,
                submission.RawResponse);

            return new SubmissionResult
            {
                OperationId =
                    operationId,

                NeedsReview =
                    true,

                ErrorCode =
                    submission.ErrorCode,

                ErrorMessage =
                    submission.ErrorMessage
            };
        }

        if (!submission.Data.CanQueryFinalStatus ||
            string.IsNullOrWhiteSpace(
                submission.Data.StatusQueryIdentifier))
        {
            repository.MarkNeedsReview(
                operationId,
                "MISSING_STATUS_IDENTIFIER",
                "EPTTS did not return a status identifier.",
                submission.RawRequest,
                submission.RawResponse);

            return new SubmissionResult
            {
                OperationId =
                    operationId,

                NeedsReview =
                    true,

                ErrorCode =
                    "MISSING_STATUS_IDENTIFIER",

                ErrorMessage =
                    "EPTTS did not return a status identifier."
            };
        }

        repository.MarkPending(
            operationId,
            submission.Data.RequestInstanceIdentifier,
            submission.Data.MessageId,
            submission.Data.StatusQueryIdentifier,
            ConvertHttpStatusCode(
                submission.HttpStatusCode),
            submission.RawRequest,
            submission.RawResponse);

        return new SubmissionResult
        {
            Accepted =
                true,

            OperationId =
                operationId,

            StatusQueryIdentifier =
                submission.Data.StatusQueryIdentifier
        };
    }
    catch (EpttsConfigurationException exception)
    {
        repository.MarkNeedsReview(
            operationId,
            "CONFIGURATION_ERROR",
            exception.Message,
            null,
            null);

        throw;
    }
    catch (EpttsValidationException exception)
    {
        repository.MarkNeedsReview(
            operationId,
            "VALIDATION_ERROR",
            exception.Message,
            null,
            null);

        throw;
    }
    catch (Exception exception)
    {
        repository.MarkNeedsReview(
            operationId,
            "UNEXPECTED_SUBMISSION_ERROR",
            exception.Message,
            null,
            null);

        throw;
    }
}
```

المثال يوضح النمط، ويجب مواءمته مع أنواع الخصائص الفعلية وتقنية التخزين داخل ERP.

---

## 28. لماذا نستخدم NeedsReview عند فشل الإرسال غير المؤكد؟

إذا أعادت العملية:

```text
Timeout
Cancellation
Connection reset
```

فلا توجد ضمانة أن EPTTS لم تستلم الرسالة.

لذلك لا تستخدم `Failed` مباشرة.

استخدم:

```text
NeedsReview
```

ثم:

- راجع `RawRequest`.
- استخرج المعرف إن توفر.
- استعلم عن الرسالة.
- امنع عملية جديدة متعارضة.

---

# القسم الخامس: Background Job

## 29. لماذا نحتاج Job؟

لا يجب أن يعتمد نجاح العملية على بقاء شاشة المستخدم مفتوحة.

قد يحدث:

- إغلاق التطبيق.
- إعادة تشغيل الجهاز.
- انتهاء Session.
- تعطل الشبكة.
- استغراق المعالجة مدة أطول من وقت الشاشة.

الـ Job يضمن استمرار المتابعة من قاعدة البيانات.

---

## 30. مهام Job

```text
1. قراءة دفعة من العمليات Pending
2. قفل السجلات لمنع Job آخر من معالجتها
3. تحميل إعدادات الصيدلية والبيئة
4. تنفيذ GetMessageStatusAsync
5. تحديث Successful أو Failed أو إبقاء Pending
6. حفظ LastStatusCheckAt
7. زيادة RetryCount للاستعلام
8. تحرير القفل
```

---

## 31. مثال Job مبسط

```csharp
public sealed class EpttsMessageStatusJob
{
    private readonly IEpttsOperationRepository _repository;
    private readonly IEpttsSettingsProvider _settingsProvider;

    public EpttsMessageStatusJob(
        IEpttsOperationRepository repository,
        IEpttsSettingsProvider settingsProvider)
    {
        _repository = repository;
        _settingsProvider = settingsProvider;
    }

    public async Task RunAsync(
        CancellationToken cancellationToken)
    {
        IReadOnlyCollection<EpttsOperationRecord> operations =
            _repository.GetPendingOperations(
                batchSize: 100);

        foreach (EpttsOperationRecord operation in operations)
        {
            cancellationToken.ThrowIfCancellationRequested();

            if (string.IsNullOrWhiteSpace(
                    operation.StatusQueryIdentifier))
            {
                _repository.MarkNeedsReview(
                    operation.Id,
                    "MISSING_STATUS_IDENTIFIER",
                    "The pending operation has no status identifier.",
                    operation.RequestJson,
                    operation.ResponseJson);

                continue;
            }

            ErpEpttsSettings settings =
                _settingsProvider.GetForPharmacy(
                    operation.PharmacyId);

            EpttsOptions options =
                CreateOptions(settings);

            try
            {
                EpttsResult<MessageStatusResponse> result;

                using (EpttsClient client =
                    new EpttsClient(options))
                {
                    result =
                        await client.GetMessageStatusAsync(
                            operation.StatusQueryIdentifier,
                            cancellationToken);
                }

                if (!result.IsSuccess ||
                    result.Data == null)
                {
                    /*
                     * احفظ فشل الاستعلام، لكن لا تغير
                     * العملية الأصلية إلى Failed نهائيًا.
                     */

                    continue;
                }

                if (result.Data.IsSuccessful)
                {
                    _repository.MarkSuccessful(
                        operation.Id,
                        result.Data.MessageStatus,
                        result.RawResponse);

                    continue;
                }

                if (result.Data.IsApplicationError)
                {
                    _repository.MarkFailed(
                        operation.Id,
                        result.Data.MessageStatus,
                        result.Data.FirstErrorMessage,
                        result.RawResponse);

                    continue;
                }

                /*
                 * ما زالت Pending.
                 */
            }
            catch (OperationCanceledException)
                when (cancellationToken.IsCancellationRequested)
            {
                throw;
            }
            catch (Exception exception)
            {
                /*
                 * سجل فشل محاولة الاستعلام.
                 * لا تعتبر العملية الأصلية Failed.
                 */
            }
        }
    }
}
```

---

## 32. عدم تشغيل Jobs متداخلة

إذا كان ERP يعمل على أكثر من Server أو Process، يجب منع معالجة السجل نفسه في الوقت نفسه.

استخدم واحدة من الطرق المناسبة للنظام:

- قفل سجل في قاعدة البيانات.
- `RowVersion` أو Optimistic Concurrency.
- عمود `LockedAt` و`LockedBy`.
- Queue موثوقة.
- Distributed Lock.

لا تعتمد على متغير `static` داخل Process واحد.

---

## 33. سياسة Polling

لا تستعلم عن كل عملية كل ثانية بلا حاجة.

مثال سياسة:

```text
أول دقيقة
→ كل 5 ثوانٍ

بعد ذلك حتى 10 دقائق
→ كل 30 ثانية

بعد ذلك
→ كل عدة دقائق أو NeedsReview حسب الاتفاق
```

الأرقام النهائية يجب أن تتوافق مع متطلبات EPTTS وحدود الخدمة.

احفظ:

```text
LastStatusCheckAt
NextStatusCheckAt
RetryCount
```

حتى يستطيع Job اختيار السجلات المستحقة فقط.

---

## 34. العمليات القديمة المعلقة

حدد سياسة للعمليات التي تبقى `Pending` مدة غير معتادة.

مثال:

```text
Pending أقل من الحد
→ استمر في الاستعلام

Pending تجاوز الحد
→ NeedsReview
→ تنبيه فريق الدعم
```

لا تحولها تلقائيًا إلى `Failed` دون نتيجة نهائية من EPTTS.

---

# القسم السادس: ربط النتيجة بمستندات ERP

## 35. متى يتم تحديث مستند ERP؟

يعتمد ذلك على قواعد النظام، لكن يجب التمييز بين:

```text
إنشاء مستند محلي
```

و:

```text
اعتماد نجاح EPTTS
```

يمكن أن يمر المستند بحالة مثل:

```text
Draft
Awaiting EPTTS
Completed
EPTTS Failed
Needs Review
```

---

## 36. التطبيق المقترح عند Submission

بعد قبول الرسالة:

```text
EPTTS Operation = Pending
ERP Document = Awaiting EPTTS
```

لا تعرض للمستخدم أن العملية اكتملت نهائيًا.

رسالة مناسبة:

```text
تم إرسال العملية إلى EPTTS وهي قيد المعالجة.
```

---

## 37. التطبيق عند Successful

عند:

```csharp
result.Data.IsSuccessful == true
```

يمكن تنفيذ:

```text
EPTTS Operation = Successful
ERP Document = Completed
تطبيق أثر المخزون النهائي
تسجيل CompletedAt
```

يجب أن يكون تحديث سجل التكامل ومستند ERP متسقًا داخل معاملة قاعدة بيانات محلية واحدة متى أمكن.

---

## 38. التطبيق عند Application Error

عند:

```csharp
result.Data.IsApplicationError == true
```

نفذ:

```text
EPTTS Operation = Failed
ERP Document = EPTTS Failed
عدم تطبيق أثر النجاح
حفظ FirstErrorMessage
```

إذا سبق ERP أن طبق أثرًا مؤقتًا، يجب أن تكون هناك سياسة Rollback أو Reversal داخلية واضحة.

---

## 39. تحقق ما بعد النجاح

بعد نجاح عمليات معينة، نفذ `VerifyProduct` لتأكيد الحالة المتوقعة.

أمثلة:

```text
Receiving successful
→ active لدى PharmacyGln

Full Dispensing successful
→ dispensed

Dispense Cancel successful
→ active

Partial Dispensing successful
→ partially_dispensed غالبًا

Return Cancel successful
→ active لدى PharmacyGln
```

إذا نجحت الرسالة نهائيًا لكن حالة العبوة لم تطابق التوقع:

```text
لا تعِد إرسال العملية
→ سجل NeedsReview
→ احتفظ بالأدلة
→ راجع الدعم
```

---

# القسم السابع: الأمان وحماية البيانات

## 40. حماية IntegratorKey

`IntegratorKey` سر يجب حمايته.

لا تحفظه في:

- كود المصدر.
- GitHub.
- ملف Markdown.
- Screenshot.
- Logs.
- Error Message.
- `RawRequest`.
- `RawResponse`.

استخدم مصدرًا آمنًا يناسب بنية ERP، مثل:

- Secret Store.
- متغير بيئة محمي.
- إعداد مشفر في قاعدة البيانات.
- خدمة مركزية للأسرار.

---

## 41. عدم كشف المفتاح في UI

استخدم حقل Password أو إخفاء مناسب إذا كانت هناك شاشة إعدادات.

لا تضف زرًا يعرض المفتاح نصيًا دون ضوابط أمنية.

لا تعرض القيمة داخل شاشة التشخيص.

يمكن عرض:

```text
Integrator Key: Configured
```

بدل القيمة نفسها.

---

## 42. تشفير المفتاح في التخزين

إذا اضطر ERP إلى تخزين المفتاح محليًا:

- لا تخزنه كنص واضح.
- استخدم آلية تشفير معتمدة داخل المؤسسة.
- افصل مفتاح التشفير عن البيانات المشفرة.
- قيد صلاحية القراءة لخدمة التكامل فقط.
- سجل عمليات تغيير المفتاح دون تسجيل قيمته.

---

## 43. تدوير IntegratorKey

يجب دعم تغيير المفتاح دون إعادة Build.

التسلسل:

```text
إضافة المفتاح الجديد إلى مصدر الأسرار
→ اختبار الاتصال
→ تفعيل المفتاح الجديد
→ إبطال القديم
→ تسجيل تاريخ التغيير
```

لا تضع المفتاح في Constants داخل DLL أو ERP.

---

## 44. بيانات المريض

لا ترسل بيانات شخصية مباشرة داخل:

```text
PatientReference
PrescriptionReference
```

استخدم معرفات داخلية لا تكشف بيانات المريض.

لا تسجل معلومات طبية أو شخصية غير لازمة في سجل التكامل.

طبق سياسات حماية البيانات الخاصة بالمؤسسة.

---

## 45. حماية RawRequest وRawResponse

قد تحتوي البيانات الخام على معرفات عبوات ومستندات ومراجع داخلية.

لذلك:

- قيد الوصول إليها.
- لا تعرضها للمستخدم العادي.
- طبق سياسة احتفاظ وحذف.
- لا ترسلها عبر قنوات غير آمنة.
- راجعها قبل مشاركتها مع الدعم الخارجي.

---

# القسم الثامن: Logging وMonitoring

## 46. ما الذي يجب تسجيله؟

سجل على الأقل:

```text
Timestamp
EnvironmentName
OperationType
OperationId
PharmacyId
PharmacyGln
ErpDocumentId
SGTIN أو SSCC
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
LocalStatus
HttpStatusCode
ErrorCode
ErrorMessage
ElapsedMilliseconds
```

يمكن حفظ النصوص الخام في جدول أو مخزن مخصص حسب سياسة النظام.

---

## 47. ما الذي لا يجب تسجيله؟

```text
IntegratorKey
Authorization Headers
Password
Connection String كاملة تحتوي على أسرار
بيانات مريض شخصية مباشرة
```

---

## 48. Correlation داخل Logs

استخدم معرفًا محليًا ثابتًا مثل:

```text
OperationId
```

مع المعرفات الخارجية:

```text
RequestInstanceIdentifier
MessageId
StatusQueryIdentifier
```

بهذا يستطيع فريق الدعم تتبع العملية من شاشة ERP إلى Logs ثم إلى EPTTS.

---

## 49. مؤشرات المراقبة المقترحة

راقب:

```text
عدد العمليات Pending
عمر أقدم عملية Pending
عدد Failed
عدد NeedsReview
نسبة HTTP 401 و403
عدد Timeouts
متوسط زمن الوصول للحالة النهائية
عدد العمليات المكررة الممنوعة
عدد Jobs الفاشلة
```

أنشئ تنبيهًا عندما:

- يزداد عدد `Pending` بصورة غير معتادة.
- يظهر عدد كبير من `401` أو `403`.
- يفشل Job عدة مرات متتالية.
- تبقى عملية معلقة أكثر من الحد المعتمد.

---

# القسم التاسع: التعامل مع التعافي والأعطال

## 50. إغلاق التطبيق بعد Submission

إذا تم حفظ العملية `Pending` قبل الإغلاق، يمكن للـ Job استكمال المتابعة بعد إعادة التشغيل.

إذا لم تحفظ `StatusQueryIdentifier`، قد تفقد القدرة السهلة على متابعة الرسالة.

لذلك يجب أن يحدث الحفظ فورًا بعد الاستجابة.

---

## 51. تعطل قاعدة البيانات بعد قبول EPTTS

هذه حالة حساسة:

```text
EPTTS قبلت الرسالة
لكن ERP فشل في حفظ StatusQueryIdentifier
```

لتقليل المخاطر:

- أنشئ سجل `Submitting` قبل الإرسال.
- احفظ `RawRequest` أو `RequestInstanceIdentifier` مبكرًا إذا تسمح البنية.
- سجل التفاصيل في Logging موثوق.
- وفر شاشة أو أداة لاستعادة الرسائل غير المحسومة.
- لا تسمح بإعادة الإرسال المباشر.

---

## 52. تعطل Job

لا تغير العمليات إلى `Failed` بسبب توقف Job.

عند عودة Job:

```text
اقرأ العمليات Pending
→ استأنف GetMessageStatusAsync
```

ينبغي أن يكون Job قابلًا لإعادة التشغيل دون آثار جانبية.

---

## 53. إعادة معالجة Failed مقابل Pending

### Failed نهائيًا

لا تعِد نفس الرسالة تلقائيًا.

يجب تصحيح سبب الخطأ وإنشاء قرار أعمال واضح بشأن إعادة العملية.

### Pending

استمر في الاستعلام عن الرسالة نفسها.

### NeedsReview

لا ترسل عملية بديلة قبل مراجعة الرسالة والمعرفات.

---

# القسم العاشر: Testing

## 54. Unit Tests

اختبر منطق ERP دون استدعاء الخدمة الحقيقية:

- منع التكرار.
- تحويل `SubmissionResponse` إلى `Pending`.
- تحويل `MessageStatusResponse` إلى `Successful` أو `Failed`.
- الحفاظ على `Pending` عند حالة غير نهائية.
- استخدام `NeedsReview` عند نتيجة غير مؤكدة.
- عدم كشف المفتاح في Logs.
- استخدام نفس `ReturnRequestNumber` في الإلغاء.

يفضل وضع واجهة حول استخدام المكتبة لتسهيل Mocking.

---

## 55. Integration Tests

نفذ في Staging:

- `VerifyProduct` لعبوات بحالات مختلفة.
- كل عملية تغيير حالة.
- `Message Status` حتى النتيجة النهائية.
- `Timeout` محاكى.
- `Cancellation`.
- HTTP 401 و403 إن أمكن بصورة آمنة.
- إعادة تشغيل التطبيق مع عمليات `Pending`.
- تشغيل Job أكثر من مرة.
- محاولة إرسال عملية مكررة.

---

## 56. UAT

اختبر السيناريوهات الكاملة:

```text
Receiving
→ Pending
→ Successful
→ VerifyProduct

Full Dispensing
→ Pending
→ Successful
→ VerifyProduct = dispensed

Dispense Cancel
→ Pending
→ Successful
→ VerifyProduct = active

Partial Dispensing
→ Pending
→ Successful
→ VerifyProduct

Return
→ Pending
→ Successful

Return Cancel
→ نفس ReturnRequestNumber
→ Pending
→ Successful
→ VerifyProduct = active
```

---

## 57. اختبار تعدد المستخدمين

اختبر:

```text
مستخدمان يرسلان العملية نفسها
شاشتان مفتوحتان لنفس العبوة
Job يعمل أثناء محاولة إرسال جديدة
إعادة فتح المستند أثناء Pending
```

يجب أن تمنع قاعدة البيانات التكرار، لا UI فقط.

---

# القسم الحادي عشر: النشر والإصدارات

## 58. تثبيت إصدار المكتبة

سجل داخل ERP أو ملف النشر:

```text
Eptts.Client.dll version
Newtonsoft.Json version
Deployment date
Environment
Release notes
```

لا تستبدل DLL في Production دون اختبار Regression.

---

## 59. تحديث DLL

التسلسل المقترح:

```text
1. استلام DLL وXML الجديدين
2. حفظ نسخة الإصدار السابق
3. مراجعة Release Notes
4. استبدال الملفات في فرع تطوير
5. Rebuild
6. تشغيل Unit Tests
7. تشغيل Integration Tests في Staging
8. تنفيذ UAT مختصر
9. نشر Production
10. مراقبة Logs وPending Operations
```

يجب أن يأتي `Eptts.Client.xml` من Build نفسه الخاص بـ DLL.

---

## 60. التوافق الخلفي

قبل التحديث، راجع:

- Namespaces.
- توقيعات الدوال.
- Request properties.
- Response properties.
- أنواع `HttpStatusCode`.
- أكواد الأخطاء.
- Dependencies.
- Target Framework.

لا تعتمد على Build ناجح فقط. اختبر السيناريوهات الفعلية.

---

# القسم الثاني عشر: قائمة تنفيذ عملية

## 61. قبل الإرسال

```text
[ ] تم تحديد الصيدلية الصحيحة
[ ] تم تحميل إعدادات البيئة الصحيحة
[ ] تم تنفيذ options.Validate()
[ ] تم التحقق من المستند المحلي
[ ] تم التحقق من SGTIN أو SSCC
[ ] تم تنفيذ VerifyProduct عند الحاجة
[ ] تم فحص Status وCurrentGln وIsRecalled وAlerts
[ ] تم منع عملية Pending متعارضة
[ ] تم إنشاء سجل Draft
[ ] تم حجز السجل كـ Submitting
```

---

## 62. بعد Submission

```text
[ ] تم فحص IsSuccess
[ ] تم فحص Data
[ ] تم حفظ RawRequest
[ ] تم حفظ RawResponse
[ ] تم حفظ RequestInstanceIdentifier
[ ] تم حفظ MessageId
[ ] تم حفظ StatusQueryIdentifier
[ ] تم تحويل العملية إلى Pending
[ ] لم يتم اعتماد النجاح النهائي
```

---

## 63. عند Message Status

```text
[ ] تم استخدام StatusQueryIdentifier الصحيح
[ ] تم فحص IsSuccess
[ ] تم فحص Data
[ ] تم فحص IsSuccessful
[ ] تم فحص IsApplicationError
[ ] تم حفظ FirstErrorMessage
[ ] تم حفظ StatusResponseJson
[ ] تم تحديث LastStatusCheckAt
[ ] بقيت الحالة غير النهائية Pending
```

---

## 64. عند النجاح النهائي

```text
[ ] تم تحويل التكامل إلى Successful
[ ] تم تحديث مستند ERP
[ ] تم تطبيق أثر المخزون الصحيح
[ ] تم تسجيل CompletedAt
[ ] تم تنفيذ VerifyProduct بعد العملية عند الحاجة
[ ] لم يتم إرسال العملية مرة أخرى
```

---

## 65. عند الفشل أو النتيجة غير المؤكدة

```text
[ ] تم التفريق بين Failed وNeedsReview
[ ] لم يتم اعتبار Timeout فشلًا وظيفيًا نهائيًا
[ ] لم يتم إعادة الإرسال تلقائيًا
[ ] تم الاحتفاظ بالمعرفات
[ ] تم حفظ التفاصيل الفنية دون IntegratorKey
[ ] تم منع العمليات المتعارضة حتى المراجعة
```

---

## 66. أخطاء تصميم يجب تجنبها

### وضع كل الكود داخل الشاشة

يؤدي إلى صعوبة الاختبار وإعادة الاستخدام.

### حفظ StatusQueryIdentifier في الذاكرة فقط

يؤدي إلى فقدان المتابعة عند إغلاق التطبيق.

### اعتبار HTTP 202 نجاحًا

يؤدي إلى تحديث ERP قبل النتيجة النهائية.

### إعادة الإرسال التلقائي لعمليات التغيير

قد يؤدي إلى عمليات مكررة.

### استخدام Job دون قفل

قد يؤدي إلى معالجة السجل نفسه أكثر من مرة.

### تخزين IntegratorKey كنص واضح

يمثل خطرًا أمنيًا.

### استخدام MessageId بدل ReturnRequestNumber

يؤدي إلى فشل `Return Cancel`.

### مشاركة إعدادات صيدلية واحدة بين جميع الفروع

قد يؤدي إلى إرسال العمليات باسم جهة خاطئة.

---

## 67. خلاصة الجزء السادس

التكامل الإنتاجي الصحيح داخل ERP يجب أن يعمل بهذا الشكل:

```text
مستند ERP
→ تحقق من البيانات والعبوة
→ سجل Draft
→ حجز Submitting
→ إرسال EPTTS مرة واحدة
→ حفظ Pending ومعرفات الرسالة
→ Background Job يتابع Message Status
→ Successful أو Failed
→ تحديث مستند ERP والمخزون
→ VerifyProduct بعد النجاح عند الحاجة
```

والقواعد الأهم:

```text
لا تعتمد على الشاشة لحفظ حالة العملية
لا تعتبر HTTP 202 نجاحًا نهائيًا
لا تعِد عمليات التغيير تلقائيًا
احفظ StatusQueryIdentifier في قاعدة البيانات
استخدم Job للمتابعة
امنع التكرار في قاعدة البيانات
احمِ IntegratorKey
افصل Staging عن Production
استخدم نفس ReturnRequestNumber الأصلي عند Return Cancel
```
