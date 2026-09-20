# الجزء الأول: البداية والتثبيت

> يشرح هذا الجزء كيفية إضافة مكتبة `Eptts.Client` إلى مشروع ERP جديد، وتجهيز Dependencies، وإنشاء إعدادات الاتصال، والتحقق منها، ثم إنشاء `EpttsClient` بنجاح دون إرسال أي طلب إلى EPTTS.

---

## 1. مقدمة

مكتبة `Eptts.Client` هي مكتبة C# جاهزة لربط نظام ERP بخدمات `EPTTS`.

يمرر نظام ERP البيانات المطلوبة إلى دوال واضحة، وتتولى المكتبة:

- إنشاء طلبات HTTP.
- إضافة Headers المطلوبة.
- إضافة `IntegratorKey` إلى الطلب بطريقة داخلية.
- إنشاء JSON المطلوب.
- إنشاء رسائل EPCIS للعمليات التي تحتاجها.
- إرسال الطلبات إلى EPTTS.
- تحويل الاستجابات إلى Classes يمكن استخدامها مباشرة في C#.
- إعادة نتيجة موحدة من النوع `EpttsResult<T>`.

لا يحتاج مطور ERP إلى:

- إنشاء JSON يدويًا.
- إنشاء رسائل EPCIS يدويًا.
- إضافة HTTP Headers بنفسه.
- معرفة المسارات الداخلية للـ Endpoints.
- تعديل سورس المكتبة.
- استخدام Classes الداخلية للمكتبة.

---

## 2. العمليات التي توفرها المكتبة

توفر النسخة الحالية من المكتبة العمليات التالية:

- التحقق من عبوة: `VerifyProductAsync`.
- متابعة حالة رسالة: `GetMessageStatusAsync`.
- الانتظار حتى ظهور نتيجة نهائية: `WaitForFinalStatusAsync`.
- استلام شحنة من فرع: `ReceiveFromBranchAsync`.
- صرف عبوة كاملة: `DispenseFullPackAsync`.
- صرف كمية جزئية: `DispensePartialAsync`.
- إلغاء صرف عبوة كاملة: `CancelDispensingAsync`.
- إرسال مرتجع إلى فرع: `ReturnToBranchAsync`.
- إلغاء مرتجع معلق: `CancelReturnAsync`.

سيتم شرح كل عملية بالتفصيل في الأجزاء التالية من الدليل.

---

## 3. مسؤولية المكتبة ومسؤولية ERP

### مسؤولية المكتبة

المكتبة مسؤولة عن:

- التحقق المحلي من تنسيق إعدادات الاتصال والمدخلات.
- إنشاء طلب HTTP.
- إنشاء محتوى الطلب.
- إرسال الطلب إلى EPTTS.
- قراءة استجابة EPTTS.
- تحويل الاستجابة إلى Class منظمة.
- توفير `RawRequest` و`RawResponse` لأغراض الدعم والتشخيص.

### مسؤولية ERP

نظام ERP مسؤول عن:

- تحميل إعدادات الصيدلية الحالية.
- حماية `IntegratorKey`.
- قراءة `SGTIN` أو `SSCC` من المستخدم أو جهاز المسح.
- إنشاء مستند البيع أو الاستلام أو المرتجع داخل ERP.
- حفظ العمليات المرسلة إلى EPTTS.
- حفظ `StatusQueryIdentifier`.
- متابعة العمليات المعلقة.
- منع إرسال العملية نفسها أكثر من مرة.
- تحديث المخزون والمستندات بعد ظهور النتيجة النهائية.
- عرض رسالة مفهومة للمستخدم.

المكتبة لا تقوم تلقائيًا بتعديل قاعدة بيانات ERP أو المستندات أو المخزون.

---

## 4. متطلبات الاستخدام

### بيانات المكتبة

اسم ملف المكتبة:

```text
Eptts.Client.dll
```

الـ Namespace الأساسي:

```csharp
Eptts.Client
```

Target Framework الخاص بالمكتبة:

```text
.NET Standard 2.0
```

### مشروع ERP

يجب أن يستهدف مشروع ERP إصدارًا يدعم `.NET Standard 2.0`، مثل:

- `.NET Framework 4.8`.
- `.NET 6` أو أحدث.
- `.NET 9 Windows`.
- WPF.
- Windows Forms.
- ASP.NET.
- Console Application.
- Windows Service.

يجب اختبار المكتبة داخل نفس نوع وإصدار مشروع ERP الفعلي قبل التشغيل في بيئة الإنتاج.

---

## 5. ملفات المكتبة

قد تحتوي حزمة المكتبة على الملفات التالية:

```text
Eptts.Client.dll
Eptts.Client.xml
Eptts.Client.pdb
Eptts.Client.deps.json
```

### `Eptts.Client.dll`

ملف المكتبة الأساسي الذي يجب إضافته إلى مشروع ERP كـ Reference.

### `Eptts.Client.xml`

يحتوي على XML Documentation الخاصة بالفئات والدوال والخصائص.

يستخدمه Visual Studio لعرض الشرح داخل `IntelliSense`.

يجب أن يكون بجوار ملف DLL، ويجب أن يكون الاسم متطابقًا:

```text
Eptts.Client.dll
Eptts.Client.xml
```

### `Eptts.Client.pdb`

يحتوي على معلومات Debug.

ليس مطلوبًا لتشغيل المكتبة، لكنه مفيد أثناء الاختبار وتحليل الأخطاء.

### `Eptts.Client.deps.json`

يحتوي على معلومات Dependencies الخاصة بالمكتبة.

لا تتم إضافته كـ Reference داخل المشروع. يمكن توزيعه مع ملفات المكتبة، لكن ملف الاستخدام الأساسي هو:

```text
Eptts.Client.dll
```

---

## 6. وضع المكتبة داخل مشروع ERP

أنشئ مجلدًا ثابتًا داخل مشروع ERP أو بجوار ملف Solution.

الشكل المقترح:

```text
ErpProject
├── Libraries
│   └── Eptts
│       ├── Eptts.Client.dll
│       ├── Eptts.Client.xml
│       ├── Eptts.Client.pdb
│       └── Eptts.Client.deps.json
│
├── ErpProject.csproj
└── ...
```

لا تضف Reference مباشرة من مجلد:

```text
bin\Debug
```

أو:

```text
bin\Release
```

الخاص بمشروع المكتبة الأصلي، لأن هذه المجلدات قد تُحذف عند تنفيذ `Clean`.

استخدم دائمًا مجلدًا ثابتًا مثل:

```text
Libraries\Eptts
```

---

## 7. إضافة المكتبة باستخدام Visual Studio

داخل Visual Studio:

```text
Solution Explorer
→ المشروع
→ Dependencies أو References
→ Add Reference
→ Browse
```

اختر:

```text
Libraries\Eptts\Eptts.Client.dll
```

بعد إضافة المرجع، يجب أن يظهر:

```text
Eptts.Client
```

داخل `Dependencies` أو `References` حسب نوع المشروع.

### التحقق من Copy Local

حدد Reference الخاص بالمكتبة، ثم افتح نافذة `Properties`.

تأكد من:

```text
Copy Local = True
```

في المشاريع الحديثة قد تظهر الخاصية داخل ملف المشروع باسم:

```xml
<Private>true</Private>
```

هذا يضمن نسخ DLL إلى مجلد تشغيل البرنامج.

---

## 8. إضافة Reference من ملف المشروع

إذا كان المشروع يستخدم ملف `.csproj` حديثًا، يمكن إضافة المرجع مباشرة:

```xml
<ItemGroup>
  <Reference Include="Eptts.Client">
    <HintPath>Libraries\Eptts\Eptts.Client.dll</HintPath>
    <Private>true</Private>
  </Reference>
</ItemGroup>
```

مثال مشروع WPF يستهدف `.NET 9`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
  </PropertyGroup>

  <ItemGroup>
    <Reference Include="Eptts.Client">
      <HintPath>Libraries\Eptts\Eptts.Client.dll</HintPath>
      <Private>true</Private>
    </Reference>
  </ItemGroup>

</Project>
```

يجب أن يكون `HintPath` صحيحًا بالنسبة إلى مكان ملف `.csproj`.

---

## 9. إضافة Dependencies

تعتمد المكتبة على `Newtonsoft.Json`.

يجب استخدام الإصدار المتوافق مع الإصدار المستخدم عند بناء المكتبة.

يمكن إضافة الحزمة من NuGet:

```text
Right-click على المشروع
→ Manage NuGet Packages
→ ابحث عن Newtonsoft.Json
→ Install
```

أو من Package Manager Console:

```powershell
Install-Package Newtonsoft.Json
```

إذا كان مشروع ERP يحتوي بالفعل على `Newtonsoft.Json`، فلا تضف Reference آخر قبل التأكد من عدم وجود تعارض في الإصدار.

لا تستخدم الطريقتين معًا:

```text
PackageReference إلى Newtonsoft.Json
مع
Reference يدوي إلى Newtonsoft.Json.dll
```

استخدم طريقة واحدة فقط.

---

## 10. التحقق من ملفات Output

نفذ:

```text
Build
→ Rebuild Solution
```

ثم افتح مجلد Output، مثل:

```text
bin\Debug
et9.0-windows
```

أو:

```text
bin\Debug
```

بحسب نوع المشروع.

يجب أن تجد:

```text
Eptts.Client.dll
Eptts.Client.xml
Newtonsoft.Json.dll
```

قد تجد أيضًا:

```text
Eptts.Client.pdb
Eptts.Client.deps.json
```

إذا لم يظهر `Eptts.Client.dll`، فتأكد من:

```text
Copy Local = True
```

أو:

```xml
<Private>true</Private>
```

---

## 11. التحقق من XML Documentation

لظهور شرح الدوال داخل Visual Studio، يجب أن يكون الملفان بجوار بعضهما:

```text
Eptts.Client.dll
Eptts.Client.xml
```

افتح ملف C# واكتب:

```csharp
using Eptts.Client.Configuration;
```

ثم:

```csharp
EpttsOptions options;
```

مرر مؤشر الفأرة فوق `EpttsOptions`. يجب أن يظهر شرح Class.

جرّب أيضًا:

```csharp
options.Validate();
```

يجب أن يظهر شرح قريب من:

```text
Validates all required EPTTS configuration values.
```

إذا لم تظهر التعليقات:

1. تأكد أن `Eptts.Client.xml` بجوار DLL التي يشير إليها المشروع.
2. تأكد أن اسم XML يطابق اسم DLL.
3. افتح ملف XML وابحث عن `EpttsOptions` و`Validate`.
4. تأكد أن XML وDLL نُسخا من Build نفسه.
5. احذف مجلدي `bin` و`obj`.
6. أغلق Visual Studio.
7. افتح Solution مرة أخرى.
8. نفذ `Rebuild Solution`.

---

## 12. Namespaces الأساسية

لإعداد الاتصال وإنشاء Client:

```csharp
using System;
using Eptts.Client.Clients;
using Eptts.Client.Configuration;
using Eptts.Client.Exceptions;
```

عند إنشاء Requests للعمليات:

```csharp
using Eptts.Client.Requests;
```

عند قراءة Responses:

```csharp
using Eptts.Client.Responses;
```

لا تحتاج إلى إضافة جميع Namespaces في كل ملف. أضف فقط ما يستخدمه الملف الحالي.

---

## 13. إعدادات الاتصال المطلوبة

قبل إنشاء `EpttsClient` يجب توفير القيم التالية:

- `BaseUrl`: عنوان بيئة EPTTS.
- `IntegratorKey`: المفتاح السري الخاص بالـ Integrator.
- `PharmacyGln`: GLN الصيدلية الحالية.
- `PharmacySgln`: SGLN الخاص بموقع الصيدلية.
- `Timeout`: مهلة انتظار HTTP Request الواحد.

المكتبة لا تقرأ هذه القيم تلقائيًا من ERP.

مشروع ERP مسؤول عن تحميل القيم من مصدر الإعدادات الخاص به ثم تمريرها إلى المكتبة.

---

## 14. مصدر إعدادات الاتصال

### أثناء الاختبار

يمكن استخدام قيم مباشرة مؤقتًا:

```csharp
string baseUrl =
    "https://masar-api.v2.daf-holding.com";

string integratorKey =
    "PUT-YOUR-INTEGRATOR-KEY-HERE";

string pharmacyGln =
    "6221388358239";

string pharmacySgln =
    "urn:epc:id:sgln:6221388.35823.0";
```

استبدل:

```text
PUT-YOUR-INTEGRATOR-KEY-HERE
```

بالمفتاح الصحيح الخاص ببيئة الاختبار.

لا ترفع المفتاح الحقيقي إلى GitHub، ولا تضعه داخل ملف Markdown أو Screenshot.

### داخل ERP الحقيقي

يجب أن تأتي القيم من مصدر إعدادات ERP.

مثال توضيحي لكائن يملكه ERP:

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

هذه Class ليست جزءًا من مكتبة `Eptts.Client`.

يجب على فريق ERP إنشاؤها أو استخدام نظام الإعدادات الموجود بالفعل.

مثال توضيحي لتحميل الإعدادات:

```csharp
ErpEpttsSettings settings =
    LoadEpttsSettingsForPharmacy(
        currentPharmacyId);
```

الدالة `LoadEpttsSettingsForPharmacy` والمتغير `currentPharmacyId` أمثلة من ERP وليسا جزءًا من المكتبة.

---

## 15. إنشاء `EpttsOptions`

بعد الحصول على القيم، أنشئ:

```csharp
EpttsOptions options =
    new EpttsOptions
    {
        BaseUrl =
            baseUrl,

        IntegratorKey =
            integratorKey,

        PharmacyGln =
            pharmacyGln,

        PharmacySgln =
            pharmacySgln,

        Timeout =
            TimeSpan.FromSeconds(60)
    };
```

### مثال باستخدام إعدادات ERP

```csharp
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
```

في هذا المثال، المتغير `settings` هو كائن حمّله ERP من مصدر إعداداته، وليس متغيرًا توفره المكتبة.

---

## 16. شرح خصائص `EpttsOptions`

### `BaseUrl`

عنوان بيئة EPTTS فقط:

```text
https://masar-api.v2.daf-holding.com
```

لا تضف مسار Endpoint:

```text
https://masar-api.v2.daf-holding.com/masar-service/api/v1/VerifyProduct
```

المكتبة تضيف مسار Endpoint المناسب عند استدعاء كل دالة.

### `IntegratorKey`

مفتاح سري توفره EPTTS أو الجهة المسؤولة عن البيئة.

لا تضع المفتاح في:

- GitHub.
- ملفات Markdown.
- Screenshots.
- Logs.
- رسائل الأخطاء.
- `RawRequest`.
- `RawResponse`.

في تطبيق الإنتاج يجب تحميل المفتاح من مصدر إعدادات محمي.

### `PharmacyGln`

معرف الصيدلية الحالية، ويتكون من 13 رقمًا.

مثال:

```text
6221388358239
```

يجب أن تكون هذه الصيدلية هي الجهة التي ينفذ ERP العملية باسمها.

لا تستخدم GLN الخاص بالفرع المرسل أو الفرع المستقبل مكان `PharmacyGln`.

### `PharmacySgln`

معرف موقع الصيدلية بصيغة EPC SGLN.

مثال:

```text
urn:epc:id:sgln:6221388.35823.0
```

يجب أن يخص نفس الصيدلية الموجودة في `PharmacyGln`.

### `Timeout`

يمثل أقصى مدة لانتظار HTTP Request واحد:

```csharp
Timeout =
    TimeSpan.FromSeconds(60);
```

لا يمثل المدة التي تحتاجها EPTTS لمعالجة العملية بعد قبول الرسالة.

قد تعيد العملية `HTTP 202` ثم تحتاج إلى متابعة النتيجة النهائية بصورة منفصلة.

---

## 17. التحقق من الإعدادات

الدالة:

```csharp
options.Validate();
```

عامة `public` ويمكن لمشروع ERP استدعاؤها.

تتحقق محليًا من:

- وجود `BaseUrl`.
- صحة تنسيق `BaseUrl`.
- وجود `IntegratorKey`.
- صحة `PharmacyGln`.
- صحة `PharmacySgln`.
- صلاحية `Timeout`.

لا ترسل هذه الدالة أي HTTP Request.

ولا تتحقق من:

- أن `IntegratorKey` فعال.
- أن الصيدلية مخولة باستخدام البيئة.
- أن خادم EPTTS متاح.
- أن الاتصال بالإنترنت يعمل.

### مثال

```csharp
try
{
    options.Validate();

    /*
     * الإعدادات صحيحة من ناحية التنسيق المحلي.
     */
}
catch (EpttsConfigurationException exception)
{
    string errorMessage =
        exception.Message;

    /*
     * اعرض الخطأ أو سجله دون تسجيل IntegratorKey.
     */
}
```

---

## 18. إنشاء `EpttsClient`

بعد إنشاء الإعدادات والتحقق منها:

```csharp
using (EpttsClient client =
    new EpttsClient(options))
{
    /*
     * EpttsClient is ready.
     */
}
```

إنشاء `EpttsClient` لا يرسل HTTP Request.

يبدأ الاتصال الفعلي عند استدعاء دالة مثل:

```csharp
await client.VerifyProductAsync(...);
```

أو:

```csharp
await client.GetMessageStatusAsync(...);
```

---

## 19. لماذا نستخدم `using`؟

تنفذ `EpttsClient` الواجهة:

```csharp
IDisposable
```

لذلك يجب التخلص منها بعد انتهاء الاستخدام:

```csharp
using (EpttsClient client =
    new EpttsClient(options))
{
    /*
     * استخدم Client هنا.
     */
}
```

بعد الخروج من `using` لا تستخدم `client` مرة أخرى.

---

## 20. أول اختبار محلي

أنشئ Class مؤقتة داخل مشروع الاختبار:

```csharp
using System;
using Eptts.Client.Clients;
using Eptts.Client.Configuration;
using Eptts.Client.Exceptions;

namespace ErpIntegrationTest
{
    public static class EpttsConnectionTest
    {
        public static string TestClientCreation(
            string integratorKey)
        {
            try
            {
                EpttsOptions options =
                    new EpttsOptions
                    {
                        BaseUrl =
                            "https://masar-api.v2.daf-holding.com",

                        IntegratorKey =
                            integratorKey,

                        PharmacyGln =
                            "6221388358239",

                        PharmacySgln =
                            "urn:epc:id:sgln:6221388.35823.0",

                        Timeout =
                            TimeSpan.FromSeconds(60)
                    };

                options.Validate();

                using (EpttsClient client =
                    new EpttsClient(options))
                {
                    return
                        "EpttsClient was created successfully." +
                        Environment.NewLine +
                        "No HTTP request was sent.";
                }
            }
            catch (EpttsConfigurationException exception)
            {
                return
                    "Configuration error: " +
                    exception.Message;
            }
            catch (Exception exception)
            {
                return
                    "Unexpected error: " +
                    exception.Message;
            }
        }
    }
}
```

استدعاء هذه الدالة يختبر:

- أن Reference أضيف بصورة صحيحة.
- أن DLL تعمل داخل المشروع.
- أن `EpttsOptions` متاحة.
- أن `Validate()` تعمل.
- أن `EpttsClient` يمكن إنشاؤها.
- أن Dependencies المطلوبة موجودة.

لا يختبر الاتصال الفعلي بالخادم.

أول اختبار اتصال فعلي سيكون من خلال `VerifyProduct`، وسيتم شرحه في جزء لاحق.

---

## 21. التحقق من أول Build

نفذ:

```text
Build
→ Clean Solution
→ Rebuild Solution
```

المطلوب:

```text
Build succeeded
```

ثم تحقق من مجلد Output:

```text
Eptts.Client.dll
Eptts.Client.xml
Newtonsoft.Json.dll
```

إذا كان Build ناجحًا ويمكن إنشاء `EpttsClient`، يكون تثبيت المكتبة قد اكتمل.

---

## 22. أخطاء التثبيت الشائعة

### Namespace أو نوع `Eptts.Client` غير معروف

مثال الخطأ:

```text
The type or namespace name 'Eptts' could not be found
```

راجع:

- إضافة Reference إلى `Eptts.Client.dll`.
- صحة `HintPath` داخل `.csproj`.
- أن Namespace المستخدمة تبدأ بـ `Eptts.Client`.
- أن DLL موجودة في المسار المحدد.

### عدم ظهور XML Documentation

راجع وجود:

```text
Eptts.Client.dll
Eptts.Client.xml
```

في المجلد نفسه وبالاسم نفسه.

احذف `bin` و`obj`، ثم أعد فتح Visual Studio ونفذ `Rebuild`.

### خطأ تحميل `Newtonsoft.Json`

راجع أن الحزمة مثبتة داخل مشروع ERP.

لا تضف إصدارين مختلفين بالطريقتين اليدوية وNuGet في الوقت نفسه.

### ظهور `EpttsConfigurationException`

يعني وجود إعداد محلي غير صحيح، مثل:

- `BaseUrl` فارغ أو غير صحيح.
- `IntegratorKey` فارغ.
- `PharmacyGln` غير صحيح.
- `PharmacySgln` غير صحيح.
- `Timeout` غير صالح.

اقرأ:

```csharp
exception.Message
```

لتحديد القيمة غير الصحيحة.

### إنشاء Client نجح لكن الاتصال قد يفشل لاحقًا

نجاح:

```csharp
new EpttsClient(options)
```

لا يعني أن المفتاح مقبول أو أن الخادم متاح.

إنشاء Client يتحقق من الإعدادات المحلية فقط.

يتم التحقق الفعلي عند استدعاء أول Endpoint.

---

## 23. قائمة إتمام الجزء الأول

قبل الانتقال إلى الجزء التالي، تأكد من:

```text
[ ] تم استلام Eptts.Client.dll
[ ] تم وضع DLL داخل مجلد ثابت
[ ] تم وضع Eptts.Client.xml بجوار DLL
[ ] تم إضافة Reference إلى المشروع
[ ] تم ضبط Copy Local = True
[ ] تم تثبيت Newtonsoft.Json
[ ] تم تنفيذ Rebuild Solution بنجاح
[ ] ظهرت Eptts.Client.dll داخل Output
[ ] ظهرت Eptts.Client.xml داخل Output
[ ] ظهرت XML Documentation في IntelliSense
[ ] تم توفير BaseUrl
[ ] تم توفير IntegratorKey
[ ] تم توفير PharmacyGln
[ ] تم توفير PharmacySgln
[ ] تم تحديد Timeout
[ ] تم إنشاء EpttsOptions
[ ] تم تنفيذ options.Validate()
[ ] تم إنشاء EpttsClient
[ ] تم التأكد أن إنشاء Client لا يرسل HTTP Request
[ ] لم يتم تسجيل IntegratorKey
```

---

## 24. خلاصة الجزء الأول

التسلسل الصحيح لبدء استخدام المكتبة هو:

```text
استلام ملفات المكتبة
→ وضعها داخل مجلد ثابت
→ إضافة DLL Reference
→ تثبيت Newtonsoft.Json
→ إنشاء EpttsOptions
→ تنفيذ Validate
→ إنشاء EpttsClient
→ الانتقال إلى أول Endpoint
```

أقصر كود صحيح للبداية:

```csharp
using System;
using Eptts.Client.Clients;
using Eptts.Client.Configuration;

EpttsOptions options =
    new EpttsOptions
    {
        BaseUrl =
            "https://masar-api.v2.daf-holding.com",

        IntegratorKey =
            integratorKeyFromSecureSettings,

        PharmacyGln =
            "6221388358239",

        PharmacySgln =
            "urn:epc:id:sgln:6221388.35823.0",

        Timeout =
            TimeSpan.FromSeconds(60)
    };

options.Validate();

using (EpttsClient client =
    new EpttsClient(options))
{
    /*
     * Client is ready.
     * No HTTP request has been sent yet.
     */
}
```

المتغير:

```csharp
integratorKeyFromSecureSettings
```

يمثل قيمة حمّلها ERP من مصدر إعدادات محمي. لا توفره المكتبة ولا يجب كتابته داخل Source Code.
