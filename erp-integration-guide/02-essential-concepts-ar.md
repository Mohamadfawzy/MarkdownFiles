# الجزء الثاني: المفاهيم الضرورية

> يشرح هذا الجزء المصطلحات التي يحتاجها مطور ERP لفهم بيانات EPTTS واستخدام المكتبة بطريقة صحيحة. لا يحتاج المطور إلى دراسة EPCIS أو GS1 بالتفصيل، لكن يجب فهم الفرق بين معرف العبوة، ومعرف الشحنة، ومعرفات الرسائل، ورقم المرتجع.

---

## 1. لماذا يجب فهم هذه المفاهيم؟

تستخدم عمليات EPTTS عدة معرفات متشابهة في الشكل، لكن لكل معرف غرض مختلف.

الخلط بينها قد يؤدي إلى:

- الاستعلام عن الرسالة باستخدام معرف غير صحيح.
- إرسال `GTIN` بدل `SGTIN`.
- استخدام `MessageId` بدل `ReturnRequestNumber`.
- محاولة استلام عبوة باستخدام `SSCC` غير صحيح.
- إرسال عملية على صيدلية أو فرع غير مقصود.

أهم قاعدة في هذا الجزء:

```text
معرف العبوة ليس معرف الشحنة
ومعرف الرسالة ليس رقم المرتجع
```

---

## 2. ما هو EPTTS؟

`EPTTS` هو النظام الذي يستقبل عمليات تتبع العبوات الدوائية ويعالجها.

يتعامل نظام ERP مع EPTTS من خلال المكتبة لتنفيذ عمليات مثل:

- التحقق من عبوة.
- استلام شحنة.
- صرف عبوة كاملة.
- صرف كمية جزئية.
- إلغاء الصرف.
- إرسال مرتجع إلى فرع.
- إلغاء مرتجع.
- متابعة نتيجة رسالة سبق إرسالها.

بعض العمليات تكون فورية من ناحية HTTP، مثل `VerifyProduct`، بينما عمليات أخرى تُقبل أولًا ثم تُعالج بصورة غير متزامنة.

في العمليات غير المتزامنة يكون التسلسل:

```text
إرسال العملية
→ قبول الرسالة للمعالجة
→ الحصول على معرف متابعة
→ الاستعلام عن النتيجة النهائية
```

قبول الرسالة لا يعني أن العملية التجارية نجحت نهائيًا. سيتم شرح ذلك بالتفصيل في الجزء الثالث.

---

## 3. ما هو EPC URI؟

تستخدم المكتبة معرفات بصيغة نصية تسمى في هذا الدليل `EPC URI`.

أمثلة:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
```

```text
urn:epc:id:sscc:80026600.000444363
```

```text
urn:epc:id:sgln:6221388.35823.0
```

توضح بداية المعرف نوعه:

```text
urn:epc:id:sgtin:
→ معرف عبوة مسلسلة

urn:epc:id:sscc:
→ معرف شحنة أو حاوية لوجستية

urn:epc:id:sgln:
→ معرف موقع
```

يجب تمرير المعرف كاملًا إلى المكتبة، بما في ذلك الجزء:

```text
urn:epc:id:...
```

لا تحذف بداية المعرف ولا تغير ترتيب أجزائه.

---

## 4. ما هو GTIN؟

`GTIN` هو معرف نوع المنتج التجاري.

يميز المنتج نفسه، لكنه لا يميز عبوة مادية محددة عن عبوة أخرى من المنتج نفسه.

مثال توضيحي:

```text
GTIN
→ يحدد نوع المنتج

Serial Number
→ يحدد الرقم المسلسل لعبوة محددة

SGTIN
→ يجمع هوية المنتج مع الرقم المسلسل للعبوة
```

لا تستخدم `GTIN` وحده عندما تطلب الدالة `SGTIN`.

مثال خطأ:

```csharp
request.Sgtin =
    "06290009990001";
```

مثال صحيح:

```csharp
request.Sgtin =
    "urn:epc:id:sgtin:629000999.0001.SBX104047";
```

> قد تختلف القيم الحقيقية حسب المنتج والبيئة. استخدم المعرف الذي توفره بيانات العبوة أو استجابة EPTTS.

---

## 5. ما هو SGTIN؟

`SGTIN` هو المعرف المسلسل لعبوة دوائية محددة.

مثال:

```text
urn:epc:id:sgtin:629000999.0001.SBX104047
```

يستخدم `SGTIN` في العمليات التي تتعامل مع عبوة بعينها، مثل:

- `VerifyProductAsync`.
- `DispenseFullPackAsync`.
- `DispensePartialAsync`.
- `CancelDispensingAsync`.
- `ReturnToBranchAsync`.
- `CancelReturnAsync`.

### ما الذي يمثله SGTIN؟

يمثل عبوة واحدة محددة، وليس نوع المنتج كله، وليس مجموعة عبوات.

عبوتان من المنتج نفسه يجب أن يكون لكل منهما رقم مسلسل مختلف، وبالتالي `SGTIN` مختلف.

### إدخال SGTIN

يجب استخدام المعرف كاملًا:

```csharp
string sgtin =
    "urn:epc:id:sgtin:629000999.0001.SBX104047";
```

ثم تمريره إلى الدالة أو Request المطلوبة:

```csharp
var result =
    await client.VerifyProductAsync(
        sgtin,
        cancellationToken);
```

أو:

```csharp
var request =
    new FullDispensingRequest
    {
        Sgtin = sgtin,
        EventDateTime = DateTimeOffset.Now
    };
```

### أخطاء شائعة

لا تضع داخل خاصية `Sgtin`:

- `GTIN` فقط.
- `SSCC`.
- نص DataMatrix الخام إذا لم تحول المكتبة النص تلقائيًا.
- رقم الصنف الداخلي في ERP.
- `ItemID` أو `ItemRefNo`.

خاصية `Sgtin` تحتاج معرف العبوة المسلسلة الذي تقبله EPTTS.

---

## 6. ما هو SSCC؟

`SSCC` هو معرف حاوية أو شحنة لوجستية.

مثال:

```text
urn:epc:id:sscc:80026600.000444363
```

يمثل `SSCC` مجموعة لوجستية قد تحتوي على عدة عبوات.

يستخدم عادة في عملية:

```text
Receiving from Branch
```

عند استلام حاوية أو شحنة كاملة.

### مثال استلام SSCC

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
                "urn:epc:id:sscc:80026600.000444363"
            },

        EventDateTime =
            DateTimeOffset.Now
    };
```

المتغيران:

```csharp
sourceBranchGln
sourceBranchSgln
```

يمثلان بيانات الفرع المرسل، ويجب أن يوفرهما ERP.

### SSCC أم SGTINs؟

عند استلام شحنة كاملة داخل `SSCC`، يرسل ERP معرف `SSCC`.

عند استلام عبوات منفردة دون حاوية واحدة، يمكن إرسال قائمة `SGTINs`:

```csharp
EpcList =
    new List<string>
    {
        firstSgtin,
        secondSgtin
    };
```

لا ترسل `SSCC` وجميع محتوياته من `SGTINs` في الطلب نفسه، إلا إذا كانت مواصفة العملية المعتمدة تطلب ذلك صراحة.

---

## 7. الفرق بين SGTIN وSSCC

### SGTIN

```text
يمثل عبوة واحدة محددة
```

يستخدم في:

```text
VerifyProduct
Dispensing
Dispense Cancel
Return
Return Cancel
```

### SSCC

```text
يمثل حاوية أو شحنة لوجستية
```

يستخدم عادة في:

```text
Receiving
```

### اختيار المعرف الصحيح

استخدم هذا القرار:

```text
هل العملية على عبوة محددة؟
→ استخدم SGTIN

هل العملية على حاوية أو شحنة كاملة؟
→ استخدم SSCC
```

---

## 8. ما هو GLN؟

`GLN` هو معرف جهة أو موقع تجاري.

في الاستخدام الحالي يكون عادة رقمًا مكونًا من 13 رقمًا.

مثال:

```text
6221388358239
```

تستخدم المكتبة عدة قيم GLN، ويجب عدم الخلط بينها.

### PharmacyGln

يمثل الصيدلية التي ينفذ ERP العملية باسمها:

```csharp
options.PharmacyGln =
    "6221388358239";
```

هذه القيمة جزء من إعدادات الاتصال.

### SourceGln

يمثل الجهة أو الفرع الذي أرسل الشحنة في عملية `Receiving`:

```csharp
request.SourceGln =
    sourceBranchGln;
```

### DestinationGln

يمثل الفرع الذي سيستقبل المرتجع:

```csharp
request.DestinationGln =
    destinationBranchGln;
```

### CurrentGln

قيمة تعيدها `VerifyProduct` لتوضح الجهة المرتبطة حاليًا بالعبوة:

```csharp
string currentGln =
    result.Data.Pack.CurrentGln;
```

لمعرفة هل العبوة مرتبطة بالصيدلية الحالية:

```csharp
bool ownedByCurrentPharmacy =
    string.Equals(
        result.Data.Pack.CurrentGln,
        options.PharmacyGln,
        StringComparison.Ordinal);
```

> المقارنة السابقة هي تحقق محلي مساعد. القرار النهائي وقواعد الملكية تطبقها EPTTS عند معالجة العملية.

---

## 9. ما هو SGLN؟

`SGLN` هو معرف الموقع بصيغة EPC URI.

مثال:

```text
urn:epc:id:sgln:6221388.35823.0
```

### PharmacySgln

يمثل موقع الصيدلية الحالية:

```csharp
options.PharmacySgln =
    "urn:epc:id:sgln:6221388.35823.0";
```

### SourceSgln

يمثل موقع الفرع المرسل في `Receiving`:

```csharp
request.SourceSgln =
    sourceBranchSgln;
```

### DestinationSgln

يمثل موقع الفرع المستقبل في `Return to Branch`:

```csharp
request.DestinationSgln =
    destinationBranchSgln;
```

يجب أن تتوافق كل قيمة `SGLN` مع الجهة التي يمثلها `GLN` المرتبط بها.

مثال:

```text
PharmacyGln
↔ PharmacySgln

SourceGln
↔ SourceSgln

DestinationGln
↔ DestinationSgln
```

---

## 10. الفرق بين GLN وSGLN

### GLN

معرف الجهة أو الموقع التجاري بصيغة رقمية:

```text
6221388358239
```

### SGLN

تمثيل الموقع بصيغة EPC URI:

```text
urn:epc:id:sgln:6221388.35823.0
```

لا تنشئ `SGLN` يدويًا من `GLN` ما لم تكن قواعد تكوينه معتمدة وواضحة في نظامك.

الأفضل أن يحفظ ERP القيمتين ضمن إعدادات كل صيدلية أو فرع:

```csharp
public sealed class ErpLocationSettings
{
    public string Gln { get; set; } =
        string.Empty;

    public string Sgln { get; set; } =
        string.Empty;
}
```

هذه Class مثال من ERP وليست جزءًا من المكتبة.

---

## 11. ما هو DataMatrix؟

قد يقرأ جهاز المسح نصًا من رمز `DataMatrix` الموجود على العبوة.

قد يحتوي النص المقروء على بيانات مثل:

- `GTIN`.
- الرقم المسلسل.
- رقم التشغيلة.
- تاريخ الانتهاء.

لكن خاصية `Sgtin` في Requests تحتاج المعرف الذي تقبله EPTTS، مثل:

```text
urn:epc:id:sgtin:...
```

إذا كان ERP يحصل على نص DataMatrix خام، فيجب أن تكون هناك خطوة واضحة لتحويله أو تحليله قبل تمريره إلى المكتبة، حسب صيغة البيانات المعتمدة في النظام.

لا تفترض أن نص DataMatrix الخام يساوي `SGTIN` دائمًا.

في هذا الدليل سنفترض أن ERP حصل بالفعل على `SGTIN` الصحيح قبل استدعاء العمليات.

---

## 12. ما هو InstanceIdentifier؟

`InstanceIdentifier` هو معرف رسالة يستخدم لتمييز طلب EPCIS محدد.

تنشئ المكتبة معرفًا جديدًا عند بناء رسالة عملية مثل:

- `Receiving`.
- `Full Pack Dispensing`.
- `Partial Dispensing`.
- `Dispense Cancel`.
- `Return`.
- `Return Cancel`.

قد يظهر في النتيجة باسم:

```csharp
RequestInstanceIdentifier
```

مثال قراءة القيمة:

```csharp
string? requestInstanceIdentifier =
    result.Data.RequestInstanceIdentifier;
```

يجب حفظ هذه القيمة مع سجل العملية لأغراض:

- التتبع.
- الدعم الفني.
- مراجعة الرسالة عند حدوث Timeout.
- الربط بين الطلب المحلي ورسالة EPTTS.

لا تستخدم معرف عملية قديمة عند إنشاء عملية جديدة.

---

## 13. ما هو RequestInstanceIdentifier؟

`RequestInstanceIdentifier` هو الاسم المستخدم داخل `SubmissionResponse` للمعرف الذي ارتبط بالطلب المرسل.

مثال:

```csharp
string? requestInstanceIdentifier =
    submission.Data.RequestInstanceIdentifier;
```

في هذا الدليل سنستخدم المصطلحين كالتالي:

```text
InstanceIdentifier
→ المفهوم العام لمعرف رسالة EPCIS

RequestInstanceIdentifier
→ الخاصية التي تقرأ منها معرف الطلب في SubmissionResponse
```

عند حفظ العملية في ERP، احفظ القيمة كما أعادتها المكتبة دون تغيير.

---

## 14. ما هو MessageId؟

`MessageId` هو معرف تعيده EPTTS عند قبول الرسالة أو معالجتها حسب الاستجابة.

يمكن قراءته من:

```csharp
string? messageId =
    submission.Data.MessageId;
```

يستخدم `MessageId` في:

- التتبع.
- الربط مع سجلات الدعم.
- مراجعة استجابة EPTTS.

لا تستخدم `MessageId` بدل:

- `SGTIN`.
- `SSCC`.
- `StatusQueryIdentifier`.
- `ReturnRequestNumber`.

---

## 15. ما هو StatusQueryIdentifier؟

`StatusQueryIdentifier` هو المعرف الذي تستخدمه المكتبة للاستعلام عن النتيجة النهائية لعملية غير متزامنة.

يمكن قراءته من `SubmissionResponse`:

```csharp
string? statusQueryIdentifier =
    submission.Data.StatusQueryIdentifier;
```

ثم استخدامه:

```csharp
var statusResult =
    await client.GetMessageStatusAsync(
        statusQueryIdentifier,
        cancellationToken);
```

أو:

```csharp
var finalResult =
    await client.WaitForFinalStatusAsync(
        statusQueryIdentifier,
        pollingInterval:
            TimeSpan.FromSeconds(3),
        maximumWaitTime:
            TimeSpan.FromSeconds(30),
        cancellationToken:
            cancellationToken);
```

### متى نحفظه؟

احفظه فور قبول العملية إذا كانت:

```csharp
submission.Data.CanQueryFinalStatus == true
```

ولا تعتمد على بقائه داخل الذاكرة أو الشاشة فقط.

في ERP الحقيقي يجب حفظه في قاعدة البيانات مع سجل العملية.

---

## 16. العلاقة بين RequestInstanceIdentifier وMessageId وStatusQueryIdentifier

بعد إرسال عملية قد تظهر القيم التالية:

```csharp
submission.Data.RequestInstanceIdentifier
submission.Data.MessageId
submission.Data.StatusQueryIdentifier
```

كل قيمة لها وظيفة مختلفة:

### `RequestInstanceIdentifier`

معرف الطلب الذي أنشأته المكتبة أو ارتبط بالرسالة المرسلة.

### `MessageId`

معرف أعادته EPTTS للرسالة.

### `StatusQueryIdentifier`

المعرف الذي يجب تمريره إلى `GetMessageStatusAsync` أو `WaitForFinalStatusAsync`.

لا تفترض أن القيم الثلاث متساوية، حتى إذا تطابقت في استجابة معينة.

استخدم دائمًا الخاصية المخصصة للغرض المطلوب.

---

## 17. ما هو ReturnRequestNumber؟

`ReturnRequestNumber` هو رقم تجاري ينشئه ERP لعملية المرتجع.

مثال:

```text
RET-UAT-000123
```

أو يمكن أن يكون رقم مستند المرتجع في ERP، بشرط أن يكون فريدًا حسب سياسة النظام.

يتم تمريره عند إرسال المرتجع:

```csharp
ReturnRequest request =
    new ReturnRequest
    {
        Sgtin =
            sgtin,

        DestinationGln =
            destinationGln,

        DestinationSgln =
            destinationSgln,

        ReturnRequestNumber =
            returnRequestNumber,

        EventDateTime =
            DateTimeOffset.Now
    };
```

ويجب حفظه لأن `Return Cancel` يحتاج نفس القيمة:

```csharp
ReturnCancelRequest cancelRequest =
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

في المثال السابق، `originalReturn` هو سجل المرتجع الأصلي داخل ERP، وليس Class توفرها المكتبة.

---

## 18. أهم قاعدة في Return Cancel

يجب استخدام القيم الأصلية نفسها:

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

العلاقة الصحيحة:

```text
ReturnRequestNumber ≠ MessageId
ReturnRequestNumber ≠ RequestInstanceIdentifier
ReturnRequestNumber ≠ StatusQueryIdentifier
```

إذا استُخدمت قيمة مختلفة، قد تعجز EPTTS عن العثور على المرتجع المعلق.

---

## 19. ما هو EventDateTime؟

`EventDateTime` هو الوقت الفعلي لحدوث العملية التجارية.

مثال:

```csharp
EventDateTime =
    DateTimeOffset.Now;
```

يستخدم في Requests الخاصة بالعمليات، مثل:

- `ReceivingRequest`.
- `FullDispensingRequest`.
- `PartialDispensingRequest`.
- `DispenseCancelRequest`.
- `ReturnRequest`.
- `ReturnCancelRequest`.

استخدم `DateTimeOffset` بدل `DateTime` عندما تطلب الخاصية ذلك، لأنه يحتفظ بمعلومة فرق التوقيت.

لا تستخدم وقتًا افتراضيًا مثل:

```csharp
default(DateTimeOffset)
```

ويجب أن تكون ساعة الجهاز أو الخادم مضبوطة بصورة صحيحة.

---

## 20. ما هو PatientReference؟

`PatientReference` هو مرجع داخلي اختياري يربط عملية الصرف بسجل أو معاملة داخل ERP.

مثال مناسب:

```text
ERP-SALE-1001
```

أو:

```text
PATIENT-TRANSACTION-1001
```

لا تضع داخله بيانات شخصية مباشرة، مثل:

- اسم المريض.
- الرقم القومي.
- رقم الهاتف.
- العنوان.
- بيانات طبية تفصيلية غير مطلوبة.

مثال:

```csharp
PatientReference =
    "ERP-SALE-1001";
```

يمكن تمرير `null` إذا لم يكن المرجع مطلوبًا:

```csharp
PatientReference =
    null;
```

---

## 21. ما هو PrescriptionReference؟

`PrescriptionReference` هو مرجع اختياري للوصفة أو الروشتة داخل ERP.

مثال:

```text
ERP-RX-1001
```

يجب أن يكون مرجعًا داخليًا مناسبًا للربط، وليس نص الوصفة الطبية كاملًا.

مثال:

```csharp
PrescriptionReference =
    "ERP-RX-1001";
```

أو:

```csharp
PrescriptionReference =
    null;
```

إذا لم يوجد مرجع.

---

## 22. ما هي Quantity في Partial Dispensing؟

في عملية `Partial Dispensing` تمثل `Quantity` كمية العملية الحالية فقط.

مثال:

```csharp
Quantity =
    1;
```

لا تعني بالضرورة:

```text
قرصًا واحدًا
```

قد تمثل وحدة صرف أخرى حسب Master Data الخاصة بالمنتج.

يجب أن يعرف ERP:

- هل المنتج يسمح بالصرف الجزئي.
- ما وحدة الصرف الجزئي.
- ما الكمية المتاحة.
- ما الكمية التي تم صرفها سابقًا.

لا تخمّن وحدة `Quantity` من اسم المنتج فقط.

---

## 23. حالات العبوة الشائعة

تعيد `VerifyProduct` حالة العبوة داخل:

```csharp
result.Data.Pack.Status
```

القيم التالية هي الحالات التي يتعامل معها التكامل عادة.

### `active`

تعني أن العبوة نشطة ومتاحة لعملية مناسبة، بشرط تحقق بقية القواعد.

تُستخدم عادة قبل:

- `Full Pack Dispensing`.
- أول `Partial Dispensing`.
- `Return to Branch`.

### `in_transit`

تعني أن العبوة في مسار نقل أو شحنة.

قد تكون الحالة المتوقعة قبل `Receiving` بحسب سير العمل.

### `dispensed`

تعني أن العبوة صُرفت بالكامل.

تكون الحالة المتوقعة قبل:

```text
Dispense Cancel
```

### `partially_dispensed`

تعني أنه تم صرف جزء من العبوة.

قد تسمح العملية بصرف كمية جزئية أخرى، حسب قواعد المنتج والكمية المتبقية.

لا تستخدم مسار `Dispense Cancel` الخاص بالصرف الكامل لإلغاء الصرف الجزئي ما لم تعتمد EPTTS ذلك صراحة.

### `returned`

تعني أن العبوة دخلت في مسار مرتجع أو أصبح المرتجع مسجلًا حسب نتيجة البيئة.

قد تكون من الحالات المرتبطة بـ `Return Cancel`.

### `return_pending`

قد تظهر في بعض البيئات أو السيناريوهات للدلالة على مرتجع معلق.

يجب الاعتماد على القيم التي تعيدها البيئة الفعلية، وعدم إنشاء حالات جديدة من طرف ERP دون مواصفة.

---

## 24. الحالة وحدها لا تكفي لاتخاذ القرار

لا تعتمد على `Pack.Status` وحدها.

قبل عملية مثل الصرف، يجب أيضًا فحص:

```csharp
result.IsSuccess
result.Data.Verified
result.Data.Pack
result.Data.Pack.CurrentGln
result.Data.Pack.IsRecalled
result.Data.Alerts
```

مثال قرار مبدئي للصرف الكامل:

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

هذا تحقق محلي مبدئي.

القرار النهائي بقبول العملية يظل لدى EPTTS عند معالجة الرسالة.

---

## 25. ما هي Alerts؟

قد تعيد `VerifyProduct` قائمة تنبيهات:

```csharp
result.Data.Alerts
```

لفحص وجود تنبيهات:

```csharp
bool hasAlerts =
    result.Data.Alerts != null &&
    result.Data.Alerts.Count > 0;
```

ولقراءتها:

```csharp
if (result.Data.Alerts != null)
{
    foreach (string alert in result.Data.Alerts)
    {
        /*
         * اعرض التنبيه أو سجله حسب سياسة ERP.
         */
    }
}
```

لا تتجاهل التنبيهات عند اتخاذ قرار العملية.

يجب أن يوضح ERP للمستخدم التنبيه المفيد، مع حفظ التفاصيل اللازمة للدعم.

---

## 26. ما هو RawRequest؟

`RawRequest` هو النص الذي أرسلته المكتبة في Body الطلب.

يمكن الوصول إليه من النتيجة:

```csharp
string? rawRequest =
    result.RawRequest;
```

يستخدم لأغراض:

- الدعم الفني.
- التدقيق.
- مراجعة البيانات المرسلة.
- العثور على `instanceIdentifier` عند حدوث نتيجة غير مؤكدة.

لا تستخدم `RawRequest` بدل Classes المنظمة في منطق العمل الطبيعي.

استخدم:

```csharp
result.Data
```

لقراءة النتيجة، واستخدم `RawRequest` للتشخيص فقط.

---

## 27. ما هو RawResponse؟

`RawResponse` هو النص الأصلي الذي أعاده خادم EPTTS.

يمكن قراءته من:

```csharp
string? rawResponse =
    result.RawResponse;
```

يستخدم لأغراض:

- التحقق من الاستجابة الأصلية.
- الدعم الفني.
- التحقيق في أخطاء التحويل أو البيانات.
- حفظ سجل تدقيق للعملية.

لا تحلل `RawResponse` يدويًا داخل منطق ERP المعتاد إذا كانت `Data` متاحة.

الترتيب الطبيعي:

```text
منطق العمل
→ يستخدم Data

الدعم والتشخيص
→ يستخدم RawRequest وRawResponse
```

---

## 28. المعرفات التي يجب حفظها بعد إرسال عملية

بعد إرسال عملية غير متزامنة وقبولها، احفظ القيم التالية عند توفرها:

```csharp
submission.Data.RequestInstanceIdentifier
submission.Data.MessageId
submission.Data.StatusQueryIdentifier
submission.RawRequest
submission.RawResponse
```

واحفظ معها بيانات العملية المحلية، مثل:

```text
OperationType
ErpDocumentId
PharmacyGln
SGTIN
SSCC
Quantity
SourceGln
DestinationGln
DestinationSgln
ReturnRequestNumber
CreatedAt
CreatedBy
LocalStatus
```

القيمة الأهم لمتابعة النتيجة النهائية هي:

```csharp
StatusQueryIdentifier
```

---

## 29. مثال يوضح المعرفات داخل عملية واحدة

نفترض أن ERP أرسل مرتجعًا.

### بيانات العبوة والمرتجع

```text
SGTIN
→ urn:epc:id:sgtin:629000999.0001.SBX104047

DestinationGln
→ 6224010324336

ReturnRequestNumber
→ RET-UAT-000123
```

### بعد إرسال الرسالة

قد تعيد المكتبة:

```text
RequestInstanceIdentifier
→ معرف الطلب المرسل

MessageId
→ معرف أعادته EPTTS

StatusQueryIdentifier
→ المعرف المستخدم لمتابعة حالة الرسالة
```

### الاستخدام الصحيح

```text
للتحقق من العبوة
→ استخدم SGTIN

لمتابعة النتيجة
→ استخدم StatusQueryIdentifier

لإلغاء المرتجع
→ استخدم نفس ReturnRequestNumber
```

لا تستخدم معرفًا مكان الآخر.

---

## 30. قائمة مرجعية سريعة

```text
GTIN
→ نوع المنتج

SGTIN
→ عبوة مسلسلة محددة

SSCC
→ شحنة أو حاوية لوجستية

GLN
→ معرف جهة أو موقع تجاري

SGLN
→ معرف موقع بصيغة EPC URI

RequestInstanceIdentifier
→ معرف الطلب المرتبط بالرسالة المرسلة

MessageId
→ معرف أعادته EPTTS

StatusQueryIdentifier
→ معرف متابعة النتيجة النهائية

ReturnRequestNumber
→ رقم مرتجع تجاري ينشئه ERP

EventDateTime
→ وقت حدوث العملية التجارية

PatientReference
→ مرجع داخلي لعملية المريض أو البيع

PrescriptionReference
→ مرجع داخلي للوصفة

Quantity
→ كمية الصرف الجزئي في العملية الحالية
```

---

## 31. أخطاء شائعة يجب تجنبها

### استخدام GTIN بدل SGTIN

خطأ:

```csharp
request.Sgtin =
    gtin;
```

الصحيح:

```csharp
request.Sgtin =
    sgtin;
```

### استخدام SSCC داخل عملية صرف عبوة

عمليات الصرف تتعامل مع عبوة محددة، لذلك تحتاج `SGTIN` وليس `SSCC`.

### استخدام PharmacyGln مكان SourceGln

في `Receiving`:

```text
PharmacyGln
→ الصيدلية المستلمة

SourceGln
→ الفرع المرسل
```

### استخدام MessageId للاستعلام عن الحالة دون الرجوع للخاصية المخصصة

استخدم:

```csharp
submission.Data.StatusQueryIdentifier
```

مع دالة متابعة الحالة.

### استخدام MessageId مكان ReturnRequestNumber

خطأ:

```csharp
cancelRequest.ReturnRequestNumber =
    originalReturn.MessageId;
```

الصحيح:

```csharp
cancelRequest.ReturnRequestNumber =
    originalReturn.ReturnRequestNumber;
```

### عدم حفظ StatusQueryIdentifier

لا تعتمد على إبقاء المعرف في الشاشة أو الذاكرة فقط.

احفظه مع العملية `Pending` في ERP.

### استخدام بيانات مريض مباشرة

استخدم مرجعًا داخليًا بدل الاسم أو الرقم القومي أو الهاتف.

---

## 32. قائمة إتمام الجزء الثاني

قبل الانتقال إلى الجزء التالي، تأكد من فهم النقاط التالية:

```text
[ ] أعرف الفرق بين GTIN وSGTIN
[ ] أعرف الفرق بين SGTIN وSSCC
[ ] أعرف استخدام PharmacyGln
[ ] أعرف استخدام SourceGln
[ ] أعرف استخدام DestinationGln
[ ] أعرف الفرق بين GLN وSGLN
[ ] أعرف معنى CurrentGln
[ ] أعرف وظيفة RequestInstanceIdentifier
[ ] أعرف وظيفة MessageId
[ ] أعرف وظيفة StatusQueryIdentifier
[ ] أعرف أن ReturnRequestNumber ينشئه ERP
[ ] أعرف أن Return Cancel يستخدم نفس ReturnRequestNumber
[ ] أعرف معنى EventDateTime
[ ] أعرف أن PatientReference لا يحتوي على بيانات شخصية مباشرة
[ ] أعرف معنى Quantity في Partial Dispensing
[ ] أعرف الحالات الشائعة للعبوة
[ ] أعرف أن الحالة وحدها لا تكفي لاتخاذ القرار
[ ] أعرف أهمية Alerts
[ ] أعرف استخدام RawRequest وRawResponse للتشخيص
[ ] أعرف المعرفات التي يجب حفظها بعد إرسال العملية
```

---

## 33. خلاصة الجزء الثاني

قبل استدعاء أي عملية، حدد نوع المعرف المطلوب:

```text
عبوة محددة
→ SGTIN

شحنة أو حاوية
→ SSCC

الصيدلية الحالية
→ PharmacyGln وPharmacySgln

الفرع المرسل
→ SourceGln وSourceSgln

الفرع المستقبل للمرتجع
→ DestinationGln وDestinationSgln

متابعة نتيجة الرسالة
→ StatusQueryIdentifier

إلغاء مرتجع
→ نفس ReturnRequestNumber الأصلي
```

واحفظ دائمًا المعرفات كما أعادتها المكتبة دون تغيير.
