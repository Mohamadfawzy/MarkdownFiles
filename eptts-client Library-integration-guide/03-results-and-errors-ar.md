# الجزء الثالث: فهم النتائج والأخطاء

> يشرح هذا الجزء كيف يقرأ نظام ERP نتائج مكتبة `Eptts.Client`، وكيف يفرق بين نجاح طلب HTTP ونجاح العملية التجارية، وكيف يتعامل مع الاستثناءات والأخطاء و`Timeout` و`CancellationToken` دون تكرار العمليات أو اتخاذ قرار غير صحيح.

---

## 1. لماذا يجب فهم النتائج قبل استخدام Endpoints؟

جميع العمليات لا تعيد النوع نفسه من النجاح.

هناك فرق بين:

```text
نجاح الاتصال وطلب HTTP
```

و:

```text
نجاح التحقق من العبوة
```

و:

```text
قبول رسالة العملية للمعالجة
```

و:

```text
نجاح العملية التجارية نهائيًا
```

أخطر خطأ يمكن أن يقع فيه ERP هو اعتبار:

```text
HTTP 202
```

نجاحًا نهائيًا لعملية مثل الصرف أو الاستلام أو المرتجع.

التسلسل الصحيح للعمليات غير المتزامنة هو:

```text
إرسال العملية
→ قبول الرسالة للمعالجة
→ حفظ العملية Pending
→ الاستعلام عن حالة الرسالة
→ Successful أو Failed
```

---

## 2. النتيجة الموحدة `EpttsResult<T>`

تعيد دوال المكتبة نتيجة موحدة من النوع:

```csharp
EpttsResult<T>
```

يمثل `T` نوع البيانات الخاصة بالعملية.

أمثلة:

```csharp
EpttsResult<VerifyProductResponse>
```

لعملية `VerifyProduct`.

```csharp
EpttsResult<SubmissionResponse>
```

للعمليات التي ترسل رسالة إلى EPTTS.

```csharp
EpttsResult<MessageStatusResponse>
```

للاستعلام عن حالة رسالة.

### Namespace المطلوب

```csharp
using Eptts.Client.Responses;
```

وقد تحتاج أيضًا إلى:

```csharp
using Eptts.Client.Clients;
using Eptts.Client.Configuration;
using Eptts.Client.Exceptions;
```

---

## 3. الخصائص الأساسية في `EpttsResult<T>`

تحتوي النتيجة عادة على الخصائص التالية:

### `IsSuccess`

توضح هل نجح تنفيذ الطلب تقنيًا واستطاعت المكتبة معالجة الاستجابة.

مثال:

```csharp
if (!result.IsSuccess)
{
    // فشل تقني أو HTTP أو Timeout أو Cancellation.
}
```

لا تعني `IsSuccess` وحدها أن العملية التجارية نجحت نهائيًا.

### `HttpStatusCode`

تحمل HTTP Status Code عند توفر استجابة من الخادم.

مثال:

```csharp
var httpStatusCode =
    result.HttpStatusCode;
```

قد تكون فارغة إذا لم تصل استجابة HTTP، مثل بعض حالات:

- انقطاع الاتصال.
- `Timeout`.
- إلغاء الطلب محليًا.
- فشل قبل وصول الطلب إلى استجابة قابلة للقراءة.

### `ErrorCode`

كود مختصر يوضح نوع الفشل.

مثال:

```csharp
string? errorCode =
    result.ErrorCode;
```

قد يحتوي على قيم مثل:

```text
REQUEST_TIMEOUT
REQUEST_CANCELLED
```

أو كود آخر تعيده المكتبة أو EPTTS.

### `ErrorMessage`

رسالة مقروءة توضح الفشل:

```csharp
string? errorMessage =
    result.ErrorMessage;
```

يجب عرض رسالة مناسبة للمستخدم، مع حفظ التفاصيل الفنية اللازمة للدعم.

### `Data`

الاستجابة المنظمة الخاصة بالعملية:

```csharp
var response =
    result.Data;
```

نوع `Data` يعتمد على `T`.

### `RawRequest`

النص الأصلي المرسل في Body الطلب:

```csharp
string? rawRequest =
    result.RawRequest;
```

يُستخدم للتشخيص والتدقيق، وليس بديلًا عن Request Class.

### `RawResponse`

النص الأصلي الذي أعاده الخادم:

```csharp
string? rawResponse =
    result.RawResponse;
```

يُستخدم للتشخيص والتدقيق، بينما يستخدم منطق ERP العادي `Data`.

---

## 4. القالب الأساسي لمعالجة أي نتيجة

استخدم هذا الترتيب عند قراءة أي `EpttsResult<T>`:

```csharp
if (!result.IsSuccess)
{
    string? errorCode =
        result.ErrorCode;

    string? errorMessage =
        result.ErrorMessage;

    var httpStatusCode =
        result.HttpStatusCode;

    string? rawRequest =
        result.RawRequest;

    string? rawResponse =
        result.RawResponse;

    /*
     * سجل الخطأ أو اعرض رسالة مناسبة.
     * لا تكمل مسار النجاح.
     */

    return;
}

if (result.Data == null)
{
    /*
     * نجح الطلب تقنيًا، لكن لم تُعد بيانات منظمة.
     * تعامل مع الحالة باعتبارها غير متوقعة
     * وتحتاج مراجعة.
     */

    return;
}

/*
 * اقرأ result.Data حسب نوع العملية.
 */
```

لا تستخدم:

```csharp
result.Data.SomeProperty
```

قبل فحص:

```csharp
result.Data != null
```

---

## 5. الفرق بين `Data` و`RawResponse`

### `Data`

Class منظمة للاستخدام داخل منطق ERP.

مثال:

```csharp
string status =
    result.Data.Pack.Status;
```

### `RawResponse`

النص الأصلي الذي أعاده EPTTS.

مثال:

```csharp
string? rawResponse =
    result.RawResponse;
```

### الاستخدام الصحيح

```text
قرارات ERP اليومية
→ استخدم Data

التحقيق والدعم والتدقيق
→ استخدم RawRequest وRawResponse
```

لا تحلل JSON يدويًا إذا كانت المكتبة أعادت `Data` بصورة صحيحة.

---

## 6. نتيجة `VerifyProduct`

تعيد `VerifyProductAsync`:

```csharp
EpttsResult<VerifyProductResponse>
```

مثال:

```csharp
EpttsResult<VerifyProductResponse> result =
    await client.VerifyProductAsync(
        sgtin,
        cancellationToken);
```

يجب فحص مستويين مختلفين:

```csharp
result.IsSuccess
```

ثم:

```csharp
result.Data.Verified
```

### الحالة الأولى: فشل الطلب

```csharp
result.IsSuccess == false
```

يعني وجود فشل تقني أو HTTP أو فشل في معالجة الاستجابة.

### الحالة الثانية: نجح الطلب ولم تُتحقق العبوة

```csharp
result.IsSuccess == true
result.Data != null
result.Data.Verified == false
```

يعني أن طلب `VerifyProduct` نجح، لكن EPTTS لم يؤكد العبوة.

لا تكمل إلى عملية صرف أو مرتجع اعتمادًا على `IsSuccess` وحدها.

### الحالة الثالثة: تم التحقق من العبوة

```csharp
result.IsSuccess == true
result.Data != null
result.Data.Verified == true
result.Data.Pack != null
```

يمكن بعد ذلك قراءة:

```csharp
result.Data.Pack.Status
result.Data.Pack.CurrentGln
result.Data.Pack.IsRecalled
result.Data.Alerts
```

ثم اتخاذ قرار مبدئي بشأن العملية المطلوبة.

---

## 7. قالب آمن لمعالجة `VerifyProduct`

```csharp
EpttsResult<VerifyProductResponse> result =
    await client.VerifyProductAsync(
        sgtin,
        cancellationToken);

if (!result.IsSuccess)
{
    /*
     * سجل HttpStatusCode وErrorCode وErrorMessage.
     */

    return;
}

if (result.Data == null)
{
    /*
     * استجابة منظمة فارغة.
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

سيتم شرح `VerifyProduct` بالكامل في الجزء الرابع من الدليل.

---

## 8. نتيجة العمليات غير المتزامنة `SubmissionResponse`

تعيد العمليات التي تغير حالة العبوة أو ملكيتها نتيجة مثل:

```csharp
EpttsResult<SubmissionResponse>
```

ومن أمثلتها:

- `ReceiveFromBranchAsync`.
- `DispenseFullPackAsync`.
- `DispensePartialAsync`.
- `CancelDispensingAsync`.
- `ReturnToBranchAsync`.
- `CancelReturnAsync`.

مثال:

```csharp
EpttsResult<SubmissionResponse> submission =
    await client.DispenseFullPackAsync(
        request,
        cancellationToken);
```

---

## 9. الخصائص المهمة في `SubmissionResponse`

بعد التأكد من:

```csharp
submission.IsSuccess == true
```

و:

```csharp
submission.Data != null
```

يمكن قراءة الخصائص المهمة.

### `RequestInstanceIdentifier`

معرف الطلب المرتبط بالرسالة المرسلة:

```csharp
string? requestInstanceIdentifier =
    submission.Data.RequestInstanceIdentifier;
```

### `MessageId`

معرف أعادته EPTTS:

```csharp
string? messageId =
    submission.Data.MessageId;
```

### `StatusQueryIdentifier`

المعرف المستخدم لمتابعة النتيجة النهائية:

```csharp
string? statusQueryIdentifier =
    submission.Data.StatusQueryIdentifier;
```

### `IsAcceptedForProcessing`

توضح هل قُبلت الرسالة للمعالجة:

```csharp
bool accepted =
    submission.Data.IsAcceptedForProcessing;
```

### `CanQueryFinalStatus`

توضح هل توجد قيمة مناسبة للاستعلام عن النتيجة النهائية:

```csharp
bool canQuery =
    submission.Data.CanQueryFinalStatus;
```

قد توفر الاستجابة خصائص إضافية مثل:

```csharp
submission.Data.StatusType
submission.Data.Code
submission.Data.Status
```

استخدمها للتشخيص وقراءة تفاصيل قبول الرسالة، لكن لا تعتبرها بديلًا عن `MsgStatusQuery` عندما تكون العملية غير متزامنة.

---

## 10. معنى HTTP 202 وI001

إذا أعادت العملية:

```text
HTTP 202
```

ومؤشرًا مثل:

```text
I001
```

فالمعنى هو:

```text
تم قبول الرسالة للمعالجة
```

ولا يعني:

```text
Receiving نجح نهائيًا
Dispensing نجح نهائيًا
Return نجح نهائيًا
Return Cancel نجح نهائيًا
```

القرار الصحيح داخل ERP:

```text
احفظ العملية بحالة Pending
→ احفظ StatusQueryIdentifier
→ لا تطبق مسار النجاح النهائي بعد
→ استعلم عن حالة الرسالة
```

---

## 11. قالب معالجة `SubmissionResponse`

```csharp
if (!submission.IsSuccess)
{
    /*
     * فشل الإرسال تقنيًا أو عبر HTTP.
     * لا تعتبر العملية ناجحة.
     */

    return;
}

if (submission.Data == null)
{
    /*
     * لا توجد SubmissionResponse منظمة.
     * احفظ الحالة للمراجعة.
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
     * لا توجد طريقة واضحة لمتابعة الرسالة.
     * احفظ العملية NeedsReview مع RawRequest وRawResponse.
     */

    return;
}

string statusQueryIdentifier =
    submission.Data.StatusQueryIdentifier;

/*
 * احفظ العملية داخل ERP بحالة Pending.
 * لا تعتبرها Successful بعد.
 */
```

---

## 12. ما الذي يجب حفظه بعد قبول العملية؟

احفظ على الأقل:

```csharp
submission.Data.RequestInstanceIdentifier
submission.Data.MessageId
submission.Data.StatusQueryIdentifier
submission.RawRequest
submission.RawResponse
```

واحفظ بيانات العملية المحلية:

```text
OperationType
ErpDocumentId
PharmacyGln
SGTIN أو SSCC
Quantity عند وجودها
SourceGln عند Receiving
DestinationGln عند Return
ReturnRequestNumber عند Return
LocalStatus = Pending
CreatedAt
CreatedBy
```

لا تنتظر حتى الاستعلام النهائي لحفظ العملية. احفظها فور الحصول على معرف المتابعة.

---

## 13. نتيجة `MessageStatusResponse`

تعيد دوال متابعة الرسائل:

```csharp
EpttsResult<MessageStatusResponse>
```

مثال:

```csharp
EpttsResult<MessageStatusResponse> statusResult =
    await client.GetMessageStatusAsync(
        statusQueryIdentifier,
        cancellationToken);
```

أو:

```csharp
EpttsResult<MessageStatusResponse> finalResult =
    await client.WaitForFinalStatusAsync(
        statusQueryIdentifier,
        TimeSpan.FromSeconds(3),
        TimeSpan.FromSeconds(30),
        cancellationToken);
```

---

## 14. الخصائص المهمة في `MessageStatusResponse`

### `MessageStatus`

النص الذي يوضح حالة معالجة الرسالة:

```csharp
string? messageStatus =
    statusResult.Data.MessageStatus;
```

### `IsFinal`

توضح هل وصلت الرسالة إلى حالة نهائية:

```csharp
bool isFinal =
    statusResult.Data.IsFinal;
```

### `IsSuccessful`

توضح هل العملية الأصلية نجحت نهائيًا:

```csharp
bool isSuccessful =
    statusResult.Data.IsSuccessful;
```

### `IsApplicationError`

توضح هل فشلت العملية الأصلية وظيفيًا:

```csharp
bool isApplicationError =
    statusResult.Data.IsApplicationError;
```

### `FirstErrorMessage`

أول رسالة خطأ وظيفي مفيدة:

```csharp
string? firstErrorMessage =
    statusResult.Data.FirstErrorMessage;
```

### `LogList`

قائمة بالسجلات أو الرسائل التي أعادتها معالجة EPTTS:

```csharp
var logList =
    statusResult.Data.LogList;
```

استخدمها للتشخيص وعرض تفاصيل مفيدة عند الحاجة.

---

## 15. القرار النهائي بناءً على `MessageStatusResponse`

استخدم الترتيب التالي:

```csharp
if (!statusResult.IsSuccess)
{
    /*
     * فشل طلب الاستعلام نفسه.
     * لا تغير العملية إلى Successful أو Failed نهائيًا.
     * احتفظ بها Pending أو NeedsReview حسب نوع الخطأ.
     */

    return;
}

if (statusResult.Data == null)
{
    /*
     * لا توجد بيانات حالة منظمة.
     * احتفظ بالعملية Pending أو NeedsReview.
     */

    return;
}

if (statusResult.Data.IsSuccessful)
{
    /*
     * العملية الأصلية نجحت نهائيًا.
     * حدث ERP إلى Successful.
     */

    return;
}

if (statusResult.Data.IsApplicationError)
{
    /*
     * استعلام الحالة نجح، لكن العملية الأصلية
     * فشلت وظيفيًا.
     * حدث ERP إلى Failed.
     */

    return;
}

/*
 * الحالة ليست نهائية.
 * احتفظ بالعملية Pending.
 */
```

---

## 16. معنى S - Successful

عندما تكون:

```csharp
statusResult.Data.IsSuccessful == true
```

وتشير الحالة إلى:

```text
S - Successful
```

فهذا هو النجاح النهائي للعملية الأصلية.

عندها يمكن للـ ERP:

- تحديث سجل التكامل إلى `Successful`.
- تطبيق مسار نجاح المستند أو المخزون حسب تصميم ERP.
- تنفيذ `VerifyProduct` بعد العملية عندما يكون ذلك مطلوبًا للتأكد من حالة العبوة النهائية.
- تسجيل وقت اكتمال العملية.

---

## 17. معنى E - Application Error

قد يكون:

```csharp
statusResult.IsSuccess == true
```

وفي الوقت نفسه:

```csharp
statusResult.Data.IsApplicationError == true
```

هذا ليس تناقضًا.

المعنى هو:

```text
طلب الاستعلام عن الحالة نجح تقنيًا
لكن العملية التجارية الأصلية فشلت داخل EPTTS
```

الإجراء الصحيح:

- تحديث العملية داخل ERP إلى `Failed`.
- عدم تطبيق مسار النجاح على المخزون أو المستند.
- حفظ `FirstErrorMessage`.
- حفظ `RawResponse` وسجل المعالجة عند الحاجة.
- عرض رسالة مناسبة للمستخدم أو فريق الدعم.

مثال:

```csharp
if (statusResult.Data.IsApplicationError)
{
    string? applicationError =
        statusResult.Data.FirstErrorMessage;

    /*
     * احفظ العملية Failed.
     */
}
```

---

## 18. الحالة غير النهائية

إذا كانت:

```csharp
statusResult.IsSuccess == true
```

لكن:

```csharp
statusResult.Data.IsSuccessful == false
```

و:

```csharp
statusResult.Data.IsApplicationError == false
```

فقد تكون الرسالة ما زالت قيد المعالجة.

الإجراء:

```text
احتفظ بالعملية Pending
→ لا تعِد إرسال العملية الأصلية
→ أعد الاستعلام لاحقًا
```

---

## 19. الحالات المحلية المقترحة داخل ERP

يوصى باستخدام الحالات التالية محليًا:

### `Draft`

قبل إرسال العملية إلى EPTTS.

### `Pending`

بعد قبول الرسالة للمعالجة ووجود `StatusQueryIdentifier`.

### `Successful`

بعد نتيجة نهائية ناجحة فقط.

### `Failed`

بعد `E - Application Error` أو رفض نهائي واضح.

### `NeedsReview`

عندما تكون النتيجة غير مؤكدة، مثل:

- `Timeout` أثناء إرسال العملية الأصلية.
- إلغاء انتظار عملية تغير حالة العبوة.
- استجابة فارغة غير متوقعة.
- قبول غير واضح دون معرف متابعة.
- خطأ تقني يحتاج مراجعة قبل إعادة الإرسال.

---

## 20. خريطة تحويل النتائج إلى حالات ERP

```text
قبل الإرسال
→ Draft

HTTP 202 + معرف متابعة صالح
→ Pending

S - Successful
→ Successful

E - Application Error
→ Failed

Timeout أثناء الإرسال الأصلي أو نتيجة غير مؤكدة
→ NeedsReview

MsgStatusQuery غير نهائي
→ Pending

انتهاء مدة Polling فقط
→ Pending
```

لا تعتبر انتهاء مدة الانتظار التلقائي فشلًا للعملية الأصلية.

---

## 21. الاستثناءات التي ترميها المكتبة

تستخدم المكتبة Exceptions للأخطاء التي يمكن اكتشافها محليًا قبل أو أثناء إعداد الطلب.

أهم الأنواع:

```csharp
EpttsConfigurationException
EpttsValidationException
EpttsException
```

قد توجد أنواع مشتقة أو تفاصيل إضافية حسب إصدار المكتبة.

### Namespace المطلوب

```csharp
using Eptts.Client.Exceptions;
```

---

## 22. `EpttsConfigurationException`

يُستخدم عند وجود إعدادات اتصال أو صيدلية غير صحيحة.

أمثلة:

- `BaseUrl` فارغ أو غير صالح.
- `IntegratorKey` فارغ.
- `PharmacyGln` غير صحيح.
- `PharmacySgln` غير صحيح.
- `Timeout` غير صالح.

مثال:

```csharp
try
{
    options.Validate();
}
catch (EpttsConfigurationException exception)
{
    string message =
        exception.Message;
}
```

هذا الخطأ محلي، ولا يعني أن طلبًا وصل إلى EPTTS.

---

## 23. `EpttsValidationException`

يُستخدم عند وجود مدخلات عملية غير صحيحة محليًا.

أمثلة محتملة:

- `SGTIN` فارغ أو غير صالح.
- `SSCC` غير صالح.
- `Quantity` غير صالحة.
- `StatusQueryIdentifier` فارغ.
- قائمة `EPCs` فارغة.
- قيمة مطلوبة داخل Request غير موجودة.

مثال:

```csharp
catch (EpttsValidationException exception)
{
    string message =
        exception.Message;

    /*
     * اطلب من المستخدم تصحيح المدخلات.
     */
}
```

عادة لا يُرسل HTTP Request إذا فشل التحقق المحلي قبل الإرسال.

---

## 24. `EpttsException`

هو نوع عام لأخطاء المكتبة أو أخطاء التكامل المرتبطة بها.

مثال:

```csharp
catch (EpttsException exception)
{
    string message =
        exception.Message;
}
```

يجب وضعه بعد الأنواع الأكثر تحديدًا:

```csharp
catch (EpttsConfigurationException exception)
{
}
catch (EpttsValidationException exception)
{
}
catch (EpttsException exception)
{
}
```

---

## 25. ترتيب `catch` الصحيح

استخدم الترتيب التالي:

```csharp
try
{
    /*
     * إنشاء Client واستدعاء العملية.
     */
}
catch (EpttsConfigurationException exception)
{
    /*
     * خطأ في إعدادات الاتصال أو الصيدلية.
     */
}
catch (EpttsValidationException exception)
{
    /*
     * خطأ في مدخلات العملية.
     */
}
catch (EpttsException exception)
{
    /*
     * خطأ آخر من مكتبة EPTTS.
     */
}
catch (OperationCanceledException)
{
    /*
     * إلغاء محلي إذا كان الـ Overload يرمي هذا النوع.
     */
}
catch (Exception exception)
{
    /*
     * خطأ غير متوقع في تطبيق ERP.
     * سجله للمراجعة.
     */
}
```

قد تعيد بعض حالات الإلغاء أو `Timeout` داخل `EpttsResult<T>` بدل رمي Exception، لذلك يجب دائمًا معالجة كل من:

```text
Exceptions
و
result.IsSuccess == false
```

---

## 26. الفرق بين Exception وFailure Result

### Exception

يُستخدم عادة عندما لا يمكن متابعة تنفيذ الاستدعاء بصورة طبيعية.

مثال:

```text
إعداد فارغ
مدخل غير صالح محليًا
خطأ استخدام للمكتبة
```

### Failure Result

تعيد الدالة كائن `EpttsResult<T>` بقيمة:

```csharp
IsSuccess == false
```

في حالات مثل:

- رفض HTTP.
- `Timeout`.
- `Cancellation`.
- فشل اتصال.
- خطأ خادم.
- استجابة غير صالحة.

يجب ألا يعتمد ERP على `try/catch` وحده.

الترتيب الصحيح:

```text
try/catch حول الاستدعاء
ثم
فحص result.IsSuccess
ثم
فحص result.Data
ثم
فحص خصائص الاستجابة الوظيفية
```

---

## 27. القالب الكامل لمعالجة الاستدعاء

```csharp
try
{
    using (EpttsClient client =
        new EpttsClient(options))
    {
        EpttsResult<VerifyProductResponse> result =
            await client.VerifyProductAsync(
                sgtin,
                cancellationToken);

        if (!result.IsSuccess)
        {
            /*
             * عالج فشل HTTP أو Timeout أو Cancellation.
             */

            return;
        }

        if (result.Data == null)
        {
            /*
             * استجابة منظمة فارغة.
             */

            return;
        }

        /*
         * اقرأ النتيجة الوظيفية.
         */
    }
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

---

## 28. أخطاء HTTP الشائعة

### HTTP 400

يعني أن الطلب غير صالح أو يحتوي على بيانات غير مقبولة.

راجع:

- المدخلات.
- `RawRequest`.
- `ErrorCode`.
- `ErrorMessage`.
- `RawResponse`.

### HTTP 401

يعني عادة أن بيانات المصادقة غير مقبولة.

راجع:

- `IntegratorKey`.
- البيئة المستخدمة.
- عدم استخدام مفتاح بيئة أخرى.

لا تسجل قيمة المفتاح في Logs.

### HTTP 403

يعني أن الطلب مفهوم لكن الجهة غير مخولة بالعملية أو المورد.

راجع:

- صلاحية `IntegratorKey`.
- `PharmacyGln`.
- ملكية الرسالة عند الاستعلام.
- صلاحية الصيدلية على العملية.

### HTTP 404

يعني أن المورد أو الرسالة أو المسار غير موجود.

راجع:

- `BaseUrl`.
- البيئة.
- `StatusQueryIdentifier`.
- عمر الرسالة وسياسة البيئة.

لا تفترض أن العملية الأصلية فشلت تجاريًا بسبب فشل استعلام لاحق وحده.

### HTTP 409

قد يشير إلى تعارض أو عملية مكررة أو حالة لا تسمح بالعملية.

راجع تفاصيل الاستجابة قبل إعادة الإرسال.

### HTTP 422

قد يشير إلى أن الطلب صحيح من ناحية البنية لكنه غير مقبول وظيفيًا.

راجع `ErrorMessage` و`RawResponse`.

### HTTP 429

يعني وجود عدد طلبات أكبر من المسموح خلال فترة معينة.

طبق سياسة Retry متدرجة ومحكومة، ولا تُعد إرسال عملية تغير حالة العبوة دون التأكد من نتيجة المحاولة السابقة.

### HTTP 500 أو 502 أو 503 أو 504

تشير إلى خطأ خادم أو وسيط أو عدم توفر مؤقت.

بالنسبة لعمليات القراءة، يمكن إعادة المحاولة وفق سياسة ERP.

بالنسبة إلى عمليات تغيير الحالة، لا تعِد الإرسال تلقائيًا قبل التحقق من احتمال وصول الرسالة السابقة.

---

## 29. تصنيف الأخطاء داخل ERP

يوصى بتصنيف الأخطاء إلى الفئات التالية.

### خطأ إعدادات

أمثلة:

```text
BaseUrl غير صحيح
IntegratorKey فارغ
PharmacyGln غير صحيح
```

الإجراء:

```text
منع الإرسال
→ مطالبة مسؤول الإعدادات بالتصحيح
```

### خطأ مدخلات

أمثلة:

```text
SGTIN فارغ
Quantity غير صالحة
ReturnRequestNumber مفقود
```

الإجراء:

```text
منع الإرسال
→ مطالبة المستخدم بتصحيح البيانات
```

### خطأ مصادقة أو صلاحيات

أمثلة:

```text
HTTP 401
HTTP 403
```

الإجراء:

```text
عدم تكرار الطلب بلا تغيير
→ مراجعة المفتاح والصيدلية والصلاحيات
```

### خطأ اتصال مؤقت

أمثلة:

```text
Timeout
HTTP 502
HTTP 503
HTTP 504
```

الإجراء يعتمد على نوع العملية:

```text
عملية قراءة
→ يمكن إعادة المحاولة

عملية تغير الحالة
→ تحقق من الرسالة السابقة قبل إعادة الإرسال
```

### خطأ وظيفي نهائي

مثال:

```text
E - Application Error
```

الإجراء:

```text
Failed
→ حفظ FirstErrorMessage
→ عدم تطبيق مسار النجاح
```

---

## 30. التعامل مع `Timeout`

قد تعيد المكتبة:

```text
ErrorCode = REQUEST_TIMEOUT
```

يعني ذلك أن مدة انتظار HTTP انتهت.

لا يثبت `Timeout` أن EPTTS لم يستلم الطلب.

### في عمليات القراءة

مثل `VerifyProduct` أو `GetMessageStatus`:

```text
يمكن إعادة المحاولة وفق سياسة ERP
```

لأن العملية لا تغير حالة العبوة.

### في عمليات تغيير الحالة

مثل الصرف أو المرتجع:

```text
لا تعِد الإرسال مباشرة
```

الإجراء الصحيح:

1. احفظ العملية بحالة `NeedsReview`.
2. احفظ `RawRequest` و`RawResponse`.
3. ابحث عن `instanceIdentifier` في الطلب إذا لزم.
4. استخدم معرف المتابعة إذا توفر.
5. نفذ `GetMessageStatusAsync`.
6. لا ترسل عملية جديدة قبل معرفة نتيجة الرسالة السابقة.

---

## 31. Timeout الخاص بطلب HTTP ومدة المعالجة النهائية

هناك نوعان مختلفان من الوقت.

### `EpttsOptions.Timeout`

مهلة طلب HTTP واحد:

```csharp
options.Timeout =
    TimeSpan.FromSeconds(60);
```

### `maximumWaitTime`

أقصى مدة انتظار عند استخدام:

```csharp
WaitForFinalStatusAsync
```

مثال:

```csharp
maximumWaitTime:
    TimeSpan.FromSeconds(30)
```

انتهاء `maximumWaitTime` يعني أن التطبيق توقف عن Polling بعد المدة المحددة.

لا يعني أن العملية الأصلية فشلت.

احتفظ بالعملية `Pending` واستعلم عنها لاحقًا.

---

## 32. التعامل مع `CancellationToken`

تقبل الدوال Overloads تدعم `CancellationToken`.

مثال:

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

يمكن لواجهة ERP استدعاء:

```csharp
source.Cancel();
```

لإيقاف انتظار المستخدم.

لكن الإلغاء محلي فقط.

لا يعني:

- حذف الرسالة من EPTTS.
- إلغاء عملية الصرف.
- إلغاء عملية المرتجع.
- إعادة حالة العبوة.

---

## 33. `REQUEST_CANCELLED`

قد تعيد المكتبة:

```text
ErrorCode = REQUEST_CANCELLED
```

أو قد يُرمى:

```csharp
OperationCanceledException
```

بحسب الدالة وتوقيت الإلغاء وطريقة معالجة المكتبة.

لذلك عالج الحالتين.

### عملية قراءة

يمكن للمستخدم إعادة الاستعلام لاحقًا.

### عملية تغير الحالة

تعامل معها كحالة غير مؤكدة إذا كان الطلب قد بدأ بالفعل.

لا تعِد الإرسال قبل مراجعة حالة الرسالة السابقة.

---

## 34. `WaitForFinalStatusAsync` والإلغاء

مثال:

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

إلغاء هذه الدالة يعني:

```text
إيقاف Polling داخل التطبيق
```

ولا يعني:

```text
إلغاء العملية الأصلية
```

يجب الاحتفاظ بـ `StatusQueryIdentifier` وإعادة الاستعلام لاحقًا.

---

## 35. Retry الآمن

لا تطبق Retry واحدًا على جميع العمليات.

### عمليات القراءة

مثل:

```text
VerifyProduct
GetMessageStatus
```

يمكن إعادة المحاولة عند فشل مؤقت وفق سياسة محدودة.

مثال سياسة بسيطة:

```text
المحاولة الأولى
→ انتظار قصير
→ المحاولة الثانية
→ إيقاف وإظهار الخطأ
```

### عمليات تغيير الحالة

مثل:

```text
Receiving
Dispensing
Dispense Cancel
Return
Return Cancel
```

لا تعِد إرسال الطلب تلقائيًا عند `Timeout` أو انقطاع اتصال غير مؤكد.

أولًا:

```text
ابحث عن معرف الرسالة
→ نفذ MsgStatusQuery
→ تأكد من نتيجة المحاولة السابقة
```

إعادة الإرسال غير المحكومة قد تنتج عملية مكررة.

---

## 36. ما الذي يجب تسجيله؟

يمكن تسجيل:

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
CreatedAt
CompletedAt
```

لا تسجل:

```text
IntegratorKey
Authorization Headers
كلمات المرور
بيانات دخول المستخدم
بيانات مريض شخصية مباشرة
```

يجب أن تكون Logs محمية وفق سياسة الشركة.

---

## 37. عرض الخطأ للمستخدم مقابل تسجيله للدعم

### للمستخدم

اعرض رسالة قصيرة قابلة للفهم، مثل:

```text
تعذر التحقق من العبوة. يرجى المحاولة مرة أخرى.
```

أو:

```text
فشلت العملية داخل EPTTS: العبوة غير مملوكة للصيدلية الحالية.
```

### لسجل الدعم

احفظ التفاصيل الفنية:

```text
HttpStatusCode
ErrorCode
ErrorMessage
StatusQueryIdentifier
RawRequest
RawResponse
```

لا تعرض `RawResponse` كاملة للمستخدم العادي إلا في شاشة اختبار أو دعم مخولة.

---

## 38. قالب موحد لتسجيل الفشل

المثال التالي توضيحي. يجب على ERP تنفيذ `SaveIntegrationFailure` حسب طريقة التخزين المستخدمة لديه.

```csharp
if (!result.IsSuccess)
{
    SaveIntegrationFailure(
        operationType:
            "VerifyProduct",

        httpStatusCode:
            result.HttpStatusCode,

        errorCode:
            result.ErrorCode,

        errorMessage:
            result.ErrorMessage,

        rawRequest:
            result.RawRequest,

        rawResponse:
            result.RawResponse);

    return;
}
```

الدالة:

```csharp
SaveIntegrationFailure
```

ليست جزءًا من المكتبة. يجب أن ينشئها فريق ERP أو يستبدلها بخدمة التسجيل الموجودة لديه.

---

## 39. منع كشف الأسرار في الأخطاء

لا تنشئ رسالة خطأ تحتوي على:

```csharp
options.IntegratorKey
```

خطأ:

```csharp
logger.LogError(
    "IntegratorKey: " +
    options.IntegratorKey);
```

الصحيح:

```csharp
logger.LogError(
    "EPTTS authentication failed for PharmacyGln: " +
    options.PharmacyGln);
```

لا تحفظ HTTP Headers السرية ضمن `RawRequest` أو Logs.

---

## 40. مثال كامل لمعالجة عملية غير متزامنة

```csharp
try
{
    EpttsResult<SubmissionResponse> submission =
        await client.ReturnToBranchAsync(
            request,
            cancellationToken);

    if (!submission.IsSuccess)
    {
        /*
         * عالج الفشل التقني.
         * لا تعتبر العملية ناجحة.
         */

        return;
    }

    if (submission.Data == null)
    {
        /*
         * استجابة غير متوقعة.
         * احفظ NeedsReview.
         */

        return;
    }

    if (!submission.Data.CanQueryFinalStatus ||
        string.IsNullOrWhiteSpace(
            submission.Data.StatusQueryIdentifier))
    {
        /*
         * لا يوجد معرف متابعة صالح.
         * احفظ NeedsReview.
         */

        return;
    }

    string identifier =
        submission.Data.StatusQueryIdentifier;

    /*
     * احفظ العملية Pending هنا.
     */

    EpttsResult<MessageStatusResponse> statusResult =
        await client.GetMessageStatusAsync(
            identifier,
            cancellationToken);

    if (!statusResult.IsSuccess ||
        statusResult.Data == null)
    {
        /*
         * لا تغير العملية إلى Failed نهائيًا.
         * احتفظ بها Pending أو NeedsReview.
         */

        return;
    }

    if (statusResult.Data.IsSuccessful)
    {
        /*
         * حدث العملية إلى Successful.
         */

        return;
    }

    if (statusResult.Data.IsApplicationError)
    {
        string? applicationError =
            statusResult.Data.FirstErrorMessage;

        /*
         * حدث العملية إلى Failed.
         */

        return;
    }

    /*
     * ما زالت Pending.
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
    /* خطأ غير متوقع داخل ERP. */
}
```

> المثال السابق يوضح التسلسل فقط. داخل ERP الحقيقي يُفضل غالبًا فصل الإرسال عن متابعة الحالة باستخدام سجل `Pending` وJob مستقل.

---

## 41. قائمة إتمام الجزء الثالث

قبل الانتقال إلى Endpoints القراءة، تأكد من فهم النقاط التالية:

```text
[ ] أعرف معنى EpttsResult<T>
[ ] أفحص IsSuccess قبل Data
[ ] أفحص Data قبل خصائصها
[ ] أعرف الفرق بين Data وRawResponse
[ ] أعرف الفرق بين IsSuccess وVerified
[ ] أعرف خصائص SubmissionResponse الأساسية
[ ] أعرف أن HTTP 202 ليس نجاحًا نهائيًا
[ ] أحفظ StatusQueryIdentifier فور قبوله
[ ] أعرف خصائص MessageStatusResponse الأساسية
[ ] أعرف معنى IsSuccessful
[ ] أعرف معنى IsApplicationError
[ ] أعرف استخدام FirstErrorMessage
[ ] أعرف متى تكون العملية Pending
[ ] أعرف متى تكون Successful
[ ] أعرف متى تكون Failed
[ ] أعرف متى أستخدم NeedsReview
[ ] أعرف الفرق بين Exception وFailure Result
[ ] أعالج EpttsConfigurationException
[ ] أعالج EpttsValidationException
[ ] أعالج EpttsException
[ ] أعرف معنى HTTP 401 و403 و404
[ ] أعرف أن Timeout لا يثبت عدم وصول الرسالة
[ ] أعرف أن Cancellation لا تلغي العملية الأصلية
[ ] لا أعيد إرسال عمليات تغيير الحالة تلقائيًا
[ ] لا أسجل IntegratorKey
```

---

## 42. خلاصة الجزء الثالث

اقرأ كل نتيجة بهذا الترتيب:

```text
1. هل حدث Exception؟
2. هل IsSuccess = true؟
3. هل Data موجودة؟
4. ما النتيجة الوظيفية داخل Data؟
5. هل العملية نهائية أم Pending؟
6. ما الحالة التي يجب حفظها داخل ERP؟
```

للعمليات غير المتزامنة:

```text
Submission ناجح وقابل للمتابعة
→ Pending

Message Status = S - Successful
→ Successful

Message Status = E - Application Error
→ Failed

حالة غير نهائية
→ Pending

Timeout أو نتيجة غير مؤكدة أثناء الإرسال
→ NeedsReview
```

ولا تعتمد نجاح العملية النهائية قبل قراءة `MessageStatusResponse`.
