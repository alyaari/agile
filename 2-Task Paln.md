

### 🧱 النموذج الموحد للمهام الفرعية (Standard Sub-tasks Template)
*(سيتم تطبيق هذا النموذج على جميع المهام الخلفية والأمامية لضمان التوحيد)*

| نوع المهمة (Task Type) | المهام الفرعية النمطية (Standard Sub-tasks) |
| :--- | :--- |
| **إنشاء API (Backend)** | 1. كتابة `Model` و `DTO`.<br>2. كتابة استعلام SQL/Stored Procedure.<br>3. كتابة `Service` Layer (منطق الأعمال).<br>4. كتابة `Controller` مع التوثيق (Swagger/OpenAPI).<br>5. كتابة `Unit Test` للتأكد من صحة المنطق. |
| **واجهة المستخدم (Frontend)** | 1. إنشاء المكون (Component) أو تعديل الشاشة.<br>2. ربط البيانات (Data Binding) مع APIs.<br>3. تطبيق منطق الإظهار/الإخفاء و `Validation`.<br>4. كتابة `Integration Test` (أو اختبار يدوي للسيناريوهات). |

---

### 📦 الحزمة التنفيذية رقم 1: إعدادات الفاتورة والبيئة (Master Setup)
*(تغطي القصص: #1 إلى #6 - الفرع، الترقيم، الدفع، الصندوق)*

**Feature Task A: تطبيق نظام الترقيم التراكمي (Accumulative Numbering)**

- **Task A.1 (Database):** إنشاء جدول `InvoiceSequence` وهندسة Stored Procedure للترقيم الذري.
  - *Sub-task 1:* كتابة `CREATE TABLE InvoiceSequence (BranchId INT, CurrentSeq INT)`.
  - *Sub-task 2:* كتابة `Stored Procedure (USP_GetNextInvoiceNumber)` باستخدام `WITH (UPDLOCK, ROWLOCK)` لضمان عدم تكرار الرقم في الضغط العالي (Concurrency).
  - *Sub-task 3:* كتابة دالة لتوليد `GUID` كمعرف فريد عام للفاتورة.
- **Task A.2 (Backend):** إنشاء API `api/invoices/next-number`.
  - *Sub-task 1:* استدعاء الـ SP لإرجاع الرقم الجديد.
  - *Sub-task 2:* إرجاع الرقم مع `InvoiceUID` الجاهز للاستخدام.
- **Task A.3 (Frontend):** ربط الترقيم بشاشة الإضافة.
  - *Sub-task 1:* عند فتح شاشة الإضافة (`OnInit`)، جلب الرقم وعرضه في حقل للقراءة فقط (`Disabled`).

---

**Feature Task B: منطق اختيار الفرع وطريقة الدفع (Branch & Payment Logic)**

- **Task B.1 (Backend):** APIs لجلب البيانات الديناميكية.
  - *Sub-task 1:* API `api/branches/user-branches` - يعيد فروع المستخدم بناءً على `UserId`.
  - *Sub-task 2:* API `api/payment-methods/available` - يعيد طرق الدفع المتاحة للمستخدم، مع تطبيق منطق (إذا كان الافتراضي غير مسموح، يرجع الأول المسموح).
- **Task B.2 (Frontend):** منطق التعيين والإخفاء/الإظهار.
  - *Sub-task 1:* إذا كان عدد الفروع `== 1`، قم بتعيين الحقل مباشرة وإخفاء الـ Dropdown.
  - *Sub-task 2:* ربط `(change)` على حقل طريقة الدفع؛ إذا تغيرت إلى "نقد" أظهر الصندوق، وإذا "آجل" أخفِ الصندوق وأظهر العميل.
  - *Sub-task 3:* عند اختيار الصندوق، جلب اسمه وعرضه في حقل آخر `Readonly`.

---

### 📦 الحزمة التنفيذية رقم 2: التحقق من العملاء والمخازن (Customers & Stores)
*(تغطي القصص: #7 إلى #14 - العميل، التوقيف، حد الدين، المخازن الموقفة)*

**Feature Task C: التحقق من العميل (التوقيف وحد الدين)**

- **Task C.1 (Backend - Logic):** كتابة `Service` للتحقق من صحة العميل (`ValidateCustomerService`).
  - *Sub-task 1:* كتابة دالة `CheckIfStopped(CustomerId)` ترجع `true/false`.
  - *Sub-task 2:* كتابة دالة `CheckCreditLimit(CustomerId, CurrencyId, InvoiceTotal)` تحسب (`إجمالي الأرصدة - المدفوعات`) وتقارن بـ `CreditLimit`.
  - *Sub-task 3:* دمج النتيجتين في API واحد `api/customers/validate`.
- **Task C.2 (Backend - Override Logic):** تطبيق صلاحية تجاوز حد الدين.
  - *Sub-task 1:* قراءة `AllowDebtOverride` و `OverridePercentage` من `UserClaims`.
  - *Sub-task 2:* إذا كان التجاوز أقل من النسبة المئوية المسموحة، يسمح بالمرور مع `Warning`، وإلا يمنع.
- **Task C.3 (Frontend):** تطبيق التحقق الفوري (`OnBlur` و `BeforeSave`).
  - *Sub-task 1:* عند الخروج من حقل العميل، استدعاء API وعرض رسائل الخطأ (مثل: "العميل موقف" أو "تجاوز الحد").
  - *Sub-task 2:* في حالة التجاوز المسموح به، عرض `Confirmation Dialog` (هل تريد المتابعة؟).

---

**Feature Task D: إدارة المخازن (Store Management)**

- **Task D.1 (Backend):** API `api/stores/available`.
  - *Sub-task 1:* جلب المخازن المرتبطة بـ `BranchId` المحدد.
  - *Sub-task 2:* فلترة النتائج بحسب صلاحيات المستخدم (`UserStores`).
  - *Sub-task 3:* إرجاع حقل `IsActive` مع كل مخزن لتحديد إذا كان موقفاً.
- **Task D.2 (Frontend):** التعامل مع المخازن الموقفة.
  - *Sub-task 1:* تلوين المخازن الموقفة باللون الرمادي في القائمة.
  - *Sub-task 2:* عند اختيار مخزن موقف، عرض `MessageBox` ("تم إيقاف البيع") وتعطيل شبكة الأصناف (Grid).

---

### 📦 الحزمة التنفيذية رقم 3: قلب النظام (إدخال الأصناف - Details Grid)
*(تغطي القصص: #15 إلى #21 - الأصناف، الباركود، F9، الوحدات، الكميات)*
*(هذه أكبر حزمة وأكثرها تعقيداً)*

**Feature Task E: محرك البحث عن الصنف (Item Search Engine)**

- **Task E.1 (Backend - API):** `api/items/search`.
  - *Sub-task 1:* البحث المباشر: التحقق من جدول `Barcodes` (إذا تطابق، استخراج `ItemId`).
  - *Sub-task 2:* المطابقة التامة: البحث في `ItemCode`.
  - *Sub-task 3:* التحقق من الوجود في المخزن: `JOIN` مع `ItemBalances` حيث `StoreId == SelectedStoreId`، إذا لم يجده، إرجاع رسالة محددة ("الصنف غير موجود في هذا المخزن").
  - *Sub-task 4:* البحث الجزئي (Partial/Contains): إذا لم يجد المطابقة، استخدام `LIKE '%input%'` لإرجاع قائمة.
- **Task E.2 (Frontend - Search UI):** ربط حقل الإدخال.
  - *Sub-task 1:* ربط `(keydown.enter)` لاستدعاء API.
  - *Sub-task 2:* إذا رجع API بيانات (اسم، سعر، وحدة)، تعبئة الصف والانتقال (`Focus`) إلى حقل الكمية.
  - *Sub-task 3:* إذا رجع API قائمة (بحث جزئي)، فتح `Modal` يعرض النتائج مع إمكانية الضغط المزدوج (Double-click) للاختيار.

---

**Feature Task F: قائمة F9 الذكية (Smart F9 Lookup)**

- **Task F.1 (Backend):** API `api/items/store-items`.
  - *Sub-task 1:* قراءة متغيرات النظام (`ShowZeroQuantity`, `ShowOnlyPricedItems`) وتطبيقها كلها في استعلام واحد.
  - *Sub-task 2:* دعم معايير البحث (StartsWith, EndsWith, Contains) استناداً إلى `User.DefaultSearchMode`.
- **Task F.2 (Frontend):** ربط زر F9.
  - *Sub-task 1:* إذا كان الحقل فارغاً، عرض كل الأصناف (وفق الفلاتر).
  - *Sub-task 2:* إذا كان هناك نص، تطبيق معيار البحث وإظهار النتائج في `Popup`.

---

**Feature Task G: إدارة الوحدات والكميات (Units & Quantities)**

- **Task G.1 (Backend):** جلب وحدات الصنف مع معامل التحويل.
  - *Sub-task 1:* عند اختيار الصنف، جلب `DefaultSellUnitId` وقائمة `ItemUnits` مع `ConversionFactor`.
- **Task G.2 (Frontend - Unit Change Logic):** 
  - *Sub-task 1:* تعبئة الوحدة تلقائياً.
  - *Sub-task 2:* ربط `(change)` على حقل الوحدة. عند التغيير، استدعاء API لجلب الكمية المتوفرة (`AvailableQuantity`) بالوحدة الجديدة.
  - *Sub-task 3:* إذا كانت الكمية المدخلة حالياً > المتوفرة الجديدة، مسح الكمية وعرض تحذير "الكمية غير متوفرة بهذه الوحدة".
- **Task G.3 (Frontend - Quantity Validation):** 
  - *Sub-task 1:* ربط `(blur)` على حقل الكمية. إذا كانت القيمة > المتوفرة، منع الحفظ وتعطيل زر الحفظ.

---

### 📦 الحزمة التنفيذية رقم 4: التسعير والضرائب والإجماليات (Pricing & Calculations)
*(تغطي القصص: #22 إلى #27 - السعر، التكلفة، الخصم، الضريبة، الإجمالي)*

**Feature Task H: محرك التسعير والتكاليف (Pricing Engine)**

- **Task H.1 (Backend):** دالة `GetItemPrice`.
  - *Sub-task 1:* جلب السعر من جدول `Pricing` (مرتبط بالصنف، العميل، أو الفئة السعرية).
  - *Sub-task 2:* جلب `ItemCost` من جدول التكاليف.
- **Task H.2 (Frontend - Permissions & Cost):** 
  - *Sub-task 1:* إذا كان المستخدم لا يملك صلاحية `CanEditPrice`، جعل حقل السعر `Readonly`.
  - *Sub-task 2:* عند محاولة تغيير السعر يدوياً (`OnChange`)، التحقق الفوري: إذا كان `NewPrice < ItemCost`، عرض رسالة خطأ ورفض التحديث.

**Feature Task I: حساب الضريبة والإجمالي (Tax & Totals)**

- **Task I.1 (Backend):** تمرير `TaxPercentage` مع بيانات الصنف.
- **Task I.2 (Frontend - Formula Engine):** تطبيق المعادلة الرئيسية في الـ Grid.
  - *Sub-task 1:* `ItemTotal = (الكمية * (السعر + الضريبة - الخصم))`.
  - *Sub-task 2:* ربط هذه المعادلة بـ `OnChange` لأي حقل (كمية، سعر، خصم) لتحديث الإجمالي ديناميكياً.
- **Task I.3 (Frontend - Footer):** حساب إجمالي الفاتورة.
  - *Sub-task 1:* جمع كل `ItemTotal` وعرضه في `Footer` أسفل الشبكة.

---

### 📦 الحزمة التنفيذية رقم 5: العروض الترويجية وصلاحيات CRUD (Advanced)
*(تغطي القصص: #28 إلى #33 - العروض، الصلاحيات، البحث، المتغيرات)*

**Feature Task J: محرك العروض الترويجية (Promotion Engine)**

- **Task J.1 (Backend - Promotion Checker):** 
  - *Sub-task 1:* عند إضافة صف، فحص العروض النشطة (Item Level). إذا تحقق الشرط (مثلاً الكمية >= 2)، حساب `FreeQuantity` وإرجاعها.
  - *Sub-task 2:* عند حساب الإجمالي الكلي، فحص العروض (Invoice Level). إذا تجاوز `Total` قيمة `Threshold`، حساب خصم إضافي وتطبيقه على صافي الفاتورة.
- **Task J.2 (Frontend):** 
  - *Sub-task 1:* عرض `FreeQuantity` في حقل للقراءة فقط عند جلبها.
  - *Sub-task 2:* عند تطبيق خصم الفاتورة، تحديث `Grand Total` في منطقة التذييل.

---

**Feature Task K: الصلاحيات وشاشة البحث (Permissions & Lookup)**

- **Task K.1 (Backend - Authorization):** 
  - *Sub-task 1:* إضافة `[Authorize(Policy = "AddInvoice")]` على `POST`، و `Edit`، و `Delete`، و `View`.
- **Task K.2 (Frontend - UI Control):** 
  - *Sub-task 1:* في شاشة الإضافة، إذا لم تكن صلاحية "إضافة" موجودة، تعطيل زر "جديد".
  - *Sub-task 2:* في شاشة البحث، إذا لم تكن صلاحية "تعديل" موجودة، تعطيل زر التعديل أو جعل الصف للقراءة فقط.
- **Task K.3 (Frontend - Search Screen):** بناء شاشة بحث (`Invoice Search`).
  - *Sub-task 1:* إنشاء `Grid` مع فلاتر (رقم الفاتورة، العميل، الفرع، التاريخ).
  - *Sub-task 2:* ربط الـ Grid بـ API البحث (`api/invoices/search`).
  - *Sub-task 3:* عند الضغط المزدوج على صف، فتح شاشة الفاتورة في وضع (عرض) أو (تعديل) حسب الصلاحية.

---

**Feature Task L: المتغيرات العامة (Global Settings)**

- **Task L.1 (Backend):** API `api/settings/invoice`.
  - *Sub-task 1:* إرجاع جميع المتغيرات المؤثرة (`IsCustomerNameRequired`, `ShowZeroItems`, `ShowOnlyPriced`).
- **Task L.2 (Frontend):** 
  - *Sub-task 1:* عند تحميل شاشة الفاتورة (`OnInit`)، جلب هذه المتغيرات وتخزينها في `AppState` أو `Context`.
  - *Sub-task 2:* تطبيق `IsCustomerNameRequired` على حقل اسم العميل النقدي (جعله `Required` أو لا).
  - *Sub-task 3:* تمرير `ShowZeroItems` و `ShowOnlyPriced` كـ Parameters عند استدعاء API جلب أصناف F9.

---

### ⏱️ تقدير الجهد (Estimation) للمطورين

| الحزمة (Package) | عدد المهام الرئيسية | إجمالي المهام الفرعية | الوقت التقديري (ساعات) |
| :--- | :--- | :--- | :--- |
| الحزمة 1 (Master Setup) | 2 Tasks | 7 Sub-tasks | 8 - 10 ساعات |
| الحزمة 2 (Customers & Stores) | 4 Tasks | 12 Sub-tasks | 16 - 20 ساعة |
| **الحزمة 3 (Details Grid - الأكبر)** | **7 Tasks** | **22 Sub-tasks** | **30 - 40 ساعة** |
| الحزمة 4 (Pricing & Calculations) | 3 Tasks | 8 Sub-tasks | 10 - 12 ساعة |
| الحزمة 5 (Promotions & Permissions) | 4 Tasks | 12 Sub-tasks | 12 - 16 ساعة |
| **المجموع الكلي** | **20 مهمة رئيسية** | **61 مهمة فرعية تفصيلية** | **~80 - 100 ساعة (تقريباً 5 سباقات)** |

---

### ✅ تعليمات التوزيع على فريق Scrum

1.  **قاعدة بيانات (DBA):** يركز على **Task A.1** (الترقيم) وجميع مهام `Sub-task` المتعلقة بكتابة `Stored Procedures` و `Indexes`.
2.  **مطور Backend (الباك إند):** ينفذ **جميع مهام APIs** (التحقق من العملاء، محرك البحث، محرك العروض، المتغيرات) بالتوازي مع قاعدة البيانات.
3.  **مطور Frontend (الفرونت إند):** يبدأ بـ **Task B.2** و **D.2** (المنطق البصري)، ثم ينتقل فوراً إلى **Task E.2** (شبكة الأصناف) فور جاهزية APIs الخاصة بالبحث.

---

**ملاحظة خبير:** تم تفكيك "البحث الجزئي" و "الباركود" في Sub-task واحدة (E.1 - 4) لأن منطقهم الخلفي متشابه (بحث عن نص)، ويتم الفصل بينهم في طبقة الخدمة (Service Layer) فقط. كما تم دمج "منع السعر أقل من التكلفة" مع "صلاحية تعديل السعر" في مهمة واحدة (H.2) لأن كلاهما يتعامل مع `OnChange` الخاص بحقل السعر.

هذا التفكيك **لا يترك أي مجال للتخمين** للمطورين. كل Sub-task محددة بـ "ماذا سأفعل" و "أين سأكتب الكود". 
 