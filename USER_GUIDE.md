# دليل استخدام وتشغيل واختبار مشروع Order Gateway (WSO2 Micro Integrator 4.6)
# Project Delivery & User Guide

هذا الدليل مخصص لتسليم المشروع، ويشرح بالتفصيل خطوات التثبيت، الإعداد، التشغيل، والاختبار لتقييم المشروع أو تسليمه للجنة الاختبار.

---

## 📌 1. نبذة عن المشروع (Project Overview)
المشروع عبارة عن **Order Gateway** مبني بواسطة **WSO2 Micro Integrator (MI) 4.6**، يقوم بتوحيد واستيعاب 3 أنظمة خلفية مختلفة خلف واجهة برمجية واحدة (REST API):
1. **REST Backend (DummyJSON)**: جلب بيانات المنتجات (`https://dummyjson.com/products/{id}`).
2. **SOAP Web Service (Calculator)**: حساب إجمالي الطلب بعملية الضرب `Multiply` عبر بروتوكول SOAP 1.1 (`http://www.dneonline.com/calculator.asmx`).
3. **Data Service (WSO2 DSS + MySQL)**: تخزين واسترجاع بيانات الطلبات في جدول `TrainingDB.orders`.

---

## 💻 2. المتطلبات الأساسية (Prerequisites)
قبل تشغيل المشروع، تأكد من توفر الآتي:
- **نظام التشغيل**: Windows / Linux / macOS.
- **Java**: JDK 11 أو 17 أو 21.
- **WSO2 Micro Integrator**: الإصدار 4.6.0.
- **قاعدة بيانات MySQL**: تعمل على المنفذ `3306` (أو حاوية Docker):
  - المستخدم: `youruser`
  - كلمة المرور: `yourpassword`
  - منفذ الاتصال: `yourport -default 3306`s
- **برنامج Postman** (اختياري للاختبار عبر الواجهة).

---

## 🗄️ 3. إعداد قاعدة البيانات (Database Setup)

1. تأكد من أن حاوية MySQL تعمل على المنفذ `3306`.
2. قم بتنفيذ ملف السكربت الموجود داخل مجلد المشروع:
   📁 المسار: `OrderGateway/database/schema.sql`

أو تشغيل الأوامر التالية مباشرة داخل MySQL:
```sql
CREATE DATABASE IF NOT EXISTS TrainingDB;
USE TrainingDB;

CREATE TABLE IF NOT EXISTS orders (
    id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    quantity INT NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'NEW',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## ⚙️ 4. محتويات وهيكل مجلد المشروع (Project Structure)

```text
OrderGateway/
├── .vscode/                               # إعدادات بيئة العمل لـ VS Code WSO2 Extension
│   └── settings.json
├── database/                              # سكربت قاعدة البيانات
│   └── schema.sql
├── deployment/                            # إعدادات النشر ومكتبات التعريف
│   ├── deployment.toml
│   └── libs/
│       └── mysql-connector-j-8.0.31.jar   # تعريف MySQL JDBC Driver
├── postman/                               # مجموعة الاختبارات الجاهزة لبرنامج Postman
│   └── OrderGateway.postman_collection.json
├── src/                                   # السورس كود البرمجي لمكونات WSO2 MI
│   └── main/wso2mi/artifacts/
│       ├── apis/                          # OrderGatewayAPI.xml (الواجهة والتنسيق الرئيسي)
│       ├── data-services/                 # OrderDataService.dbs (خدمة البيانات)
│       ├── endpoints/                     # نقاط الاتصال (Product, Calculator, DataService)
│       └── sequences/                     # معالجة الأخطاء (Fault Sequences)
├── pom.xml                                # ملف Maven لبناء تطبيق الـ CAR
├── mvnw / mvnw.cmd                        # Maven Wrapper الجاهز للتشغيل
├── README.md                              # التوثيق التقني للمشروع
└── USER_GUIDE.md                          # دليل التسليم والاستخدام
```

---

## 🚀 5. خطوات البناء والتشغيل (Build & Run)

### الخيار (أ): البناء والنشر التلقائي عبر Maven Wrapper
افتح موجه الأوامر (PowerShell أو CMD) داخل مجلد `OrderGateway` ونفذ الأمر:
```powershell
.\mvnw.cmd compile "-Dmaven.test.skip=true"
```
سيتم إنشاء ملف الأرشيف المركب:
`target/OrderGateway_1.0.0.car`

ثم قم بنسخه إلى مجلد نشر WSO2 MI:
`<MI_HOME>/repository/deployment/server/carbonapps/`

> **ملاحظة هامة:** تأكد من وجود ملف `mysql-connector-j-8.0.31.jar` داخل مجلد `<MI_HOME>/lib/` أيضاً لضمان اتصال الـ Data Service بـ MySQL بنجاح.

---

### الخيار (ب): التشغيل عبر VS Code (WSO2 Micro Integrator Extension)
1. افتح مجلد `OrderGateway` داخل **VS Code**.
2. اضغط على أيقونة إضافة **WSO2 Micro Integrator** في الشريط الجانبي.
3. اضغط على **Build** لإنشاء حزمة الـ CAR ثم **Deploy/Run**.
4. سيبدأ سيرفر WSO2 MI بالاستماع على المنفذ الافتراضي `8290`.

---

## 🧪 6. دليل الاختبار الشامل (Testing Guide)

### أولاً: عبر برنامج Postman
1. افتح برنامج **Postman**.
2. اضغط على **Import** واختر الملف:
   `OrderGateway/postman/OrderGateway.postman_collection.json`
3. ستظهر لك مجموعة تحتوي على 9 طلبات تغطي كافة الحالات بنقرة زر:
   - `TC01 - Create Order Success (200 OK)`
   - `TC02 - Create Order Dynamic Values (200 OK)`
   - `TC03 - Missing Quantity (400 Bad Request)`
   - `TC04 - Invalid Quantity Zero (400 Bad Request)`
   - `TC05 - Product Not Found (404 Not Found)`
   - `TC06 - Standalone Product Lookup (200 OK)`
   - `TC07 - Standalone Product Lookup (404 Not Found)`
   - `TC08 - Get Order by ID (Challenge 1)`
   - `TC09 - Get All Orders (Challenge 2)`

---

### ثانياً: أوامر الاختبار السريعة عبر Terminal / PowerShell

#### 1. سيناريو إنشاء طلب ناجح (Order 5001):
```powershell
curl -X POST "http://localhost:8290/order-gateway/orders" `
  -H "Content-Type: application/json" `
  -H "X-Request-ID: req-demo-5001" `
  -d '{"orderId": 5001, "customerId": 1001, "productId": 10, "quantity": 2}'
```
**الرد المتوقع (HTTP 200 OK):**
```json
{
  "orderId": 5001,
  "customerId": 1001,
  "product": {
    "id": 10,
    "name": "Gucci Bloom Eau de",
    "unitPrice": 79.99
  },
  "quantity": 2,
  "totalAmount": 160,
  "status": "NEW"
}
```

#### 2. سيناريو اختبار رفض المدخلات الناقصة (400 Bad Request):
```powershell
curl -X POST "http://localhost:8290/order-gateway/orders" `
  -H "Content-Type: application/json" `
  -d '{"orderId": 5003, "customerId": 1001, "productId": 10}'
```
**الرد المتوقع (HTTP 400 Bad Request):**
```json
{
  "status": "ERROR",
  "errorCode": 400,
  "errorMessage": "Bad Request: Missing or invalid required fields (orderId, customerId, productId, quantity > 0)",
  "errorDetail": "Internal error",
  "requestId": "req-..."
}
```

#### 3. سيناريو اختبار منتج غير موجود (404 Not Found):
```powershell
curl -X POST "http://localhost:8290/order-gateway/orders" `
  -H "Content-Type: application/json" `
  -d '{"orderId": 5005, "customerId": 1001, "productId": 999999, "quantity": 1}'
```
**الرد المتوقع (HTTP 404 Not Found):**
```json
{
  "status": "ERROR",
  "errorCode": 404,
  "errorMessage": "Product Not Found: The requested productId does not exist in product catalog",
  "errorDetail": "Internal error",
  "requestId": "req-..."
}
```

---

## 🔍 7. فحص التخزين في قاعدة البيانات
بعد إرسال الطلبات، يمكنك التحقق من حفظها مباشرة:
```sql
USE TrainingDB;
SELECT * FROM orders;
```
ستظهر السجلات مع الكميات والمبالغ المحسوبة وتاريخ الإنشاء.