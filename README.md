# پنل مدیریت گروه — مستقل از سورس ربات

این پنل کاملاً جدا از پروژه‌ی فعلی ربات (`server.ts` و بقیه) ساخته شده و **هیچ خطی از سورس ربات تغییر نمی‌کند**.
ارتباط فقط از طریق یک API مستقل (Google Apps Script) با دیتابیس واقعی (Firestore) برقرار می‌شود.

> این فایل را کامل بخوانید — چند محدودیت واقعی مهم اینجا مستند شده که مستقیماً روی نحوه‌ی استفاده از پنل اثر می‌گذارد.

---

## ۱. معماری

```
مرورگر کاربر
   │
   ▼
GitHub Pages  (frontend/  → HTML + CSS + Vanilla JS، بدون فریم‌ورک)
   │  fetch("https://script.google.com/macros/s/XXXX/exec", …)
   ▼
Google Apps Script Web App  (apps-script/  → Backend/API)
   │  Firestore REST API v1  (با Service Account، بدون دخالت سرور Node)
   ▼
همان Firestore واقعی ربات
   پروژه: gen-lang-client-0954132739
   Database ID سفارشی: ai-studio-orginbotaitelegr-86c949e5-d1c1-477d-a1fe-85e98a5e12cb
   Collections مرتبط: groups, users
```

Apps Script از طریق **Service Account** مستقیماً با Firestore REST API صحبت می‌کند — دقیقاً مثل یک برنامه‌ی سرور جدا،
بدون اینکه به فرآیند Node شما وصل شود یا فایلی از آن را بخواند/تغییر دهد.

### چرا این‌طور و نه از طریق سرور Node؟
چون گفتید سورس فعلی نباید تغییر کند، اضافه‌کردن Endpoint داخل `server.ts` منتفی بود. تنها راه واقعی باقی‌مانده، وصل‌شدن مستقیم
به همان دیتابیسی است که سرور شما هم به آن وصل است (Firestore). این یعنی این پنل و ربات شما **دو مصرف‌کننده‌ی مستقل از یک دیتابیس مشترک** هستند.

### ⚠️ محدودیت واقعی که باید بدانید (ریسک هم‌زمانی)
سرور Node شما کل دیتابیس را در حافظه (`dbCache`) نگه می‌دارد و هر از چندگاهی آن را کامل یا گرانولار روی Firestore می‌نویسد
(`saveEnterpriseDb`, `saveLicenseGranular`, …). یعنی اگر این پنل مستقیماً یک سند گروه را در Firestore تغییر دهد، **ممکن است
با نوشتار بعدیِ حافظه‌ی درون‌فرآیندیِ سرور Node رونویسی/گم شود** — چون سرور از تغییر بیرونی خبر ندارد تا وقتی که دوباره از Firestore
بارگذاری کند (مثلاً در ری‌استارت بعدی). این یک واقعیت معماری فعلی پروژه‌ی شماست، نه محدودیت این پنل.
راهکارهای عملی (بدون تغییر سورس):
- تغییرات حساس (تنظیمات/بن/میوت) را در ساعاتی که سرور کمتر در حال نوشتن است انجام دهید، یا
- بعد از تغییرات مهم از طریق پنل، سرویس Node را ری‌استارت کنید تا از Firestore دوباره بارگذاری شود، یا
- در آینده اگر خواستید، یک Endpoint فقط برای «Reload از Firestore» داخل سرورتان اضافه کنید (این خارج از دامنه‌ی درخواست فعلی است چون نیاز به تغییر سورس دارد).

---

## ۲. چه چیزی واقعاً در دیتابیس هست و چه چیزی نه (بدون حدس)

بر اساس بررسی کامل `types.ts`, `databaseManager.ts`, `server.ts`:

| چیزی که پرامپت خواسته | وضعیت واقعی در دیتابیس فعلی |
|---|---|
| تنظیمات گروه (`settings`) | ✅ وجود دارد — فیلدهای `antiSpam`, `antiFlood`, `antiLink`, `wordFilter`, `nightMode`, `welcomeMessage`, `warnLimit`, `restrictedWords`, قفل‌های `lockPhoto/Video/Voice/Sticker/Forward/Text/Join`, `autoMuteNewUsers`, `levelSystemEnabled` و... |
| نقش‌های گروه | ✅ وجود دارد ولی به‌صورت `Owner / Co-Owner / Manager / Moderator / VIP / Member` (نه OWNER/ADMIN/MODERATOR) — در این پنل نگاشت شده (بخش ۳) |
| بن/میوت | ✅ آرایه‌های `bannedUsers` و `mutedUsers` روی خود سند گروه |
| **لیست کامل اعضای گروه** | ❌ وجود ندارد. ربات فقط صاحبان نقش (owner/co-owner/manager/moderator/vip)، بن‌شده‌ها/میوت‌شده‌ها و `userStats` (کاربرانی که پیام داده و آمار XP دارند) را نگه می‌دارد. Telegram Bot API هم اصلاً اجازه نمی‌دهد یک ربات لیست کامل اعضای عادی گروه را بگیرد (فقط ادمین‌ها قابل لیست‌شدن‌اند). پس تب «اعضا» در این پنل، ترکیبی از نقش‌دارها + بن/میوت‌شده‌ها + فعال‌ترین کاربران بر اساس `userStats` است، **نه لیست کامل اعضا**. |
| **لاگ per-group** | ❌ وجود ندارد. `auditLogs` در دیتابیس ربات یک آرایه‌ی سراسری و سیستمی است (فقط ۱۰۰ مورد آخر، بدون فیلد `groupId`)، نه لاگ مخصوص هر گروه. بنابراین این پنل یک Collection کاملاً جدید و مستقل به نام `panel_audit_logs` در همان Firestore می‌سازد و فقط اعمالی که **از طریق خودِ پنل** انجام می‌شود را در آن ثبت می‌کند (طبق الزام Audit Log در پرامپت). این تغییری در سورس ربات نیست؛ فقط یک Collection جدید در همان دیتابیس است که ربات اصلاً به آن کاری ندارد. |
| ورود مالک/ادمین گروه | ❌ هیچ مکانیزمی برای این در ربات وجود نداشت. در این پنل با **Telegram Login Widget رسمی** پیاده شده (استاندارد و امن، بدون نیاز به پسورد جدا برای هر کاربر). |

---

## ۳. نقش‌ها و Permission Matrix

نقش واقعی گروه از روی همین فیلدهای موجود تشخیص داده می‌شود:

| فیلد در سند گروه | نقش واقعی | نقش پنل |
|---|---|---|
| `ownerTelegramId === userId` | Owner | **OWNER** |
| در آرایه‌ی `coOwners` یا `managers` | Co-Owner / Manager | **ADMIN** |
| در آرایه‌ی `moderators` | Moderator | **MODERATOR** |
| در آرایه‌ی `vips` یا هیچ‌کدام | VIP / Member | بدون دسترسی به پنل (403) |

| Permission | OWNER | ADMIN | MODERATOR |
|---|---|---|---|
| view | ✅ | ✅ | ✅ |
| settings | ✅ | ✅ | ❌ |
| members | ✅ | ✅ | ✅ |
| moderation (ban/mute) | ✅ | ✅ | ✅ |
| logs | ✅ | ✅ | ❌ |

این تشخیص نقش **روی هر Request، سمت سرور (Apps Script)** انجام می‌شود، نه سمت فرانت — دقیقاً طبق الزام امنیتی پرامپت.

---

## ۴. ⚠️ محدودیت واقعی Apps Script که روی «کد خطا»ی پرامپت اثر می‌گذارد

Google Apps Script Web App **همیشه در سطح HTTP کد ۲۰۰ برمی‌گرداند** — امکان فرستادن HTTP status واقعی ۴۰۱/۴۰۳/۴۰۴/۵۰۰ از یک Web App
اپس‌اسکریپت وجود ندارد (این محدودیت خودِ پلتفرم گوگل است، نه انتخاب طراحی). برای رعایت روح همان الزام، همه‌ی پاسخ‌ها یک فیلد
منطقی `status` در بدنه‌ی JSON دارند (`401`, `403`, `404`, `400`, `500`) و فرانت دقیقاً بر همان اساس رفتار می‌کند، نه بر اساس
HTTP status واقعی. این را صادقانه اینجا نوشتم تا حدس اشتباه نزنم.

همچنین به همین دلیل (و برای دور زدن CORS Preflight که Apps Script Web App پشتیبانی نمی‌کند)، همه‌ی درخواست‌های POST/PATCH
از فرانت با هدر `Content-Type: text/plain` فرستاده می‌شوند ولی بدنه‌شان JSON است — این یک ترفند شناخته‌شده و ضروری برای
Apps Script است، نه باگ.

---

## ۵. ساختار فایل‌ها

```
panel/
├── README.md                 همین فایل
├── frontend/                 → مستقیم روی GitHub Pages آپلود شود
│   ├── index.html
│   ├── css/style.css
│   └── js/
│       ├── config.js         فقط شامل آدرس Web App (هیچ Secret‌ای اینجا نیست)
│       ├── api.js            لایه‌ی ارتباط با API
│       └── app.js            رندر صفحات و منطق پنل
└── apps-script/               → محتوای این پوشه در یک پروژه‌ی Google Apps Script جدید و جدا ریخته می‌شود
    ├── appsscript.json
    ├── Code.gs               doGet/doPost + روتینگ
    ├── Auth.gs                تایید Telegram Login + صدور/بررسی Session Token
    ├── FirestoreClient.gs     کلاینت REST به Firestore با Service Account
    ├── Permissions.gs         تشخیص نقش و Permission Matrix
    ├── Groups.gs              هندلرهای گروه/تنظیمات/اعضا/بن/میوت
    └── Telegram.gs            فراخوانی واقعی Telegram Bot API برای ban/mute
```

---

## ۶. API Contract

Base URL = آدرس Deploy شده‌ی Apps Script (`.../exec`). همه چیز از طریق `POST` با بدنه‌ی JSON فرستاده می‌شود
(به‌جز `GET /me` و `GET /groups` که برای سادگی می‌توانند از Query String هم استفاده کنند). فیلد `action` مسیر را مشخص می‌کند
چون Apps Script روتینگ URL-based واقعی مثل Express ندارد.

هدر لازم برای همه (به‌جز لاگین): `X-Session-Token: <token>` (در بدنه هم به‌عنوان `sessionToken` قابل ارسال است، چون
مرورگر اجازه‌ی هدر دلخواه بدون Preflight نمی‌دهد و ما Preflight نداریم — پس در عمل توکن در بدنه‌ی JSON فرستاده می‌شود).

| Action | معادل REST در پرامپت | ورودی | Permission | خروجی موفق |
|---|---|---|---|---|
| `auth.telegramLogin` | (زیرساخت ورود) | داده‌ی خام Telegram Login Widget | - | `{ sessionToken, user }` |
| `me` | `GET /me` | `sessionToken` | فقط لاگین | اطلاعات کاربر لاگین‌شده |
| `groups.list` | `GET /groups` | `sessionToken` | فقط لاگین | فقط گروه‌هایی که کاربر در آن‌ها نقش دارد |
| `groups.get` | `GET /groups/{groupId}` | `groupId` | view | اطلاعات پایه‌ی گروه + نقش کاربر |
| `groups.settings.get` | `GET /groups/{groupId}/settings` | `groupId` | settings | تنظیمات کامل |
| `groups.settings.update` | `PATCH /groups/{groupId}/settings` | `groupId`, `patch{}` | settings | تنظیمات به‌روزشده |
| `groups.members.list` | `GET /groups/{groupId}/members` | `groupId` | members | نقش‌دارها + بن/میوت + فعال‌ترین‌ها |
| `groups.logs.list` | `GET /groups/{groupId}/logs` | `groupId` | logs | لاگ‌های همین پنل برای این گروه |
| `groups.member.ban` | `POST /groups/{groupId}/members/{userId}/ban` | `groupId`,`userId`,`reason` | moderation | - |
| `groups.member.unban` | `DELETE .../ban` | `groupId`,`userId` | moderation | - |
| `groups.member.mute` | `POST .../mute` | `groupId`,`userId`,`until`,`reason` | moderation | - |
| `groups.member.unmute` | `DELETE .../mute` | `groupId`,`userId` | moderation | - |

### فرمت پاسخ (همیشه یکسان)
```json
{ "success": true,  "status": 200, "data": { ... } }
{ "success": false, "status": 403, "error": "شما به این گروه دسترسی ندارید." }
```

کدهای `status` استفاده‌شده: `400` ورودی نامعتبر، `401` لاگین نامعتبر/منقضی، `403` عدم دسترسی به این `groupId`/Permission،
`404` گروه یا کاربر پیدا نشد، `500` خطای داخلی.

---

## ۷. مراحل Deploy — Google Apps Script

1. در https://script.google.com یک پروژه‌ی جدید بسازید (کاملاً جدا از هر پروژه‌ی دیگر).
2. تمام فایل‌های داخل `apps-script/` را با همین اسم‌ها در پروژه بسازید (پسوند `.gs` را Apps Script خودش مدیریت می‌کند).
3. یک Service Account بسازید:
   - در Google Cloud Console پروژه‌ی `gen-lang-client-0954132739` را باز کنید (همان پروژه‌ی فایربیس ربات).
   - IAM & Admin → Service Accounts → Create Service Account.
   - نقش `Cloud Datastore User` (یا `Firebase Rules System` معادل جدید: `Cloud Firestore User`) را به آن بدهید — این دسترسی
     کاملاً از طریق IAM است و **کاری به `firestore.rules` فایل شما ندارد** (rules فقط روی دسترسی کلاینتی/Firebase Auth اثر دارد).
   - یک کلید JSON برای Service Account بسازید و دانلود کنید.
4. در Apps Script: Project Settings → Script Properties، این مقادیر را اضافه کنید:
   - `FIREBASE_PROJECT_ID` = `gen-lang-client-0954132739`
   - `FIRESTORE_DATABASE_ID` = `ai-studio-orginbotaitelegr-86c949e5-d1c1-477d-a1fe-85e98a5e12cb`
   - `SA_CLIENT_EMAIL` = مقدار `client_email` از فایل JSON
   - `SA_PRIVATE_KEY` = مقدار `private_key` از فایل JSON (کامل، با `\n`ها)
   - `TELEGRAM_BOT_TOKEN` = توکن ربات (**فقط اینجا**، هرگز در فرانت/GitHub قرار نگیرد)
   - `SESSION_SECRET` = یک رشته‌ی تصادفی بلند و جدید که خودتان می‌سازید (کاملاً مستقل از `JWT_SECRET` سرور Node)
5. Deploy → New deployment → Type: **Web app**. Execute as: `Me`. Who has access: `Anyone`. آدرس `.../exec` را کپی کنید.
6. آن آدرس را در `frontend/js/config.js` به‌جای `API_BASE_URL` بگذارید.

## ۸. مراحل Deploy — GitHub Pages

1. محتوای پوشه‌ی `frontend/` را در یک ریپازیتوری جدید (یا شاخه‌ی `gh-pages`) قرار دهید.
2. در Settings → Pages، همان شاخه/پوشه را به‌عنوان منبع انتخاب کنید.
3. آدرس نهایی چیزی شبیه `https://USERNAME.github.io/REPO/` خواهد بود.
4. در BotFather، برای ربات‌تان دستور `/setdomain` را بزنید و همین دامنه‌ی GitHub Pages را ثبت کنید — Telegram Login Widget
   فقط روی دامنه‌ای که در BotFather ثبت شده کار می‌کند.

---

## ۹. تست امنیت دسترسی بین دو گروه (IDOR)

سناریوی تست:
1. با اکانت تلگرام مالک گروه A لاگین کنید → `sessionToken_A` بگیرید.
2. `groupId` واقعی گروه B را از هر طریق (مثلاً حدس، یا دیدن در URL یک ادمین دیگر) پیدا کنید.
3. با `sessionToken_A` درخواست بزنید: `groups.get` با `groupId = B`.
4. **انتظار:** `success:false, status:403`، و **هیچ داده‌ای از گروه B نباید در پاسخ باشد** (نه حتی نام گروه).
5. همین تست را برای `groups.settings.update`, `groups.member.ban` هم با `groupId=B` تکرار کنید → همه باید `403` بدهند.
6. یک بار هم با `sessionToken` جعلی/دستکاری‌شده تست کنید → باید `401` بدهد، نه `403` (چون اصلاً هویت تایید نشده).

این منطق در `Permissions.gs` → تابع `resolveRole(groupDoc, telegramId)` پیاده شده: نقش همیشه **از روی خودِ سند گروهِ درخواست‌شده**
محاسبه می‌شود، نه از چیزی که فرانت ادعا می‌کند.

---

## ۱۰. آنچه در این نسخه پیاده نشده (محدوده‌ی صریح)

- ارسال کد OTP در پیوی توسط خودِ ربات (چون یعنی باید از طریق Bot API پیام پوش کنیم؛ اگر ترجیح می‌دهید به‌جای Telegram Login
  Widget این را داشته باشید، بگویید تا اضافه کنم).
- مدیریت نقش‌ها (اضافه/حذف ادمین) — در پرامپت اصلی خواسته نشده بود، فقط permission-check روی نقش‌های موجود.
- پیج‌بندی (pagination) واقعی روی لیست گروه‌ها/لاگ‌ها — فعلاً همه در یک صفحه برمی‌گردد؛ برای دیتای کوچک/متوسط مشکلی نیست.
