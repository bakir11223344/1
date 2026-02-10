# تصميم نظام ويب محلي لإدارة وتوليد الملفات الرسمية داخل المكتب

## 1) الـ Tech Stack المقترح (عملي ومستقر للاستخدام اليومي)

### الخيار الموصى به (MVP + إنتاج فعلي)
- **Backend:** Python + FastAPI
- **App Server:** Uvicorn (خلف Nginx داخلي)
- **Frontend:** HTML + Jinja2 Templates + Bootstrap (واجهة بسيطة جدًا)
- **Database:** SQLite كبداية (مع قابلية الترقية إلى MySQL/MariaDB)
- **Auth/Sessions:** Cookie-based Session + صلاحيات Role-Based Access Control (RBAC)
- **Document Generation:**
  - Word: `python-docx`
  - Excel: `openpyxl`
  - PDF: مسار منظم (HTML/CSS → PDF عبر WeasyPrint أو LibreOffice headless للتحويل من DOCX)
- **Local Deployment:** Docker Compose داخل الشبكة المحلية فقط (بدون إنترنت)
- **Backups:** نسخ احتياطي محلي مجدول + تشفير محلي للنسخ

### لماذا هذا الاختيار؟
1. **سهولة التشغيل والصيانة:** Python/FastAPI سهلان للفِرق الصغيرة وغير المعقدة.
2. **استقرار عالٍ:** SQLite موثوق جدًا لتشغيل مكتب صغير إلى متوسط (مع قفل كتابة جيد وإعدادات WAL).
3. **واجهة بسيطة:** Jinja + Bootstrap يسرّع الإنتاج ويقلل الأعطال مقارنة بإطارات Frontend الثقيلة.
4. **إنتاج مستندات احترافي:** مكتبات Python ناضجة وتدعم التحكم بالتنسيق.
5. **مرونة مستقبلية:** نفس الطبقات قابلة للتوسع لاحقًا (MySQL + Queue + Storage أكبر).

---

## 2) معمارية النظام (System Architecture)

## 2.1 نظرة عامة
- المستخدم داخل LAN يفتح النظام من المتصفح عبر IP محلي (مثال: `http://192.168.1.20`).
- الطلب يمر عبر Nginx داخلي.
- Nginx يرسل الطلب لتطبيق FastAPI.
- FastAPI يتعامل مع:
  - التوثيق والصلاحيات
  - إدارة القوالب
  - إدخال البيانات
  - توليد الملفات
  - الأرشفة والسجلات
- الملفات المولدة تحفظ في File Storage محلي (مجلدات منظمة + checksum).
- البيانات الوصفية والسجلات تحفظ في DB.

## 2.2 الطبقات
1. **Presentation Layer**
   - صفحات بسيطة: تسجيل دخول، لوحة، إنشاء مستند، أرشيف، إدارة مستخدمين.
2. **Application Layer**
   - منطق الأعمال: validation، role checks، generation workflow.
3. **Template Engine Layer**
   - إدارة القوالب، حقول ثابتة/متغيرة، قواعد التنسيق.
4. **Document Generation Layer**
   - مولد DOCX/Excel/PDF مع قواعد ثابتة.
5. **Persistence Layer**
   - SQLite/MySQL + Local File Storage.
6. **Audit & Security Layer**
   - logs، login attempts، action trails، IP checks.

## 2.3 مخطط تدفق مختصر
1. مستخدم يسجل الدخول.
2. يختار نوع قالب (كتاب رسمي/تعهد/فاتورة...).
3. يدخل القيم في الحقول المتغيرة.
4. النظام يطبق template + style locks.
5. ينتج DOCX/PDF/XLSX.
6. يحفظ نسخة مؤرشفة + metadata + audit log.
7. يمكن إعادة التصدير لاحقًا بنفس البيانات أو نسخة جديدة بإصدار جديد.

---

## 3) تصميم قاعدة البيانات (الجداول والعلاقات)

## 3.1 الجداول الأساسية

### `users`
- `id` (PK)
- `username` (Unique)
- `password_hash`
- `full_name`
- `role_id` (FK -> roles.id)
- `is_active`
- `last_login_at`
- `created_at`

### `roles`
- `id` (PK)
- `name` (`owner`, `staff`, `viewer`)

### `permissions`
- `id` (PK)
- `code` (`create_doc`, `export_doc`, `manage_users`, ...)

### `role_permissions`
- `role_id` (FK)
- `permission_id` (FK)
- (PK مركب)

### `clients`
- `id` (PK)
- `type` (`office_customer`, `student`)
- `name`
- `phone` (اختياري)
- `identifier` (رقم جامعي/هوية...)
- `notes`
- `created_at`

### `template_categories`
- `id` (PK)
- `name` (كتب رسمية، تعهدات، ...)

### `templates`
- `id` (PK)
- `category_id` (FK)
- `name`
- `version`
- `status` (`active`, `archived`)
- `engine` (`docx`, `xlsx`, `html-pdf`)
- `style_profile_id` (FK)
- `file_path` (مسار القالب الأساسي)
- `created_by`
- `created_at`

### `template_fields`
- `id` (PK)
- `template_id` (FK)
- `field_key` (مثل `customer_name`)
- `field_label`
- `field_type` (`text`, `date`, `number`, `select`)
- `is_required`
- `validation_regex`
- `sort_order`

### `style_profiles`
- `id` (PK)
- `name`
- `font_family`
- `font_size`
- `line_spacing`
- `margins_json`
- `alignment_rules_json`
- `locked` (true)

### `documents`
- `id` (PK)
- `template_id` (FK)
- `client_id` (FK)
- `created_by` (FK -> users)
- `serial_number`
- `status` (`generated`, `re-exported`, `void`)
- `payload_json` (قيم الحقول عند الإنشاء)
- `created_at`

### `document_files`
- `id` (PK)
- `document_id` (FK)
- `file_type` (`docx`, `pdf`, `xlsx`)
- `file_path`
- `file_hash_sha256`
- `generated_at`

### `audit_logs`
- `id` (PK)
- `user_id` (FK, nullable للنظام)
- `action` (`login`, `create_document`, `export_pdf`, ...)
- `target_type` (`document`, `template`, `user`)
- `target_id`
- `ip_address`
- `details_json`
- `created_at`

### `sessions`
- `id` (PK)
- `user_id` (FK)
- `session_token_hash`
- `ip_address`
- `user_agent`
- `expires_at`
- `created_at`

## 3.2 العلاقات المهمة
- User -> Role (Many to One)
- Role -> Permissions (Many to Many)
- Template -> TemplateFields (One to Many)
- Template -> Documents (One to Many)
- Client -> Documents (One to Many)
- Document -> DocumentFiles (One to Many)
- User -> AuditLogs (One to Many)

---

## 4) تصميم نظام القوالب والتحكم بالتنسيق

## 4.1 فلسفة القوالب
- القالب عبارة عن **هيكل ثابت + متغيرات Placeholder**.
- الحقول الثابتة (نصوص رسمية، عناوين، شعارات) لا تظهر للمستخدم للتعديل.
- الحقول المتغيرة فقط تظهر داخل فورم إدخال.

## 4.2 تعريف القالب
لكل قالب 3 عناصر:
1. **ملف تصميم أساس** (DOCX أو XLSX أو HTML)
2. **Schema JSON** يعرّف الحقول المتغيرة وقواعد التحقق
3. **Style Profile** ثابت (خط/حجم/هوامش/محاذاة)

## 4.3 قفل النمط (Style Locking)
- منع حفظ أي قالب بدون Style Profile صالح.
- أثناء التوليد، يتم تعيين الخطوط والمحاذاة برمجيًا قبل الحفظ.
- رفض أي قيمة تتجاوز طولًا معينًا قد يسبب كسر تنسيق.
- إضافة auto-fit controlled في الجداول، مع حدود دنيا/قصوى للحقول.

## 4.4 مثال placeholders
- `{{customer_name}}`
- `{{date_hijri}}`
- `{{reference_number}}`
- `{{destination_entity}}`

---

## 5) توليد Word / PDF / Excel بدون كسر النمط

## 5.1 Word (.docx)
- استخدام ملف DOCX مع علامات مميزة للحقول.
- حقن القيم في runs بحذر (لتفادي كسر تنسيق النص المجزأ).
- إعادة فرض font/alignment/spacing بعد الاستبدال.
- التحقق من الطول قبل الحقن، مع قص ذكي أو رفض إدخال غير صالح.

## 5.2 PDF
مساران موصى بهما:
1. **HTML+CSS → PDF (WeasyPrint)** لصفحات قياسية سهلة الضبط.
2. **DOCX → PDF (LibreOffice headless)** عندما يلزم تطابق شبه تام مع نموذج Word.

مبدأ مهم: اختيار محرك واحد لكل نوع قالب لضمان الثبات (لا تبديل عشوائي).

## 5.3 Excel (.xlsx)
- قوالب XLSX جاهزة مع خلايا placeholders.
- تعبئة القيم عبر openpyxl مع الحفاظ على styles, borders, merged cells.
- منع تعديل بنية الورقة للمستخدم النهائي.

## 5.4 ضمان الجودة قبل الإنتاج
- Snapshot Tests للقوالب الحرجة (مقارنة مخرجات مع baseline).
- فحوص تلقائية: الهوامش، عدد الصفحات، الخط الافتراضي، وجود الحقول الإلزامية.
- إصدار version لكل قالب وتجميد القديم للأرشيف.

---

## 6) نموذج أمان قوي داخل LAN

1. **Bind داخلي فقط:** الخدمة تستمع على IP LAN فقط.
2. **Firewall داخلي:** السماح فقط لنطاق الشبكة المحلي (مثلاً 192.168.1.0/24).
3. **منع الإنترنت:** لا تحديثات ولا APIs خارجية من التطبيق.
4. **جلسات آمنة:**
   - HttpOnly cookies
   - session expiry قصيرة + تمديد مشروط
5. **كلمات مرور:** hash قوي (Argon2/Bcrypt).
6. **تسجيل شامل:** كل دخول/تعديل/تصدير في audit logs.
7. **صلاحيات صارمة:** أقل صلاحية ممكنة لكل دور.
8. **نسخ احتياطي محلي مشفر:** يومي + اختبار استرجاع دوري.
9. **قفل حساب بعد محاولات فاشلة متعددة.**
10. **توقيع checksums للملفات المولدة** لكشف أي عبث.

---

## 7) MVP مقترح (نسخة أولى عملية)

## 7.1 ما يجب أن يدخل في MVP فقط
1. تسجيل دخول + 3 أدوار (Owner/Staff/Viewer).
2. إدارة عملاء بسيطة (إضافة/بحث).
3. 6 قوالب أساسية:
   - كتاب رسمي
   - تعهد
   - طلب
   - قائمة أسماء
   - فاتورة
   - نموذج طلابي
4. توليد DOCX + PDF (Excel يمكن تأجيله إذا ضاق الوقت).
5. أرشفة كل مستند مع إعادة التصدير.
6. سجل عمليات (Audit Log) قابل للبحث.
7. نسخ احتياطي يدوي + تلقائي يومي.

## 7.2 ما يؤجل بعد MVP
- محرر قوالب متقدم داخل الواجهة.
- Workflow موافقات متعدد المراحل.
- تقارير تحليلية متقدمة.
- دعم OCR/باركود.

---

## 8) خطة تحديثات لاحقة (Roadmap)

### الإصدار 1.1
- دعم Excel بالكامل.
- تحسين البحث داخل الأرشيف (Full-text).
- لوحة مراقبة نشاط يومي.

### الإصدار 1.2
- إدارة إصدارات القوالب مع مقارنة فروقات.
- توقيع إلكتروني داخلي للمستندات.
- سياسات حفظ وأرشفة تلقائية حسب المدة.

### الإصدار 2.0
- نسخة طلابية محدودة بواجهات أبسط.
- مزامنة بين عدة فروع داخل نفس الشبكة الخاصة (بدون إنترنت).
- محرك قواعد أعمال متقدم للنماذج.

---

## 9) أخطاء شائعة يجب تجنبها

1. **السماح بتعديل التنسيق يدويًا** بعد التوليد (يكسر الهوية الرسمية).
2. **عدم فصل البيانات عن القالب** (يصعب الصيانة).
3. **تعدد محركات PDF لنفس القالب** (نتائج غير ثابتة).
4. **غياب audit log مفصل** (يصعب التتبع والمحاسبة).
5. **صلاحيات فضفاضة للموظفين.**
6. **الاعتماد على جهاز واحد بدون نسخ احتياطي.**
7. **استخدام SQLite دون إعداد WAL/Backup مناسب** في بيئة كثيفة.
8. **إهمال اختبار الطباعة الفعلية** (مخرجات الشاشة لا تكفي).

---

## 10) نصائح عملية لجعل النظام معتمدًا عليه يوميًا

1. **بسّط الشاشات:** كل عملية بحد أقصى 3 خطوات.
2. **استخدم Defaults ذكية:** التاريخ الحالي، الجهة الافتراضية، أرقام تسلسلية تلقائية.
3. **تحقق إدخال قوي:** منع القيم الفارغة/غير المنطقية قبل التوليد.
4. **اختصارات لوحة مفاتيح** للموظفين ذوي الاستخدام العالي.
5. **قوالب قليلة لكن متقنة:** ابدأ بالأكثر استخدامًا أولًا.
6. **دليل تشغيل صفحة واحدة** مطبوع بجانب كل جهاز.
7. **خطة طوارئ:** إذا تعطّل جهاز الخادم، جهاز بديل + استعادة سريعة.
8. **مراقبة دورية:** سعة التخزين، زمن التوليد، أخطاء الدخول.
9. **تدريب قصير للموظفين** على الأخطاء المتكررة.
10. **سياسة تغيير منضبطة:** أي تعديل قالب يجب أن يمر ببيئة اختبار قبل الإنتاج.

---

## 11) توصية تشغيل فعلية في المكتب

- جهاز صغير مخصص كخادم محلي (Mini PC/Server) + UPS.
- نظام Linux مستقر (Ubuntu LTS).
- تشغيل التطبيق عبر Docker Compose:
  - `app`
  - `nginx`
  - `backup`
- مشاركة مجلد الأرشيف على NAS داخلي (اختياري).
- جدول نسخ احتياطي:
  - يومي incremental
  - أسبوعي full
  - اختبار استرجاع شهري إلزامي

بهذا التصميم تحصل على نظام محلي ثابت وآمن وسهل، مناسب للاعتماد اليومي الكثيف في المكاتب، مع قابلية توسع منظمة لنسخة الطلاب لاحقًا.
