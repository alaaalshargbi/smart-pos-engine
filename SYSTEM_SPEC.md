وثيقة المواصفات الهندسية الشاملة (System Requirements & Engineering Specification - SRS)** لمشروع **Smart-POS**، مصممة بأعلى معايير هندسة البرمجيات لتكون دليلاً هندسياً كاملاً، وجاهزة للإدخال مباشرة إلى **Maestro AI Agent** والوكلاء المنفذين في بيئة GitHub Codespaces.

---

# مواصفات النظام: Smart-POS (Cloud & Offline-First POS Engine)
* **المعمارية الأساسية:** Domain-Driven Modular Monolith (Laravel 11 API) + Reactive SPA (Vue.js 3 / Pinia / Tailwind CSS).
* **قاعدة البيانات:** PostgreSQL 16 أو MySQL 8.0 (مع دعم JSONB و Pessimistic Locking).
* **التخزين المحلي (Offline):** IndexedDB (عبر Dexie.js) + Service Worker (PWA).

---

## 0. الهيكل المعماري للمشروع (Project Directory Architecture)

تم تقسيم المشروع بأسلوب **Modular Architecture** لعزل نطاق عمل كل وكيل برمجياً، مما يمنع حدوث أي تعارض ملفات (Git / Code Collisions) أثناء عمل الوكلاء المتوازيين:

```text
smart-pos/
├── backend/ (Laravel 11 Core)
│   ├── app/
│   │   ├── Modules/
│   │   │   ├── Auth/           <-- Agent 1 (Auth & Tenants)
│   │   │   │   ├── Controllers/
│   │   │   │   ├── Models/
│   │   │   │   ├── Requests/
│   │   │   │   ├── Services/
│   │   │   │   └── Tests/
│   │   │   ├── Shift/          <-- Agent 1 (Cashier Registers & Shifts)
│   │   │   │   ├── Controllers/
│   │   │   │   ├── Models/
│   │   │   │   ├── Services/
│   │   │   │   └── Tests/
│   │   │   ├── Catalog/        <-- Agent 2 (Products & Inventory)
│   │   │   │   ├── Controllers/
│   │   │   │   ├── Models/
│   │   │   │   ├── Services/
│   │   │   │   └── Tests/
│   │   │   ├── Invoicing/      <-- Agent 4 (ZATCA, Payments & Taxes)
│   │   │   │   ├── Controllers/
│   │   │   │   ├── Models/
│   │   │   │   ├── Services/ (ZatcaTlvEncoder.php)
│   │   │   │   └── Tests/
│   │   │   └── Sync/           <-- Agent 5 (Bulk Offline Sync API)
│   │   │       ├── Controllers/
│   │   │       ├── Services/
│   │   │       └── Tests/
│   │   ├── Scopes/             <-- Multi-Tenant Global Scope
│   │   └── Traits/             <-- BelongsToTenant, UUIDPrimaryKey
│   ├── database/
│   │   └── migrations/         <-- Central Migration Engine (Chronological)
│   └── routes/
│       ├── api.php             <-- Central Loader
│       └── modules/            <-- Individual route files per module
│           ├── auth.php
│           ├── shifts.php
│           ├── catalog.php
│           ├── invoices.php
│           └── sync.php
└── frontend/ (Vue.js 3 / Vite / Tailwind)
    ├── src/
    │   ├── api/                <-- Axios client with offline interceptors
    │   ├── components/         <-- POS Common UI (Keypad, Modals)
    │   ├── modules/
    │   │   ├── auth/           <-- Views & Components
    │   │   ├── shift/          <-- Shift open/close modals & reports
    │   │   ├── pos/            <-- Agent 3 (Terminal, Cart, Hold Engine)
    │   │   └── sync/           <-- Agent 5 (IndexedDB Manager & Sync Monitor)
    │   ├── stores/             <-- Pinia Stores (cart.ts, shift.ts, offline.ts)
    │   ├── db/                 <-- Dexie.js Schema (IndexedDB tables)
    │   └── utils/              <-- ZATCA TLV generator (client-side JS)
```

---

## الوحدة 1: وحدة تسجيل الدخول وإدارة الشركات والفروع (Multi-Tenant & Cashier Auth Module)

### 1.1 الـ Wireframe (هيكل الواجهة)
```text
+-------------------------------------------------------------+
|                     SMART LIFE POS                          |
|               بوابة نقاط البيع السحابية                     |
+-------------------------------------------------------------+
|                                                             |
|   +-----------------------------------------------------+   |
|   |  رمز الشركة (Company Slug): [  acme-retail       ]  |   |
|   +-----------------------------------------------------+   |
|   |  اسم المستخدم / الكاشير:    [  cashier_ahmed     ]  |   |
|   +-----------------------------------------------------+   |
|   |  كلمة المرور:               [  ••••••••••••      ]  |   |
|   +-----------------------------------------------------+   |
|   |  الفرع: [ فرع الرياض - المستودع الرئيسي      ▼ ]   |   |
|   +-----------------------------------------------------+   |
|                                                             |
|               [    تسجيل الدخول للنظام (F9)    ]            |
|                                                             |
|   (o) الاتصال بالسيرفر: متصل وآمن (TLS 1.3)                 |
+-------------------------------------------------------------+
```

### 1.2 الـ UI/UX
* **التفاعل:** عند كتابة `Company Slug`، يتم التحقق منه في الخلفية (Debounced 300ms) للتأكد من وجود الشركة وجلب الفروع المسموح للمستخدم بها تلقائياً.
* **إمكانية الوصول:** التنقل بالكامل بين الحقول عبر زر `Enter` أو `Tab`. الضغط على `F9` يقوم بتسجيل الدخول الفوري دون الحاجة للمس الفأرة.
* **الأخطاء:** تظهر رسائل الخطأ أسفل كل حقل بنص أحمر واضح واهتزاز خفيف (Micro-shake animation) للحقل غير الصحيح.

### 1.3 منطق العمل وسير الأحداث (Logics & Workflows)
1. يستقبل النظام طلب تسجيل الدخول: التحقق من وجود الشركة عبر الـ `Slug` والتأكد من أنها بحالة نشطة (`status = ACTIVE`).
2. التحقق من صلاحية المستخدم بالنسبة لهذه الشركة حصراً (عزل كامل).
3. التأكد من أن المستخدم مصرح له بالعمل على الفرع المختار (`branch_id`).
4. توليد Sanctum Token يحتوي على الـ Claims التالية: `tenant_id`, `branch_id`, `role`.
5. تحميل وحفظ بيانات الفرع وإعدادات الضرائب في متجر الـ Pinia والمتصفح.

### 1.4 جداول ومخطط البيانات (Database Schema & Models)
```sql
-- جدول الشركات (Tenants)
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(150) NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'acme-retail'
    tax_number VARCHAR(15) NOT NULL, -- 15 digits for ZATCA
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_companies_slug ON companies(slug);

-- جدول الفروع
CREATE TABLE branches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) NOT NULL,
    address TEXT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_branches_company ON branches(company_id);

-- جدول المستخدمين
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    branch_id UUID NULL REFERENCES branches(id),
    name VARCHAR(100) NOT NULL,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'CASHIER', -- 'ADMIN', 'SUPERVISOR', 'CASHIER'
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT uq_company_username UNIQUE (company_id, username)
);
```

### 1.5 قواعد التحقق (Validation Rules)
```php
public function rules(): array {
    return [
        'company_slug' => ['required', 'string', 'exists:companies,slug'],
        'username'     => ['required', 'string'],
        'password'     => ['required', 'string', 'min:6'],
        'branch_id'    => ['required', 'uuid', 'exists:branches,id']
    ];
}
```

### 1.6 الأمان والحماية (Security)
* **Tenant Scoping:** تفعيل Global Scope `BelongsToTenant` على كل استعلام يمنع قراءة أي سجل من شركة أخرى حتى في حال وجود ثغرة IDOR.
* **Rate Limiting:** حد أقصى 5 محاولات فاشلة لتسجيل الدخول لكل IP/دقيقة (تجنب الـ Brute Force).
* **Password Hashing:** استخدام خوارزمية `Argon2id` أو `Bcrypt`.

### 1.7 اختبارات الوحدة والتكامل (Unit & Feature Tests)
* `test_user_cannot_login_with_invalid_company_slug()`
* `test_user_cannot_access_branch_of_another_company()`
* `test_successful_authentication_returns_sanctum_token_with_tenant_claims()`

---

## الوحدة 2: وحدة إدارة ورديات الكاشير والصندوق (Cashier Shift & Register Management Module)

### 2.1 الـ Wireframe
```text
+-------------------------------------------------------------+
|               [وردية جديدة - فتح الصندوق]                    |
+-------------------------------------------------------------+
| الكاشير: أحمد سعيد            التاريخ: 2026-10-30 08:00 AM   |
| الفرع: فرع الرياض - نقطة بيع 1                              |
|-------------------------------------------------------------|
| الرصيد الافتتاحي في الدرج (Opening Cash Float):             |
| [  500.00  ] ر.س                                            |
|                                                             |
| ملاحظات الوردية: [ عهدة نقدية فئات صغيرة                  ]  |
|-------------------------------------------------------------|
|      [ إلغاء ]                     [ تأكيد فتح الوردية ]    |
+-------------------------------------------------------------+

نافذة إغلاق الوردية (End of Shift / Z-Report):
+-------------------------------------------------------------+
| المتوقع في الدرج: 2,450.00 ر.س | الفعلي المدخل: [ 2,450.00 ] |
| مبيعات الشبكة:   1,800.00 ر.س | الفارق (العجز/الزيادة): 0.00|
|              [ طباعة Z-Report وإغلاق الوردية ]              |
+-------------------------------------------------------------+
```

### 2.2 الـ UI/UX
* إذا لم تكن هناك وردية مفتوحة للكاشير، يتم حجب شاشة البيع بالكامل، وتظهر نافذة منبثقة غير قابلة للإغلاق (Modal) تطالبه بإدخال الرصيد الافتتاحي.
* شريط علوي صغير ودائم يعرض: رقم الوردية الحالية، وقت البدء، وزر سريع "مصروف درج سريع (Cash Out)".

### 2.3 منطق العمل وسير الأحداث (Logics & Workflows)
1. **التحقق المانع:** لا يمكن إصدار فاتورة أو تعليق عملية بيع إلا إذا كان هناك سجل وردية نشط (`status = 'OPEN'`) للمستخدم الحالي على نفس الجهاز (`terminal_id`).
2. **تسجيل حركة الأموال الداخلية:**
   * `CASH_IN`: إضافة نقد للدرج (مثل زيادة فكة).
   * `CASH_OUT`: سحب نقدي للمصاريف النثرية أو توريد جزئي للخزينة.
3. **معادلة إغلاق الوردية:**
   $$\text{Expected Cash} = \text{Opening Float} + \sum(\text{Cash Sales}) + \sum(\text{Cash In}) - \sum(\text{Cash Out})$$
   $$\text{Variance} = \text{Actual Cash Entered} - \text{Expected Cash}$$
4. يُسجل الفارق (سواء كان عجزاً أو زيادة) ويرحل إلى تقرير المشرف.

### 2.4 جداول ومخطط البيانات (Database Schema & Models)
```sql
CREATE TYPE shift_status AS ENUM ('OPEN', 'CLOSED');

CREATE TABLE cashier_shifts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id),
    branch_id UUID NOT NULL REFERENCES branches(id),
    user_id UUID NOT NULL REFERENCES users(id),
    terminal_id VARCHAR(50) NOT NULL,
    status shift_status DEFAULT 'OPEN',
    opening_balance DECIMAL(15, 4) NOT NULL,
    actual_closing_balance DECIMAL(15, 4) NULL,
    expected_closing_balance DECIMAL(15, 4) NULL,
    cash_variance DECIMAL(15, 4) NULL, -- difference
    opened_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    closed_at TIMESTAMP WITH TIME ZONE NULL
);
CREATE INDEX idx_shifts_user_status ON cashier_shifts(user_id, status);

CREATE TABLE cash_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shift_id UUID NOT NULL REFERENCES cashier_shifts(id) ON DELETE CASCADE,
    type VARCHAR(20) NOT NULL, -- 'CASH_IN', 'CASH_OUT'
    amount DECIMAL(15, 4) NOT NULL,
    reason VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 2.5 قواعد التحقق (Validation Rules)
```php
// فتح الوردية
public function openShiftRules(): array {
    return [
        'opening_balance' => ['required', 'numeric', 'min:0'],
        'terminal_id'     => ['required', 'string', 'max:50']
    ];
}

// إغلاق الوردية
public function closeShiftRules(): array {
    return [
        'actual_closing_balance' => ['required', 'numeric', 'min:0'],
        'notes'                  => ['nullable', 'string', 'max:500']
    ];
}
```

### 2.6 الأمان والحماية (Security)
* **Atomic State:** لا يمكن لكاشير فتح ورديتين في وقت واحد (Unique Partial Index على `user_id` حيث `status = 'OPEN'`).
* **إذن الإغلاق القسري:** لا يمكن إغلاق وردية كاشير آخر إلا بمستخدم يمتلك صلاحية `SUPERVISOR` أو `ADMIN`.

### 2.7 اختبارات الوحدة والتكامل (Unit & Feature Tests)
* `test_cannot_create_sale_without_an_active_open_shift()`
* `test_cannot_open_two_concurrent_shifts_for_same_user()`
* `test_reconciliation_calculates_cash_variance_accurately()`

---

## الوحدة 3: وحدة الكتالوج والمخزون السريع (Catalog & Real-Time Inventory Module)

### 3.1 الـ Wireframe
```text
+-------------------------------------------------------------------+
| [ باركود / اسم الصنف (F1)                    ] [ جميع التصنيفات ▼] |
+-------------------------------------------------------------------+
| +-------------+  +-------------+  +-------------+  +-------------+ |
| | [صورة صنف]  |  | [صورة صنف]  |  | [صورة صنف]  |  | [صورة صنف]  | |
| | عصير تفاح   |  | حليب 1 لتر  |  | أرز بسمتي   |  | زيت ذرة     | |
| | 5.00 ر.س    |  | 7.50 ر.س    |  | 45.00 ر.س   |  | 18.00 ر.س   | |
| | [المخزن: 42]|  | [المخزن: 05]|  | [المخزن: 00]|  | [المخزن: 19]| |
| | [شامل ضريبة]|  | [شامل ضريبة]|  | [نفذت الكمية]| | [شامل ضريبة]| |
| +-------------+  +-------------+  +-------------+  +-------------+ |
+-------------------------------------------------------------------+
```

### 3.2 الـ UI/UX
* عرض الأصناف في بطاقات سهلة اللمس (Touch-Friendly Cards) مع تمييز لوني واضح للأصناف منخفضة المخزون (أصفر) أو المنتهية (رمادي معطل).
* إمكانية الاستعلام بالباركود مع استجابة بصرية في أقل من 50ms دون إعادة تحميل الصفحة.

### 3.3 منطق العمل وسير الأحداث (Logics & Workflows)
1. **الاستعلام السريع:** البحث يتم أولاً في ذاكرة المتصفح (IndexedDB)، وإذا لم يتوفر يُطلب من السيرفر.
2. **منع البيع بالسالب (Zero / Negative Stock Control):**
   * عند إضافة المنتج للسلة أو عند الدفع، يتم فحص جدول `branch_stocks`.
   * في حال عدم توفر رصيد وكان إعداد الشركة يمنع البيع بالسالب: يتم رفض الإضافة مع تنبيه الكاشير.
3. **معالجة التزامن العالي (Concurrency & Concurrency Locking):**
   * عند خصم المخزون عبر السيرفر، يتم تنفيذ الاستعلام داخل معاملة مالية محكمة واستخدام قفل تشاؤمي لمنع السباق:
   ```php
   $stock = BranchStock::where('branch_id', $branchId)
       ->where('product_id', $productId)
       ->lockForUpdate() // يمنع أي كاشير آخر من القراءة لحين انتهاء الخصم
       ->first();
   ```

### 3.4 جداول ومخطط البيانات (Database Schema & Models)
```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    barcode VARCHAR(64) NOT NULL,
    sku VARCHAR(64) NOT NULL,
    name VARCHAR(200) NOT NULL,
    unit VARCHAR(20) DEFAULT 'PIECE', -- 'PIECE', 'BOX', 'KG'
    sale_price DECIMAL(15, 4) NOT NULL,
    tax_rate DECIMAL(5, 4) DEFAULT 0.1500, -- 15% VAT
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT uq_company_barcode UNIQUE (company_id, barcode)
);
CREATE INDEX idx_products_search ON products(company_id, barcode, sku);

CREATE TABLE branch_stocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    quantity DECIMAL(15, 4) NOT NULL DEFAULT 0.0000,
    min_alert_limit DECIMAL(15, 4) DEFAULT 5.0000,
    CONSTRAINT uq_branch_product UNIQUE (branch_id, product_id)
);
```

### 3.5 قواعد التحقق (Validation Rules)
```php
public function storeProductRules(): array {
    return [
        'barcode'    => ['required', 'string', 'max:64'],
        'sku'        => ['required', 'string', 'max:64'],
        'name'       => ['required', 'string', 'max:200'],
        'sale_price' => ['required', 'numeric', 'gt:0'],
        'tax_rate'   => ['required', 'numeric', 'in:0,0.05,0.15']
    ];
}
```

### 3.6 الأمان والحماية (Security)
* حماية الـ Barcode من محاولات إدخال الـ Payload الخبيث عبر الماسح الضوئي (Sanitizing Barcode Input لمنع أجهزة الـ Barcode Emulators من تنفيذ Shell Scripts).
* التحقق دائماً من أن المنتج تابع للـ `company_id` المحدد للمستخدم الحالي لمنع تعديل أسعار شركات أخرى.

### 3.7 اختبارات الوحدة والتكامل (Unit & Feature Tests)
* `test_pessimistic_locking_prevents_race_conditions_on_last_item()`
* `test_cannot_sell_out_of_stock_item_when_negative_stock_is_disabled()`
* `test_barcode_lookup_returns_exact_single_product()`

---

## الوحدة 4: واجهة شاشة البيع السريعة ومحرك تعليق الفواتير (POS Terminal UI & Hold Sales Engine)

### 4.1 الـ Wireframe
```text
+-------------------------------------------------------------------+
|  SMART POS | وردية: #12 | فرع الرياض | 🟢 أونلاين  | الفواتير المعلقة (2)|
+--------------------------------------+----------------------------+
|            كتالوج الأصناف            |        السلة الحالية       |
| [ بحث بالاسم أو الباركود...         ] |----------------------------|
|                                      | عصير تفاح   x 2  10.00 ر.س  |
| [تصنيف 1] [تصنيف 2] [تصنيف 3]        | حليب 1 لتر  x 1   7.50 ر.س  |
|                                      |----------------------------|
| [صنف]   [صنف]   [صنف]   [صنف]        | الإجمالي قبل الضريبة: 15.22 |
|                                      | ضريبة القيمة المضافة:  2.28 |
| [صنف]   [صنف]   [صنف]   [صنف]        | الإجمالي الشامل:   17.50 ر.س|
|                                      |----------------------------|
|                                      | [تعليق (F4)]  [إلغاء السلة]|
|                                      | [  دفع وإنهاء الفاتورة (Space) ]|
+--------------------------------------+----------------------------+
```

### 4.2 الـ UI/UX
* شاشة منقسمة (Split Screen): 65% للكتالوج السريع والباركود، و 35% لملخص السلة والإجماليات.
* مفاتيح اختصار ديناميكية تظهر على كل زر:
  * `F1`: التركيز في خانة الباركود.
  * `F4`: تعليق الفاتورة الحالية (Hold).
  * `F5`: استعراض الفواتير المعلقة واستئنافها (Resume).
  * `Space`: فتح نافذة الدفع الفوري.

### 4.3 منطق العمل وسير الأحداث (Logics & Workflows)
1. **محرك الحساب الرياضي الدقيق (Line-Item Calculations):**
   * لكل بند:
     $$\text{Line Subtotal} = (\text{Quantity} \times \text{Unit Price}) - \text{Discount}$$
     $$\text{Line Tax} = \text{Line Subtotal} \times \text{Tax Rate}$$
     $$\text{Line Total} = \text{Line Subtotal} + \text{Line Tax}$$
   * استخدام الحساب الدقيق وتفادي الفواصل العائمة عبر ضرب الأرقام وتنسيقها بدقة 4 خانات عشرية.
2. **محرك تعليق الفواتير (Hold Engine):**
   * عند الضغط على "تعليق"، يتم أخذ كائن الـ Cart بالكامل من الـ Pinia Store، وتوليد معرف محلي `hold_ref_uuid`، وحفظه في جدول محلي داخل المتصفح (IndexedDB) تحت جدول `held_carts` مع اسم مرجعي ووقت التعليق.
   * تُفرغ السلة فوراً لتكون جاهزة لخدمة العميل التالي في أقل من 100ms.
   * عند الضغط على استرجاع، تُسحب الفاتورة وتُحذف من الـ `held_carts` وتُعاد إلى الـ Active Pinia Cart.

### 4.4 نموذج البيانات المحلي (IndexedDB Schema - Dexie.js)
```typescript
// تعريف هيكل قاعدة بيانات المتصفح
export interface HeldCart {
    id: string; // UUID
    reference_name: string; // e.g., 'عميل معلق #1'
    created_at: Date;
    items: Array<{
        product_id: string;
        name: string;
        barcode: string;
        quantity: number;
        unit_price: number;
        tax_rate: number;
        discount: number;
    }>;
    notes?: string;
}
```

### 4.5 قواعد التحقق (Validation Rules)
* لا يمكن تعليق سلة فارغة (`items.length > 0`).
* الحد الأقصى للخصم لا يمكن أن يتجاوز إجمالي قيمة البند نفسه (`discount <= subtotal`).
* لا يمكن تجاوز الحد الأقصى للفواتير المعلقة لنفس الكاشير (الحد الافتراضي 15 فاتورة معلقة لمنع استهلاك ذاكرة المتصفح).

### 4.6 الأمان والحماية (Security)
* تشفير وتطهير (Sanitization) حقل "ملاحظات الفاتورة المعلقة" لمنع هجمات Cross-Site Scripting (XSS) في واجهة الـ POS.
* عزل الفواتير المعلقة لكل كاشير حتى لا يسترجع كاشير آخر سلة غير مخصصة له دون تصريح.

### 4.7 اختبارات الوحدة والتكامل (Unit & Feature Tests)
* `test_cart_totals_calculation_with_mixed_vat_rates()`
* `test_holding_a_cart_frees_current_active_cart_state()`
* `test_resuming_a_held_cart_restores_exact_item_quantities_and_discounts()`

---

## الوحدة 5: وحدة معالجة الفواتير والدفع الإلكتروني (Checkout & ZATCA E-Invoicing Engine)

### 5.1 الـ Wireframe
```text
+-------------------------------------------------------------+
|                 إنهاء عملية البيع والدفع                    |
+-------------------------------------------------------------+
| الإجمالي المطلوب: 115.00 ر.س                                |
|-------------------------------------------------------------|
| طريقة الدفع:                                                |
| [ (x) نقدي ]     [ ( ) بطاقة / مدى ]     [ ( ) آجل / عميل ] |
|                                                             |
| المبلغ المستلم من العميل: [ 150.00 ] ر.س                     |
| الباقي للعميل:            [  35.00 ] ر.س                     |
|-------------------------------------------------------------|
|           [ طباعة وإتمام الفاتورة (Enter) ]                 |
+-------------------------------------------------------------+

معاينة الإيصال الحراري (Thermal Receipt 80mm):
+------------------------------------+
|          شركة سمارت لايف            |
|         فرع الرياض - رقم 1         |
|   الرقم الضريبي: 310123456700003   |
| فاتورة ضريبية مبسطة: #INV-2026-0042|
| التاريخ: 2026-10-30 17:48:10       |
|------------------------------------|
| 1x عصير تفاح             10.00 ر.س |
|------------------------------------|
| الإجمالي الخاضع للضريبة: 8.70 ر.س   |
| ضريبة القيمة المضافة 15%: 1.30 ر.س |
| الإجمالي المستحق:      10.00 ر.س  |
| المدفوع نقداً:         10.00 ر.س  |
|------------------------------------|
|        [ ZATCA TLV QR CODE ]       |
|        [   المشفر بالكامل  ]       |
+------------------------------------+
```

### 5.2 الـ UI/UX
* بمجرد إدخال المبلغ المستلم، يقوم النظام بحساب الفكة (Change Return) لحظياً بخط عريض وأخضر واضح لتسهيل قراءته من قبل الكاشير.
* عند الضغط على Enter، يتم إرسال أمر الطباعة فوراً للـ Thermal Printer مع فتح درج النقدية تلقائياً (Cash Drawer Kick via ESC/POS).

### 5.3 منطق العمل وسير الأحداث (Logics & Workflows)
1. **معالجة الدفع:** دعم الدفع المتعدد (مثلاً 50 نقدي + 65 بطاقة مدى) بشرط أن يكون مجموع الدفعات مساوياً لإجمالي الفاتورة.
2. **محرك الفاتورة الإلكترونية السعودية (ZATCA Phase 2 TLV Engine):**
   * توليد الـ QR Code بصيغة **TLV (Tag-Length-Value)** المعتمدة، ثم تشفيرها إلى Base64:
     * **Tag 1:** اسم المورد (Company Name).
     * **Tag 2:** الرقم الضريبي للمورد (15 رقماً ويبدأ وينتهي برقم 3).
     * **Tag 3:** الطابع الزمني بتنسيق UTC ISO-8601 (`YYYY-MM-DDTHH:mm:ssZ`).
     * **Tag 4:** إجمالي الفاتورة مع الضريبة.
     * **Tag 5:** إجمالي مبلغ ضريبة القيمة المضافة.
3. **التنفيذ البرمجي لتوليد الـ TLV في الـ Backend (Laravel Service):**
```php
class ZatcaTlvEncoder {
    public static function encode(string $tag, string $value): string {
        $tagByte = chr((int)$tag);
        $lengthByte = chr(strlen($value));
        return $tagByte . $lengthByte . $value;
    }

    public static function generateQrBase64($seller, $taxNo, $timestamp, $total, $tax): string {
        $tlv = self::encode('1', $seller)
             . self::encode('2', $taxNo)
             . self::encode('3', $timestamp)
             . self::encode('4', (string)$total)
             . self::encode('5', (string)$tax);

        return base64_encode($tlv);
    }
}
```

### 5.4 جداول ومخطط البيانات (Database Schema & Models)
```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id),
    branch_id UUID NOT NULL REFERENCES branches(id),
    shift_id UUID NOT NULL REFERENCES cashier_shifts(id),
    user_id UUID NOT NULL REFERENCES users(id),
    invoice_number VARCHAR(50) NOT NULL, -- e.g., 'INV-BR1-2026-000001'
    client_uuid UUID NOT NULL, -- معرف العميل لتفادي التكرار (Idempotency)
    subtotal DECIMAL(15, 4) NOT NULL,
    tax_total DECIMAL(15, 4) NOT NULL,
    grand_total DECIMAL(15, 4) NOT NULL,
    payment_method VARCHAR(20) NOT NULL, -- 'CASH', 'CARD', 'SPLIT'
    zatca_qr_payload TEXT NOT NULL,
    is_synced BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT uq_company_invoice_no UNIQUE (company_id, invoice_number),
    CONSTRAINT uq_client_uuid UNIQUE (client_uuid)
);

CREATE TABLE invoice_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    quantity DECIMAL(15, 4) NOT NULL,
    unit_price DECIMAL(15, 4) NOT NULL,
    tax_rate DECIMAL(5, 4) NOT NULL,
    tax_amount DECIMAL(15, 4) NOT NULL,
    total_amount DECIMAL(15, 4) NOT NULL
);
```

### 5.5 قواعد التحقق (Validation Rules)
```php
public function storeInvoiceRules(): array {
    return [
        'client_uuid'     => ['required', 'uuid'],
        'items'           => ['required', 'array', 'min:1'],
        'items.*.id'      => ['required', 'uuid', 'exists:products,id'],
        'items.*.qty'     => ['required', 'numeric', 'gt:0'],
        'payment_method'  => ['required', 'in:CASH,CARD,SPLIT,CREDIT'],
        'amount_paid'     => ['required', 'numeric', 'gte:grand_total']
    ];
}
```

### 5.6 الأمان والحماية (Security)
* **Idempotency Key:** الحقل `client_uuid` يمنع تكرار تسجيل الفاتورة أو خصم المخزون مرتين في حال ضغط الكاشير مرتين متتاليتين أو في حال إعادة إرسال طلب المزامنة بعد انقطاع الشبكة.
* **ACID Transactions:** تضمن سلامة إنشاء الفاتورة، تسجيل القيود، وخصم المخزون في عملية واحدة غير قابلة للتجزئة (`DB::transaction`).

### 5.7 اختبارات الوحدة والتكامل (Unit & Feature Tests)
* `test_zatca_tlv_encoder_outputs_valid_base64_standard()`
* `test_invoice_creation_atomically_deducts_branch_inventory()`
* `test_idempotency_prevents_duplicate_invoices_with_same_client_uuid()`

---

## الوحدة 6: محرك العمل بدون إنترنت والمزامنة اللحظية (Offline-First & Background Sync Engine)

### 6.1 الـ Wireframe
```text
شريط الحالة العلوي عند انقطاع الإنترنت:
+-------------------------------------------------------------------+
|  SMART POS | [🟡 نمط غير متصل - أوفلاين (3 فواتير قيد الانتظار)]  |
+-------------------------------------------------------------------+

نافذة طابور المزامنة (Sync Monitor Drawer):
+-------------------------------------------------------------------+
| طابور المزامنة الخلفية (Offline Sync Queue)           [ مزامنة الآن ] |
|-------------------------------------------------------------------|
| • فاتورة #OFFLINE-001 | 115.00 ر.س | 17:40:02 | [🟢 تم المزامنة]  |
| • فاتورة #OFFLINE-002 |  45.00 ر.س | 17:42:15 | [🟢 تم المزامنة]  |
| • فاتورة #OFFLINE-003 | 210.00 ر.س | 17:45:30 | [🟡 بانتظار الخادم] |
|-------------------------------------------------------------------|
| حالة الاتصال: متصل مجدداً | الخادم السحابي: يستجيب (42ms)        |
+-------------------------------------------------------------------+
```

### 6.2 الـ UI/UX
* تحول شارة النظام من اللون الأخضر المضيء إلى الأصفر الفاقع تلقائياً عند انقطاع الإنترنت، مع إشعار صوتي خفيف لا يربك العميل: *"أنت تعمل الآن في وضع الأوفلاين، استمر بالبيع كالمعتاد"*.
* لا يُمنع الكاشير من البيع أو الطباعة نهائياً أثناء انقطاع الإنترنت؛ الـ QR Code يُنشأ محلياً في المتصفح عبر كود JavaScript مماثل للـ Backend تماماً.

### 6.3 منطق العمل وسير الأحداث (Logics & Workflows)
1. **العملية أثناء غياب الإنترنت (Offline Execution):**
   * عند إتمام البيع، يحفظ تطبيق المتصفح الفاتورة في جدول `offline_invoices` بـ IndexedDB.
   * تأخذ الفاتورة رقماً تسلسلياً محلياً: `OFFLINE-{terminal_id}-{timestamp}`.
   * تُطبع الفاتورة بإيصالها الحراري والـ QR Code محلياً.
2. **استشعار عودة الشبكة (Network Reconnect Listener):**
   * تطبيق الـ Vue.js يستمع للحدث `window.addEventListener('online')` ويرسل طلب Heartbeat فوري إلى `/api/v1/health`.
3. **محرك الرفع التجميعي (Bulk Sync Engine):**
   * يرسل المتصفح مصفوفة الفواتير دفعة واحدة إلى مسار:  
     `POST /api/v1/pos/sync/bulk`
   * خادم Laravel يفتح `DB::beginTransaction()` ويعالج كل فاتورة:
     * إذا كان الـ `client_uuid` مسجلاً مسبقاً، يتخطاه (Idempotent).
     * يخصم الكميات من `branch_stocks`.
     * يُنشئ سجل الفاتورة النهائي.
   * يعيد الخادم قائمة المعرفات المكتملة `synced_ids`.
   * يقوم المتصفح بتحديث حالة الفواتير في IndexedDB إلى `SYNCED` لحذفها أو أرشفتها.

### 6.4 مخطط الـ IndexedDB للـ Sync Engine (Front-End)
```typescript
import Dexie, { Table } from 'dexie';

export interface OfflineInvoice {
    client_uuid: string;
    shift_id: string;
    terminal_id: string;
    items: Array<any>;
    payment_method: string;
    grand_total: number;
    tax_total: number;
    zatca_qr_payload: string;
    created_at: string;
    sync_status: 'PENDING' | 'SYNCED' | 'FAILED';
}

export class PosDatabase extends Dexie {
    offline_invoices!: Table<OfflineInvoice, string>;

    constructor() {
        super('SmartPosOfflineDB');
        this.version(1).stores({
            offline_invoices: 'client_uuid, sync_status, created_at'
        });
    }
}

export const db = new PosDatabase();
```

### 6.5 كود معالجة التزامن على السيرفر (Bulk Sync Controller)
```php
public function bulkSync(Request $request): JsonResponse {
    $invoices = $request->input('invoices', []);
    $syncedIds = [];

    DB::transaction(function () use ($invoices, &$syncedIds) {
        foreach ($invoices as $invData) {
            // التحقق من عدم تسجيلها مسبقاً لمنع التكرار
            if (Invoice::where('client_uuid', $invData['client_uuid'])->exists()) {
                $syncedIds[] = $invData['client_uuid'];
                continue;
            }

            // إنشاء الفاتورة وخصم المخزون
            $invoice = InvoiceService::createFromSync($invData);
            $syncedIds[] = $invData['client_uuid'];
        }
    });

    return response()->json([
        'status' => 'success',
        'synced_client_uuids' => $syncedIds
    ]);
}
```

### 6.6 الأمان والحماية (Security)
* **حماية التلاعب الزمني (Timestamp Validation):** فحص تواريخ الفواتير المرفوعة أوفلاين لمنع تزوير تاريخ الفاتورة بما يخالف القوانين الضريبية (رفض الفواتير التي يزيد فارق وقتها عن أسبوع دون تصريح مدير).
* **Payload Verification:** توقيع تشفيري (HMAC) لكل دفعة فواتير مرفوعة من المتصفح لضمان عدم اعتراض البيانات والتلاعب بالأسعار أثناء انقطاع الاتصال.

### 6.7 اختبارات الوحدة والتكامل (Unit & Feature Tests)
* `test_bulk_sync_successfully_processes_offline_invoices_batch()`
* `test_bulk_sync_is_strictly_idempotent_when_retried()`
* `test_database_rolls_back_if_any_single_invoice_fails_in_bulk_transaction()`

---


