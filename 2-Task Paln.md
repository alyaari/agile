

## 🧱 النموذج الموحد للمهام الفرعية (Standard Sub-tasks Template)

*(سيتم تطبيق هذا النموذج على جميع المهام الخلفية والأمامية لضمان التوحيد)*

| نوع المهمة (Task Type) | المهام الفرعية النمطية (Standard Sub-tasks) |
| :--- | :--- |
| **إنشاء API (Backend)** | 1. تصميم جدول قاعدة البيانات (الحقول، الفالديشن، الاندكسات).<br>2. كتابة `Model` و `DTO` مع الوصف.<br>3. كتابة استعلام SQL/Stored Procedure.<br>4. كتابة `Service` Layer (منطق الأعمال).<br>5. كتابة `Controller` مع التوثيق (Swagger/OpenAPI).<br>6. كتابة `Unit Test` للتأكد من صحة المنطق. |
| **واجهة المستخدم (Frontend)** | 1. إنشاء المكون (Component) أو تعديل الشاشة.<br>2. ربط البيانات (Data Binding) مع APIs.<br>3. تطبيق منطق الإظهار/الإخفاء و `Validation`.<br>4. كتابة `Integration Test` (أو اختبار يدوي للسيناريوهات). |

---

## 📦 الحزمة التنفيذية رقم 1: إعدادات الفاتورة والبيئة (Master Setup)
*(تغطي القصص: #1 إلى #6 - الفرع، الترقيم، الدفع، الصندوق)*

---

### **Feature Task A: تطبيق نظام الترقيم التراكمي (Accumulative Numbering)**

**الوصف:**
بناء نظام ترقيم تراكمي لكل فرع على حدة، بحيث تبدأ الفواتير في كل فرع من الرقم 1. يجب ضمان عدم تكرار الرقم في حالات الدخول المتزامن (Concurrency) باستخدام Row Lock. كما يجب توليد معرف فريد عام (`InvoiceUID`) لكل فاتورة.

---

#### **Task A.1 (Database):** إنشاء جدول `InvoiceSequence` وهندسة Stored Procedure للترقيم الذري.

**🔹 Sub-task 1: تصميم جدول `InvoiceSequence`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `BranchId` | `INT` | `PRIMARY KEY, NOT NULL, FK -> Branches(BranchId)` | معرف الفرع |
| `CurrentSeq` | `INT` | `NOT NULL, DEFAULT(0)` | آخر رقم تسلسلي مستخدم |
| `RowVersion` | `ROWVERSION` | — | للتحكم بالتزامن المتفائل |

**الفالديشن:**
- `CurrentSeq` يجب أن يكون `>= 0`.
- لا يُسمح بحذف سجل فرع موجود في جدول الفواتير.

**الاندكسات:**
- `PK_InvoiceSequence` على `BranchId`.
- `IX_InvoiceSequence_BranchId` (Clustered/NonClustered حسب محرك قاعدة البيانات).

**🔹 Sub-task 2: تصميم جدول `SalesInvoices`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `InvoiceId` | `BIGINT` | `PRIMARY KEY, IDENTITY` | المعرف الداخلي للفاتورة |
| `InvoiceUID` | `UNIQUEIDENTIFIER` | `NOT NULL, DEFAULT NEWID(), UNIQUE` | المعرف الفريد العام |
| `BranchId` | `INT` | `NOT NULL, FK -> Branches(BranchId)` | رقم الفرع |
| `InvoiceNumber` | `INT` | `NOT NULL` | الرقم التراكمي داخل الفرع |
| `PaymentMethodId` | `INT` | `NOT NULL, FK -> PaymentMethods(PaymentMethodId)` | طريقة الدفع |
| `CashBoxId` | `INT` | `NULL, FK -> CashBoxes(CashBoxId)` | رقم الصندوق (نقدًا) |
| `CustomerId` | `INT` | `NULL, FK -> Customers(CustomerId)` | رقم العميل (آجل) |
| `CustomerName` | `NVARCHAR(200)` | `NULL` | اسم العميل |
| `StoreId` | `INT` | `NOT NULL, FK -> Stores(StoreId)` | رقم المخزن |
| `CurrencyId` | `INT` | `NOT NULL, FK -> Currencies(CurrencyId)` | عملة الفاتورة |
| `InvoiceDate` | `DATETIME` | `NOT NULL, DEFAULT GETDATE()` | تاريخ الفاتورة |
| `TotalAmount` | `DECIMAL(18,2)` | `NOT NULL, DEFAULT(0)` | إجمالي الفاتورة |
| `DiscountAmount` | `DECIMAL(18,2)` | `NOT NULL, DEFAULT(0)` | قيمة الخصم |
| `NetAmount` | `DECIMAL(18,2)` | `NOT NULL, DEFAULT(0)` | الصافي |
| `IsDeleted` | `BIT` | `NOT NULL, DEFAULT(0)` | علامة الحذف المنطقي |
| `CreatedBy` | `INT` | `NOT NULL` | المستخدم المنشئ |
| `CreatedAt` | `DATETIME` | `NOT NULL, DEFAULT GETDATE()` | تاريخ الإنشاء |

**الفالديشن:**
- `InvoiceNumber` فريد داخل نفس `BranchId`.
- `CashBoxId` مطلوب عندما تكون `PaymentMethodId = Cash`.
- `CustomerId` مطلوب عندما تكون `PaymentMethodId = Credit`.
- `TotalAmount >= 0`, `NetAmount >= 0`.

**الاندكسات:**
- `PK_SalesInvoices` على `InvoiceId`.
- `UQ_SalesInvoices_Branch_InvoiceNumber` على `(BranchId, InvoiceNumber)`.
- `IX_SalesInvoices_InvoiceUID` على `InvoiceUID`.
- `IX_SalesInvoices_BranchId` على `BranchId`.
- `IX_SalesInvoices_InvoiceDate` على `InvoiceDate`.

**🔹 Sub-task 3: كتابة `Stored Procedure (USP_GetNextInvoiceNumber)`**

```sql
CREATE PROCEDURE USP_GetNextInvoiceNumber
    @BranchId INT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;
    
    DECLARE @NextNumber INT;
    
    SELECT @NextNumber = CurrentSeq + 1
    FROM InvoiceSequence WITH (UPDLOCK, ROWLOCK)
    WHERE BranchId = @BranchId;
    
    IF @NextNumber IS NULL
    BEGIN
        SET @NextNumber = 1;
        INSERT INTO InvoiceSequence (BranchId, CurrentSeq)
        VALUES (@BranchId, @NextNumber);
    END
    ELSE
    BEGIN
        UPDATE InvoiceSequence
        SET CurrentSeq = @NextNumber
        WHERE BranchId = @BranchId;
    END
    
    COMMIT TRANSACTION;
    
    SELECT @NextNumber AS NextInvoiceNumber, NEWID() AS InvoiceUID;
END
```

---

#### **Task A.2 (Backend):** إنشاء API `api/invoices/next-number`.

**🔹 Sub-task 1: كتابة `Model` و `DTO`**

**Model: `InvoiceSequence`**
```csharp
public class InvoiceSequence
{
    public int BranchId { get; set; }
    public int CurrentSeq { get; set; }
    public byte[] RowVersion { get; set; }
}
```

**DTO: `NextInvoiceNumberDto`**
```csharp
public class NextInvoiceNumberDto
{
    public int NextInvoiceNumber { get; set; }
    public Guid InvoiceUID { get; set; }
}
```

**🔹 Sub-task 2:** استدعاء الـ SP وإرجاع الرقم الجديد.

**🔹 Sub-task 3:** إرجاع الرقم مع `InvoiceUID` الجاهز للاستخدام.

---

#### **Task A.3 (Frontend):** ربط الترقيم بشاشة الإضافة.

**🔹 Sub-task 1:** عند فتح شاشة الإضافة (`OnInit`)، جلب الرقم وعرضه في حقل للقراءة فقط (`Disabled`).

---

### **Feature Task B: منطق اختيار الفرع وطريقة الدفع (Branch & Payment Logic)**

**الوصف:**
بناء APIs ومنطق واجهة المستخدم لاختيار الفرع وطريقة الدفع. إذا كان المستخدم له صلاحية على فرع واحد فقط يُعبأ تلقائيًا. طريقة الدفع تُعبأ بالقيمة الافتراضية للمستخدم مع التحقق من عدم تعارضها مع صلاحياته.

---

#### **Task B.1 (Backend):** APIs لجلب البيانات الديناميكية.

**🔹 Sub-task 1: API `api/branches/user-branches`**

**الجدول المرتبط: `UserBranches`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `UserBranchId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `UserId` | `INT` | `NOT NULL, FK -> Users(UserId)` | المستخدم |
| `BranchId` | `INT` | `NOT NULL, FK -> Branches(BranchId)` | الفرع |

**الاندكس:** `IX_UserBranches_UserId` على `UserId`.

**Model:**
```csharp
public class UserBranch
{
    public int UserBranchId { get; set; }
    public int UserId { get; set; }
    public int BranchId { get; set; }
    public Branch Branch { get; set; }
}
```

**DTO:**
```csharp
public class UserBranchDto
{
    public int BranchId { get; set; }
    public string BranchName { get; set; }
}
```

**🔹 Sub-task 2: API `api/payment-methods/available`**

**الجدول المرتبط: `UserPaymentMethods`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `UserPaymentMethodId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `UserId` | `INT` | `NOT NULL, FK -> Users(UserId)` | المستخدم |
| `PaymentMethodId` | `INT` | `NOT NULL, FK -> PaymentMethods(PaymentMethodId)` | طريقة الدفع |

**الجدول المرتبط: `Users`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `UserId` | `INT` | `PRIMARY KEY, IDENTITY` | معرف المستخدم |
| `DefaultPaymentMethodId` | `INT` | `NULL, FK -> PaymentMethods(PaymentMethodId)` | طريقة الدفع الافتراضية |

**Model:**
```csharp
public class UserPaymentMethod
{
    public int UserPaymentMethodId { get; set; }
    public int UserId { get; set; }
    public int PaymentMethodId { get; set; }
    public PaymentMethod PaymentMethod { get; set; }
}
```

**DTO:**
```csharp
public class AvailablePaymentMethodDto
{
    public int PaymentMethodId { get; set; }
    public string PaymentMethodName { get; set; }
    public bool IsDefault { get; set; }
    public bool IsAllowed { get; set; }
}
```

**منطق API:**
- إرجاع جميع طرق الدفع مع تحديد أيها مسموح للمستخدم.
- إذا كانت `DefaultPaymentMethodId` غير مسموحة، يُرجع أول طريقة مسموحة كبديل.

---

#### **Task B.2 (Frontend):** منطق التعيين والإخفاء/الإظهار.

**🔹 Sub-task 1:** إذا كان عدد الفروع `== 1`، قم بتعيين الحقل مباشرة وإخفاء الـ Dropdown.

**🔹 Sub-task 2:** ربط `(change)` على حقل طريقة الدفع؛ إذا تغيرت إلى "نقد" أظهر الصندوق، وإذا "آجل" أخفِ الصندوق وأظهر العميل.

**🔹 Sub-task 3:** عند اختيار الصندوق، جلب اسمه وعرضه في حقل آخر `Readonly`.

---

## 📦 الحزمة التنفيذية رقم 2: التحقق من العملاء والمخازن (Customers & Stores)
*(تغطي القصص: #7 إلى #14 - العميل، التوقيف، حد الدين، المخازن الموقفة)*

---

### **Feature Task C: التحقق من العميل (التوقيف وحد الدين)**

**الوصف:**
بناء خدمة للتحقق من صحة العميل عند البيع الآجل، تشمل التحقق من حالة التوقيف وحد الدين مع دعم صلاحية التجاوز.

---

#### **Task C.1 (Backend - Logic):** كتابة `Service` للتحقق من صحة العميل (`ValidateCustomerService`).

**🔹 Sub-task 1: تصميم جدول `Customers`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `CustomerId` | `INT` | `PRIMARY KEY, IDENTITY` | معرف العميل |
| `CustomerCode` | `NVARCHAR(50)` | `NOT NULL, UNIQUE` | كود العميل |
| `CustomerName` | `NVARCHAR(200)` | `NOT NULL` | اسم العميل |
| `IsStopped` | `BIT` | `NOT NULL, DEFAULT(0)` | هل العميل موقف |
| `CurrencyId` | `INT` | `NOT NULL, FK -> Currencies(CurrencyId)` | عملة العميل |
| `CreditLimit` | `DECIMAL(18,2)` | `NULL` | حد الدين |

**الاندكسات:**
- `PK_Customers` على `CustomerId`.
- `UQ_Customers_CustomerCode` على `CustomerCode`.
- `IX_Customers_IsStopped` على `IsStopped`.

**Model:**
```csharp
public class Customer
{
    public int CustomerId { get; set; }
    public string CustomerCode { get; set; }
    public string CustomerName { get; set; }
    public bool IsStopped { get; set; }
    public int CurrencyId { get; set; }
    public decimal? CreditLimit { get; set; }
}
```

**🔹 Sub-task 2: دالة `CheckIfStopped(CustomerId)`**

**DTO:**
```csharp
public class CustomerStatusDto
{
    public bool IsStopped { get; set; }
    public string Message { get; set; }
}
```

**🔹 Sub-task 3: دالة `CheckCreditLimit(CustomerId, CurrencyId, InvoiceTotal)`**

حساب: `إجمالي الأرصدة - المدفوعات` ومقارنته بـ `CreditLimit`.

**DTO:**
```csharp
public class CreditLimitCheckDto
{
    public decimal CurrentDebt { get; set; }
    public decimal? CreditLimit { get; set; }
    public decimal NewTotalDebt { get; set; }
    public decimal OverLimitAmount { get; set; }
    public decimal OverLimitPercentage { get; set; }
    public bool IsAllowed { get; set; }
    public string Message { get; set; }
}
```

**🔹 Sub-task 4: دمج النتيجتين في API واحد `api/customers/validate`**

```csharp
public class CustomerValidationRequestDto
{
    public int CustomerId { get; set; }
    public int CurrencyId { get; set; }
    public decimal InvoiceTotal { get; set; }
}

public class CustomerValidationResultDto
{
    public bool IsValid { get; set; }
    public bool IsStopped { get; set; }
    public bool CreditLimitExceeded { get; set; }
    public decimal OverLimitPercentage { get; set; }
    public string Message { get; set; }
}
```

---

#### **Task C.2 (Backend - Override Logic):** تطبيق صلاحية تجاوز حد الدين.

**🔹 Sub-task 1: تصميم جدول `UserClaims`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `UserClaimId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `UserId` | `INT` | `NOT NULL, FK -> Users(UserId)` | المستخدم |
| `ClaimType` | `NVARCHAR(100)` | `NOT NULL` | نوع الصلاحية |
| `ClaimValue` | `NVARCHAR(500)` | `NULL` | قيمة الصلاحية |

**الصلاحيات المطلوبة:**
- `AllowDebtOverride` = `true`
- `DebtOverridePercentage` = نسبة التجاوز المسموحة

**Model:**
```csharp
public class UserClaim
{
    public int UserClaimId { get; set; }
    public int UserId { get; set; }
    public string ClaimType { get; set; }
    public string ClaimValue { get; set; }
}
```

**🔹 Sub-task 2:** إذا كان التجاوز أقل من النسبة المئوية المسموحة، يسمح بالمرور مع `Warning`، وإلا يمنع.

---

#### **Task C.3 (Frontend):** تطبيق التحقق الفوري (`OnBlur` و `BeforeSave`).

**🔹 Sub-task 1:** عند الخروج من حقل العميل، استدعاء API وعرض رسائل الخطأ.

**🔹 Sub-task 2:** في حالة التجاوز المسموح به، عرض `Confirmation Dialog` (هل تريد المتابعة؟).

---

### **Feature Task D: إدارة المخازن (Store Management)**

**الوصف:**
بناء API لجلب المخازن المرتبطة بالفرع وبصلاحيات المستخدم، مع تمييز المخازن الموقفة ومنع البيع منها.

---

#### **Task D.1 (Backend):** API `api/stores/available`.

**🔹 Sub-task 1: تصميم جداول `Stores` و `UserStores` و `BranchStores`**

**جدول `Stores`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `StoreId` | `INT` | `PRIMARY KEY, IDENTITY` | معرف المخزن |
| `StoreCode` | `NVARCHAR(50)` | `NOT NULL, UNIQUE` | كود المخزن |
| `StoreName` | `NVARCHAR(200)` | `NOT NULL` | اسم المخزن |
| `IsActive` | `BIT` | `NOT NULL, DEFAULT(1)` | هل المخزن نشط |

**جدول `BranchStores`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `BranchStoreId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `BranchId` | `INT` | `NOT NULL, FK -> Branches(BranchId)` | الفرع |
| `StoreId` | `INT` | `NOT NULL, FK -> Stores(StoreId)` | المخزن |

**الاندكس:** `IX_BranchStores_BranchId` على `BranchId`.

**جدول `UserStores`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `UserStoreId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `UserId` | `INT` | `NOT NULL, FK -> Users(UserId)` | المستخدم |
| `StoreId` | `INT` | `NOT NULL, FK -> Stores(StoreId)` | المخزن |

**الاندكس:** `IX_UserStores_UserId` على `UserId`.

**Models:**
```csharp
public class Store
{
    public int StoreId { get; set; }
    public string StoreCode { get; set; }
    public string StoreName { get; set; }
    public bool IsActive { get; set; }
}

public class BranchStore
{
    public int BranchStoreId { get; set; }
    public int BranchId { get; set; }
    public int StoreId { get; set; }
    public Store Store { get; set; }
}

public class UserStore
{
    public int UserStoreId { get; set; }
    public int UserId { get; set; }
    public int StoreId { get; set; }
}
```

**DTO:**
```csharp
public class AvailableStoreDto
{
    public int StoreId { get; set; }
    public string StoreCode { get; set; }
    public string StoreName { get; set; }
    public bool IsActive { get; set; }
    public bool IsAllowed { get; set; }
}
```

**🔹 Sub-task 2:** جلب المخازن المرتبطة بـ `BranchId` المحدد.

**🔹 Sub-task 3:** فلترة النتائج بحسب صلاحيات المستخدم (`UserStores`).

**🔹 Sub-task 4:** إرجاع حقل `IsActive` مع كل مخزن لتحديد إذا كان موقفًا.

---

#### **Task D.2 (Frontend):** التعامل مع المخازن الموقفة.

**🔹 Sub-task 1:** تلوين المخازن الموقفة باللون الرمادي في القائمة.

**🔹 Sub-task 2:** عند اختيار مخزن موقف، عرض `MessageBox` ("تم إيقاف البيع") وتعطيل شبكة الأصناف (Grid).

---

## 📦 الحزمة التنفيذية رقم 3: قلب النظام (إدخال الأصناف - Details Grid)
*(تغطي القصص: #15 إلى #21 - الأصناف، الباركود، F9، الوحدات، الكميات)*  
*(هذه أكبر حزمة وأكثرها تعقيدًا)*

---

### **Feature Task E: محرك البحث عن الصنف (Item Search Engine)**

**الوصف:**
بناء محرك بحث متكامل للأصناف يدعم الباركود، رقم الصنف، البحث الجزئي، وقائمة F9، مع التحقق من وجود الصنف في المخزن المحدد.

---

#### **Task E.1 (Backend - API):** `api/items/search`.

**🔹 Sub-task 1: تصميم الجداول المرتبطة**

**جدول `Items`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `ItemId` | `INT` | `PRIMARY KEY, IDENTITY` | معرف الصنف |
| `ItemCode` | `NVARCHAR(50)` | `NOT NULL, UNIQUE` | رقم الصنف |
| `ItemName` | `NVARCHAR(500)` | `NOT NULL` | اسم الصنف |
| `DefaultSellUnitId` | `INT` | `NULL, FK -> Units(UnitId)` | وحدة البيع الافتراضية |
| `TaxPercentage` | `DECIMAL(5,2)` | `NOT NULL, DEFAULT(0)` | نسبة الضريبة |
| `IsActive` | `BIT` | `NOT NULL, DEFAULT(1)` | هل الصنف نشط |

**الاندكسات:**
- `PK_Items` على `ItemId`.
- `UQ_Items_ItemCode` على `ItemCode`.
- `IX_Items_ItemName` على `ItemName`.

**جدول `Barcodes`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `BarcodeId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `Barcode` | `NVARCHAR(100)` | `NOT NULL, UNIQUE` | الباركود |
| `ItemId` | `INT` | `NOT NULL, FK -> Items(ItemId)` | الصنف |
| `UnitId` | `INT` | `NULL, FK -> Units(UnitId)` | الوحدة |

**الاندكسات:**
- `PK_Barcodes` على `BarcodeId`.
- `UQ_Barcodes_Barcode` على `Barcode`.
- `IX_Barcodes_ItemId` على `ItemId`.

**جدول `ItemBalances`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `ItemBalanceId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `ItemId` | `INT` | `NOT NULL, FK -> Items(ItemId)` | الصنف |
| `StoreId` | `INT` | `NOT NULL, FK -> Stores(StoreId)` | المخزن |
| `UnitId` | `INT` | `NOT NULL, FK -> Units(UnitId)` | الوحدة |
| `Quantity` | `DECIMAL(18,4)` | `NOT NULL, DEFAULT(0)` | الكمية |

**الاندكس:** `IX_ItemBalances_Item_Store_Unit` على `(ItemId, StoreId, UnitId)`.

**Models:**
```csharp
public class Item
{
    public int ItemId { get; set; }
    public string ItemCode { get; set; }
    public string ItemName { get; set; }
    public int? DefaultSellUnitId { get; set; }
    public decimal TaxPercentage { get; set; }
    public bool IsActive { get; set; }
}

public class Barcode
{
    public int BarcodeId { get; set; }
    public string BarcodeValue { get; set; }
    public int ItemId { get; set; }
    public int? UnitId { get; set; }
}

public class ItemBalance
{
    public int ItemBalanceId { get; set; }
    public int ItemId { get; set; }
    public int StoreId { get; set; }
    public int UnitId { get; set; }
    public decimal Quantity { get; set; }
}
```

**🔹 Sub-task 2: البحث المباشر:**
- التحقق من جدول `Barcodes` (إذا تطابق، استخراج `ItemId`).
- المطابقة التامة في `ItemCode`.

**🔹 Sub-task 3: التحقق من الوجود في المخزن:**
- `JOIN` مع `ItemBalances` حيث `StoreId == SelectedStoreId`.
- إذا لم يُوجد، إرجاع رسالة: "الصنف غير موجود في هذا المخزن".

**🔹 Sub-task 4: البحث الجزئي (Partial/Contains):**
- إذا لم يُجد مطابقة تامة، استخدام `LIKE '%input%'` لإرجاع قائمة.

**DTOs:**
```csharp
public class ItemSearchRequestDto
{
    public string SearchText { get; set; }
    public int StoreId { get; set; }
    public int? DefaultSearchMode { get; set; } // 1=StartsWith, 2=EndsWith, 3=Contains
}

public class ItemSearchResultDto
{
    public int ItemId { get; set; }
    public string ItemCode { get; set; }
    public string ItemName { get; set; }
    public int? DefaultSellUnitId { get; set; }
    public string DefaultSellUnitName { get; set; }
    public decimal AvailableQuantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal TaxPercentage { get; set; }
    public bool IsExactMatch { get; set; }
}
```

---

#### **Task E.2 (Frontend - Search UI):** ربط حقل الإدخال.

**🔹 Sub-task 1:** ربط `(keydown.enter)` لاستدعاء API.

**🔹 Sub-task 2:** إذا رجع API بيانات (اسم، سعر، وحدة)، تعبئة الصف والانتقال (`Focus`) إلى حقل الكمية.

**🔹 Sub-task 3:** إذا رجع API قائمة (بحث جزئي)، فتح `Modal` يعرض النتائج مع إمكانية الضغط المزدوج (Double-click) للاختيار.

---

### **Feature Task F: قائمة F9 الذكية (Smart F9 Lookup)**

**الوصف:**
بناء API وواجهة مستخدم لعرض قائمة الأصناف عند الضغط على F9، مع دعم الفلاتر والبحث حسب معيار المستخدم الافتراضي.

---

#### **Task F.1 (Backend):** API `api/items/store-items`.

**🔹 Sub-task 1: قراءة متغيرات النظمة (`SystemSettings`)**

**جدول `SystemSettings`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `SettingId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `SettingKey` | `NVARCHAR(100)` | `NOT NULL, UNIQUE` | مفتاح المتغير |
| `SettingValue` | `NVARCHAR(500)` | `NULL` | قيمة المتغير |

**المتغيرات المطلوبة:**
- `ShowZeroQuantityItems`
- `ShowOnlyPricedItems`
- `IsCustomerNameRequiredInCash`

**Model:**
```csharp
public class SystemSetting
{
    public int SettingId { get; set; }
    public string SettingKey { get; set; }
    public string SettingValue { get; set; }
}
```

**DTO:**
```csharp
public class InvoiceSettingsDto
{
    public bool ShowZeroQuantityItems { get; set; }
    public bool ShowOnlyPricedItems { get; set; }
    public bool IsCustomerNameRequiredInCash { get; set; }
}
```

**🔹 Sub-task 2:** تطبيق المتغيرات في استعلام واحد.

**🔹 Sub-task 3:** دعم معايير البحث (StartsWith, EndsWith, Contains) استنادًا إلى `User.DefaultSearchMode`.

---

#### **Task F.2 (Frontend):** ربط زر F9.

**🔹 Sub-task 1:** إذا كان الحقل فارغًا، عرض كل الأصناف (وفق الفلاتر).

**🔹 Sub-task 2:** إذا كان هناك نص، تطبيق معيار البحث وإظهار النتائج في `Popup`.

---

### **Feature Task G: إدارة الوحدات والكميات (Units & Quantities)**

**الوصف:**
بناء منطق الوحدات ومعاملات التحويل والتحقق من الكميات المتوفرة عند تغيير الوحدة.

---

#### **Task G.1 (Backend):** جلب وحدات الصنف مع معامل التحويل.

**🔹 Sub-task 1: تصميم جدول `ItemUnits` و `Units`**

**جدول `Units`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `UnitId` | `INT` | `PRIMARY KEY, IDENTITY` | معرف الوحدة |
| `UnitCode` | `NVARCHAR(50)` | `NOT NULL` | كود الوحدة |
| `UnitName` | `NVARCHAR(100)` | `NOT NULL` | اسم الوحدة |

**جدول `ItemUnits`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `ItemUnitId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `ItemId` | `INT` | `NOT NULL, FK -> Items(ItemId)` | الصنف |
| `UnitId` | `INT` | `NOT NULL, FK -> Units(UnitId)` | الوحدة |
| `ConversionFactor` | `DECIMAL(18,6)` | `NOT NULL, DEFAULT(1)` | معامل التحويل للوحدة الرئيسية |
| `IsDefaultSell` | `BIT` | `NOT NULL, DEFAULT(0)` | هل هي وحدة البيع الافتراضية |

**الاندكس:** `IX_ItemUnits_ItemId` على `ItemId`.

**Models:**
```csharp
public class Unit
{
    public int UnitId { get; set; }
    public string UnitCode { get; set; }
    public string UnitName { get; set; }
}

public class ItemUnit
{
    public int ItemUnitId { get; set; }
    public int ItemId { get; set; }
    public int UnitId { get; set; }
    public decimal ConversionFactor { get; set; }
    public bool IsDefaultSell { get; set; }
}
```

**DTO:**
```csharp
public class ItemUnitDto
{
    public int UnitId { get; set; }
    public string UnitName { get; set; }
    public decimal ConversionFactor { get; set; }
    public bool IsDefaultSell { get; set; }
}
```

**🔹 Sub-task 2:** عند اختيار الصنف، جلب `DefaultSellUnitId` وقائمة `ItemUnits` مع `ConversionFactor`.

---

#### **Task G.2 (Frontend - Unit Change Logic):**

**🔹 Sub-task 1:** تعبئة الوحدة تلقائيًا.

**🔹 Sub-task 2:** ربط `(change)` على حقل الوحدة. عند التغيير، استدعاء API لجلب الكمية المتوفرة (`AvailableQuantity`) بالوحدة الجديدة.

**🔹 Sub-task 3:** إذا كانت الكمية المدخلة حاليًا > المتوفرة الجديدة، مسح الكمية وعرض تحذير "الكمية غير متوفرة بهذه الوحدة".

---

#### **Task G.3 (Frontend - Quantity Validation):**

**🔹 Sub-task 1:** ربط `(blur)` على حقل الكمية. إذا كانت القيمة > المتوفرة، منع الحفظ وتعطيل زر الحفظ.

---

## 📦 الحزمة التنفيذية رقم 4: التسعير والضرائب والإجماليات (Pricing & Calculations)
*(تغطي القصص: #22 إلى #27 - السعر، التكلفة، الخصم، الضريبة، الإجمالي)*

---

### **Feature Task H: محرك التسعير والتكاليف (Pricing Engine)**

**الوصف:**
بناء محرك تسعير يجلب السعر المسبق للصنف ويمنع البيع بأقل من التكلفة، مع التحكم في صلاحية تعديل السعر.

---

#### **Task H.1 (Backend):** دالة `GetItemPrice`.

**🔹 Sub-task 1: تصميم جدول `Pricing`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `PricingId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `ItemId` | `INT` | `NOT NULL, FK -> Items(ItemId)` | الصنف |
| `CustomerId` | `INT` | `NULL, FK -> Customers(CustomerId)` | العميل |
| `PriceCategoryId` | `INT` | `NULL` | فئة السعر |
| `UnitId` | `INT` | `NOT NULL, FK -> Units(UnitId)` | الوحدة |
| `Price` | `DECIMAL(18,4)` | `NOT NULL` | السعر |
| `EffectiveDate` | `DATETIME` | `NOT NULL` | تاريخ السريان |

**الاندكس:** `IX_Pricing_Item_Customer_Unit` على `(ItemId, CustomerId, UnitId, EffectiveDate)`.

**جدول `ItemCosts`:**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `ItemCostId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `ItemId` | `INT` | `NOT NULL, FK -> Items(ItemId)` | الصنف |
| `UnitId` | `INT` | `NOT NULL, FK -> Units(UnitId)` | الوحدة |
| `Cost` | `DECIMAL(18,4)` | `NOT NULL` | التكلفة |

**الاندكس:** `IX_ItemCosts_Item_Unit` على `(ItemId, UnitId)`.

**Models:**
```csharp
public class Pricing
{
    public int PricingId { get; set; }
    public int ItemId { get; set; }
    public int? CustomerId { get; set; }
    public int? PriceCategoryId { get; set; }
    public int UnitId { get; set; }
    public decimal Price { get; set; }
    public DateTime EffectiveDate { get; set; }
}

public class ItemCost
{
    public int ItemCostId { get; set; }
    public int ItemId { get; set; }
    public int UnitId { get; set; }
    public decimal Cost { get; set; }
}
```

**DTO:**
```csharp
public class ItemPriceRequestDto
{
    public int ItemId { get; set; }
    public int UnitId { get; set; }
    public int? CustomerId { get; set; }
    public int CurrencyId { get; set; }
}

public class ItemPriceDto
{
    public decimal UnitPrice { get; set; }
    public decimal ItemCost { get; set; }
    public bool HasPricing { get; set; }
}
```

**🔹 Sub-task 2:** جلب السعر من جدول `Pricing` (مرتبط بالصنف، العميل، أو الفئة السعرية).

**🔹 Sub-task 3:** جلب `ItemCost` من جدول التكاليف.

---

#### **Task H.2 (Frontend - Permissions & Cost):**

**🔹 Sub-task 1:** إذا كان المستخدم لا يملك صلاحية `CanEditPrice`، جعل حقل السعر `Readonly`.

**🔹 Sub-task 2:** عند محاولة تغيير السعر يدويًا (`OnChange`)، التحقق الفوري: إذا كان `NewPrice < ItemCost`، عرض رسالة خطأ ورفض التحديث.

---

### **Feature Task I: حساب الضريبة والإجمالي (Tax & Totals)**

**الوصف:**
بناء محرك حسابات لحساب الضريبة والإجماليات على مستوى الصنف والفاتورة.

---

#### **Task I.1 (Backend):** تمرير `TaxPercentage` مع بيانات الصنف.

**DTO (موسع):**
```csharp
public class ItemDetailDto
{
    public int ItemId { get; set; }
    public string ItemName { get; set; }
    public int UnitId { get; set; }
    public string UnitName { get; set; }
    public decimal Quantity { get; set; }
    public decimal FreeQuantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal Discount { get; set; }
    public decimal TaxPercentage { get; set; }
    public decimal TaxAmount { get; set; }
    public decimal ItemTotal { get; set; }
}
```

---

#### **Task I.2 (Frontend - Formula Engine):** تطبيق المعادلة الرئيسية في الـ Grid.

**🔹 Sub-task 1:** `ItemTotal = (الكمية × (السعر + الضريبة - الخصم))`.

**🔹 Sub-task 2:** ربط هذه المعادلة بـ `OnChange` لأي حقل (كمية، سعر، خصم) لتحديث الإجمالي ديناميكيًا.

---

#### **Task I.3 (Frontend - Footer):** حساب إجمالي الفاتورة.

**🔹 Sub-task 1:** جمع كل `ItemTotal` وعرضه في `Footer` أسفل الشبكة.

---

## 📦 الحزمة التنفيذية رقم 5: العروض الترويجية وصلاحيات CRUD (Advanced)
*(تغطي القصص: #28 إلى #33 - العروض، الصلاحيات، البحث، المتغيرات)*

---

### **Feature Task J: محرك العروض الترويجية (Promotion Engine)**

**الوصف:**
بناء محرك عروض ترويجية يدعم العروض على مستوى الصنف (كمية مجانية) وعلى مستوى الفاتورة (خصم عند تجاوز مبلغ).

---

#### **Task J.1 (Backend - Promotion Checker):**

**🔹 Sub-task 1: تصميم جدول `Promotions`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `PromotionId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `PromotionName` | `NVARCHAR(200)` | `NOT NULL` | اسم العرض |
| `PromotionType` | `TINYINT` | `NOT NULL` | 1=ItemLevel, 2=InvoiceLevel |
| `ItemId` | `INT` | `NULL, FK -> Items(ItemId)` | الصنف (للعرض على مستوى الصنف) |
| `Threshold` | `DECIMAL(18,2)` | `NULL` | الحد الأدنى (كمية أو مبلغ) |
| `DiscountPercentage` | `DECIMAL(5,2)` | `NULL` | نسبة الخصم |
| `FreeQuantity` | `DECIMAL(18,4)` | `NULL` | الكمية المجانية |
| `StartDate` | `DATETIME` | `NOT NULL` | بداية العرض |
| `EndDate` | `DATETIME` | `NOT NULL` | نهاية العرض |
| `IsActive` | `BIT` | `NOT NULL, DEFAULT(1)` | هل العرض نشط |

**الاندكسات:**
- `PK_Promotions` على `PromotionId`.
- `IX_Promotions_Type_Item` على `(PromotionType, ItemId)`.
- `IX_Promotions_ActiveDates` على `(StartDate, EndDate, IsActive)`.

**Model:**
```csharp
public class Promotion
{
    public int PromotionId { get; set; }
    public string PromotionName { get; set; }
    public PromotionType PromotionType { get; set; }
    public int? ItemId { get; set; }
    public decimal? Threshold { get; set; }
    public decimal? DiscountPercentage { get; set; }
    public decimal? FreeQuantity { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime EndDate { get; set; }
    public bool IsActive { get; set; }
}
```

**DTO:**
```csharp
public class PromotionResultDto
{
    public int? PromotionId { get; set; }
    public string PromotionName { get; set; }
    public decimal FreeQuantity { get; set; }
    public decimal DiscountAmount { get; set; }
    public decimal DiscountPercentage { get; set; }
}
```

**🔹 Sub-task 2:** عند إضافة صف، فحص العروض النشطة (Item Level). إذا تحقق الشرط (مثلًا الكمية >= 2)، حساب `FreeQuantity` وإرجاعها.

**🔹 Sub-task 3:** عند حساب الإجمالي الكلي، فحص العروض (Invoice Level). إذا تجاوز `Total` قيمة `Threshold`، حساب خصم إضافي وتطبيقه على صافي الفاتورة.

---

#### **Task J.2 (Frontend):**

**🔹 Sub-task 1:** عرض `FreeQuantity` في حقل للقراءة فقط عند جلبها.

**🔹 Sub-task 2:** عند تطبيق خصم الفاتورة، تحديث `Grand Total` في منطقة التذييل.

---

### **Feature Task K: الصلاحيات وشاشة البحث (Permissions & Lookup)**

**الوصف:**
تطبيق صلاحيات CRUD على مستوى الواجهة والـ Backend، وبناء شاشة البحث عن الفواتير.

---

#### **Task K.1 (Backend - Authorization):**

**🔹 Sub-task 1: تصميم جدول `RolePermissions` أو `UserPermissions`**

| اسم الحقل | نوع البيانات | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `PermissionId` | `INT` | `PRIMARY KEY, IDENTITY` | المعرف |
| `UserId` | `INT` | `NOT NULL, FK -> Users(UserId)` | المستخدم |
| `PermissionKey` | `NVARCHAR(100)` | `NOT NULL` | مفتاح الصلاحية |
| `IsGranted` | `BIT` | `NOT NULL, DEFAULT(0)` | هل ممنوحة |

**الصلاحيات المطلوبة:**
- `Invoice.Add`
- `Invoice.Edit`
- `Invoice.Delete`
- `Invoice.View`

**Model:**
```csharp
public class UserPermission
{
    public int PermissionId { get; set; }
    public int UserId { get; set; }
    public string PermissionKey { get; set; }
    public bool IsGranted { get; set; }
}
```

**🔹 Sub-task 2:** إضافة `[Authorize(Policy = "AddInvoice")]` على `POST`، و `Edit`، و `Delete`، و `View`.

---

#### **Task K.2 (Frontend - UI Control):**

**🔹 Sub-task 1:** في شاشة الإضافة، إذا لم تكن صلاحية "إضافة" موجودة، تعطيل زر "جديد".

**🔹 Sub-task 2:** في شاشة البحث، إذا لم تكن صلاحية "تعديل" موجودة، تعطيل زر التعديل أو جعل الصف للقراءة فقط.

---

#### **Task K.3 (Frontend - Search Screen):** بناء شاشة بحث (`Invoice Search`).

**🔹 Sub-task 1:** إنشاء `Grid` مع فلاتر (رقم الفاتورة، العميل، الفرع، التاريخ).

**🔹 Sub-task 2:** ربط الـ Grid بـ API البحث (`api/invoices/search`).

**DTO:**
```csharp
public class InvoiceSearchRequestDto
{
    public int? InvoiceNumber { get; set; }
    public int? CustomerId { get; set; }
    public int? BranchId { get; set; }
    public DateTime? FromDate { get; set; }
    public DateTime? ToDate { get; set; }
}

public class InvoiceSearchResultDto
{
    public long InvoiceId { get; set; }
    public Guid InvoiceUID { get; set; }
    public int InvoiceNumber { get; set; }
    public string BranchName { get; set; }
    public string CustomerName { get; set; }
    public string PaymentMethodName { get; set; }
    public decimal NetAmount { get; set; }
    public DateTime InvoiceDate { get; set; }
}
```

**🔹 Sub-task 3:** عند الضغط المزدوج على صف، فتح شاشة الفاتورة في وضع (عرض) أو (تعديل) حسب الصلاحية.

---

### **Feature Task L: المتغيرات العامة (Global Settings)**

**الوصف:**
بناء API لقراءة المتغيرات العامة المؤثرة على سلوك الفاتورة وتطبيقها في الواجهة.

---

#### **Task L.1 (Backend):** API `api/settings/invoice`.

**🔹 Sub-task 1:** إرجاع جميع المتغيرات المؤثرة:
- `IsCustomerNameRequired`
- `ShowZeroItems`
- `ShowOnlyPriced`

**DTO (مكرر من F.1 للرجوع):**
```csharp
public class InvoiceSettingsDto
{
    public bool IsCustomerNameRequired { get; set; }
    public bool ShowZeroItems { get; set; }
    public bool ShowOnlyPriced { get; set; }
}
```

---

#### **Task L.2 (Frontend):**

**🔹 Sub-task 1:** عند تحميل شاشة الفاتورة (`OnInit`)، جلب هذه المتغيرات وتخزينها في `AppState` أو `Context`.

**🔹 Sub-task 2:** تطبيق `IsCustomerNameRequired` على حقل اسم العميل النقدي (جعله `Required` أو لا).

**🔹 Sub-task 3:** تمرير `ShowZeroItems` و `ShowOnlyPriced` كـ Parameters عند استدعاء API جلب أصناف F9.

---

## ⏱️ تقدير الجهد (Estimation) للمطورين

| الحزمة (Package) | عدد المهام الرئيسية | إجمالي المهام الفرعية | الوقت التقديري (ساعات) |
| :--- | :--- | :--- | :--- |
| الحزمة 1 (Master Setup) | 2 Tasks | 10 Sub-tasks | 10 - 12 ساعات |
| الحزمة 2 (Customers & Stores) | 4 Tasks | 14 Sub-tasks | 18 - 24 ساعة |
| **الحزمة 3 (Details Grid - الأكبر)** | **7 Tasks** | **28 Sub-task** | **36 - 48 ساعة** |
| الحزمة 4 (Pricing & Calculations) | 3 Tasks | 10 Sub-tasks | 12 - 16 ساعة |
| الحزمة 5 (Promotions & Permissions) | 4 Tasks | 14 Sub-tasks | 16 - 20 ساعة |
| **المجموع الكلي** | **20 مهمة رئيسية** | **76 مهمة فرعية تفصيلية** | **~90 - 120 ساعة (تقريبًا 6 - 7 سباقات)** |

---

## ✅ تعليمات التوزيع على فريق Scrum

1.  **قاعدة بيانات (DBA):** يركز على **Task A.1** (الترقيم) وجميع مهام `Sub-task` المتعلقة بكتابة `Stored Procedures` و `Indexes`.
2.  **مطور Backend (الباك إند):** ينفذ **جميع مهام APIs** (التحقق من العملاء، محرك البحث، محرك العروض، المتغيرات) بالتوازي مع قاعدة البيانات.
3.  **مطور Frontend (الفرونت إند):** يبدأ بـ **Task B.2** و **D.2** (المنطق البصري)، ثم ينتقل فورًا إلى **Task E.2** (شبكة الأصناف) فور جاهزية APIs الخاصة بالبحث.

---

**ملاحظة خبير:** تم تفكيك "البحث الجزئي" و "الباركود" في Sub-task واحدة (E.1 - 4) لأن منطقهم الخلفي متشابه (بحث عن نص)، ويتم الفصل بينهم في طبقة الخدمة (Service Layer) فقط. كما تم دمج "منع السعر أقل من التكلفة" مع "صلاحية تعديل السعر" في مهمة واحدة (H.2) لأن كلاهما يتعامل مع `OnChange` الخاص بحقل السعر.

هذا التفكيك **لا يترك أي مجال للتخمين** للمطورين. كل Sub-task محددة بـ "ماذا سأفعل" و "أين سأكتب الكود".
