# TTCRM Dynamic RBAC Full Control V2 — Implementation Prompt

## الهدف
تنفيذ نظام صلاحيات Dynamic RBAC فعلي داخل TTCRM بحيث يستطيع **Admin / Super Admin** التحكم في الصلاحيات حسب الـ Role من شاشة **الإعدادات > الأدوار والصلاحيات**، ويؤثر التغيير فعليًا على:

1. ظهور وإخفاء الأقسام في الـ Sidebar.
2. فتح/منع Routes في الـ Frontend.
3. السماح/المنع في tRPC Backend APIs.
4. صلاحيات Create / Edit / Delete / Manage / Approve / Assign/Act.
5. استمرار قواعد Ownership / Assignments الحالية في الـ Backend وعدم توسيع نطاق البيانات تلقائيًا.

## قاعدة التنفيذ
المشروع الأساسي: `mohamedamouseo-a11y/TTCRM`

لا تعمل Push على `master`.
أنشئ Branch جديد باسم:

`feature/dynamic-rbac-full-control-v2`

### المرحلة 1 — تطبيق Dynamic RBAC الأساسي
الـ `master` الحالي لا يحتوي تغييرات Dynamic RBAC الموجودة سابقًا في:

`feature/dynamic-rbac-role-permissions`

هذا الـ branch مبني مباشرة فوق الـ master الحالي ويحتوي 8 commits خاصة بالـ Dynamic RBAC. ادمج/Cherry-pick هذه التغييرات أولًا داخل الـ branch الجديد فقط، بدون لمس master.

المطلوب بعد المرحلة الأولى أن تكون الملفات التالية موجودة/محدثة:

- `server/services/rolePermissionsService.ts`
- `scripts/apply-dynamic-rbac-role-permissions-v1.ts`
- `server/_core/systemRouter.ts`
- `server/_core/trpc.ts`
- `client/src/lib/roles.ts`
- `client/src/_core/hooks/useAuth.ts`
- `client/src/components/RolePermissionsStudio.tsx`

لا تجعل Admin قابلًا لفقد صلاحياته؛ Admin/Super Admin يظل Full Access كـ break-glass role.

## المرحلة 2 — إزالة قيود الـ Sidebar المبنية على أسماء Roles
في `client/src/components/CRMLayout.tsx` لا يكفي وجود `hasRolePermission` إذا بقيت فلاتر ثابتة مثل:

- `roles.includes(role)`
- `role !== "Viewer"`
- `["Admin", "SalesManager", ...].includes(role)`
- `isBdOnly` / `BD_ALLOWED_HREFS`

حوّل عرض الأقسام الرئيسية إلى Permission-first behavior:

- Dashboard => `dashboard.view`
- Inbox => `inbox.view`
- Chat => `chat.view`
- Sales group => `sales.view`
- Sales reports/team dashboard => `sales.reports`
- Lead distribution => `sales.assign`
- Finance => `finance.view`
- Account Management => `accounts.view`
- Business Development => `businessDevelopment.view`
- Marketing => `marketing.view`
- Email Marketing => `emailMarketing.view`
- WhatsApp => `whatsapp.view`
- WhatsApp Accounts/Settings => `whatsapp.manage`
- Settings => `settings.view`
- Users => `users.view`
- Audit => `audit.view`
- Trash => `trash.view`
- Support => `support.view`

إذا كان للعنصر `permission` استخدم permission فقط لتقرير الظهور، ولا تمنعه مرة ثانية بسبب role array قديمة.

لا تستخدم special-case يمنع BusinessDeveloper أو AccountManager من أقسام تم منحهم permission لها يدويًا.

## المرحلة 3 — Routes
في `client/src/App.tsx`:

- اجعل `/` محميًا بـ `dashboard.view` بدل Route غير محمي.
- `/lead-distribution` يجب أن يستخدم `sales.assign`.
- `/support-center` و `/support-center/:id` يجب أن يستخدما `support.view`.
- اترك Help Center وCSAT والـ public BD portal public كما هي.

أي Route داخلي له Permission يجب أن يذهب إلى `/access-denied` عند عدم توفرها.

## المرحلة 4 — Backend Dynamic Gate
النسخة الأساسية تجعل `permissionProcedure()` ديناميكية، لكن يوجد عدد كبير من APIs يستخدم `protectedProcedure` أو Role guards قديمة.

أضف في `server/services/rolePermissionsService.ts` دالة pure mapping باسم مثل:

`getRequiredPermissionForProcedure(path, type)`

تعمل كـ coarse module gate على الأقل للـ namespaces التالية:

- `leads`, `activities`, `deals`, `campaigns`, `pipelineStages`, `customFields` => sales
- `salesReports` => `sales.reports`
- `accountManagement`, `dynamicWorkflow`, renewals-related APIs => accounts
- `finance` => finance
- `inbox` => inbox
- `aiSocialAgent`, Meta/TikTok/Google/Snapchat/LinkedIn routers => marketing
- `emailMarketing` => emailMarketing
- `waGateway`, `wapilot`, Tara/WA AI routers => whatsapp
- `trash` => trash
- `auditLogs` => audit
- `users`, `teams` => users
- support routers => support
- settings/configuration routers => settings

Queries تحتاج View permission. Mutations تحتاج أنسب write permission المتاح للوحدة (Edit/Manage/Reply/Restore حسب الوحدة). إذا كان الإجراء متخصصًا وله `permissionProcedure` أصلًا، تظل الصلاحية الأكثر تحديدًا هي الحاكمة.

استدعِ هذا الـ gate داخل `requireUser` في `server/_core/trpc.ts` بعد التأكد من وجود المستخدم وقبل `next()`.

**مهم:** `system.myPermissions`, `system.rolePermissions`, auth/system health لا يجب أن تتعطل بسبب module mapping.

## المرحلة 5 — إزالة Role Guards التي تتعارض مع Dynamic Grants
في `server/routers.ts` عدّل الحراس العامة التالية بحيث تعتمد على Permission بدل قائمة role ثابتة:

- `salesReadProcedure` => `permissionProcedure("sales.view")`
- `salesEditProcedure` => `permissionProcedure("sales.edit")`
- `clientOpsProcedure` => `permissionProcedure("accounts.view")` مع إبقاء checks الخاصة بملكية Client داخل العمليات نفسها.
- `marketingCenterProcedure` => `permissionProcedure("marketing.view")`
- `aiSocialReadProcedure` => `permissionProcedure("marketing.view")`
- `aiSocialSetupProcedure` => `permissionProcedure("marketing.manage")`
- `waAiConfigProcedure` => `permissionProcedure("whatsapp.manage")`
- `waAiManageProcedure` => `permissionProcedure("whatsapp.manage")`
- `waAiActProcedure` => `permissionProcedure("whatsapp.reply")`

لا تحذف checks الخاصة بـ ownership أو assignment أو account/session isolation الخاصة بواتساب والعملاء؛ هذه Data Scope وليست Module Permission.

## المرحلة 6 — شاشة الصلاحيات
احتفظ بالـ UI الموجودة في `RolePermissionsStudio.tsx` بالشكل القريب من الـ reference:

- Cards للأدوار أفقيًا.
- Permission Matrix واضحة.
- أول عمود `ظهور القسم`.
- Create / Edit / Delete / Manage / Approve / Assign/Act.
- عند إغلاق `ظهور القسم` يتم إغلاق permissions التابعة للقسم تلقائيًا.
- عند تفعيل Action فرعية يتم تفعيل ظهور القسم تلقائيًا.
- زر `حفظ الصلاحيات`.
- زر `استعادة الافتراضي`.
- تعيين Role للمستخدم من نفس الصفحة.
- Admin يظهر Locked / Full Access.
- إظهار عدد الصلاحيات الممنوحة والمحجوبة.

## Data Scope
لا تجعل شاشة الصلاحيات تغير Ownership / Assignments في هذه المرحلة.

يعني إذا أعطينا `AccountManager` صلاحية `sales.view` فهذا يسمح بفتح قسم المبيعات، لكن أي Query فيها ownership scope حالي يجب أن تظل تحترم ownership المطبق بالفعل.

اكتب ملاحظة واضحة في الشاشة أن Data Scope يُدار بقواعد Backend الحالية.

## Migration
نفّذ أولًا Dry Run:

`npx tsx scripts/apply-dynamic-rbac-role-permissions-v1.ts`

ثم عند نجاحه:

`npx tsx scripts/apply-dynamic-rbac-role-permissions-v1.ts --apply`

يجب ألا تحذف أو تعدل بيانات موجودة.

## الاختبارات المطلوبة
نفّذ:

`npm run check`

`npm test -- --runInBand` إذا كان supported، وإلا `npm test`.

`npm run build`

اختبر يدويًا على الأقل:

1. إغلاق `marketing.view` من SalesManager => Marketing يختفي والـ route/API ترجع Forbidden.
2. إعادة فتحها => القسم يعود.
3. منح `accounts.view` لدور لا يملكه افتراضيًا => مجموعة الحسابات تظهر، مع بقاء ownership rules.
4. إغلاق `whatsapp.view` => مجموعة واتساب تختفي ولا تفتح مباشرة بالرابط.
5. `Admin` لا يمكن فقد Full Access.
6. مستخدم غير Admin لا يستطيع استدعاء APIs تعديل role permissions.
7. بعد logout/login تظل الصلاحيات محفوظة من قاعدة البيانات.
8. Restore Defaults يعيد `ROLE_PERMISSIONS` الأصلية للدور.

## Git Safety
- لا تعمل Push أو Merge إلى `master`.
- كل التعديلات على `feature/dynamic-rbac-full-control-v2` فقط.
- في النهاية اعرض diff والملفات المتغيرة ونتائج check/build.
- لا تعمل PR merge تلقائيًا.
