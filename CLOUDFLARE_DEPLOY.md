# نشر Option Travel على Cloudflare

## ما الذي تم تجهيزه؟

تمت إضافة `wrangler.jsonc` و`cloudflare/worker.ts`. Worker يخدم واجهة React من Cloudflare Assets ويمرر كل طلبات `/api/*` إلى Backend خارجي عبر `API_ORIGIN`. بهذه الطريقة تبقى tRPC، قاعدة البيانات، تسجيل الدخول، رفع الإيصالات وWebhooks تعمل، بدلاً من رفع واجهة ثابتة بلا Backend.

هذا هو النموذج المناسب للمشروع الحالي لأن Backend يستخدم Express وMySQL وOAuth وتخزين الملفات. يمكن لاحقاً نقل الـ Backend بالكامل إلى Workers، لكن ذلك يتطلب نقل قاعدة البيانات والخدمات إلى D1/Hyperdrive وR2 وOAuth متوافق مع Workers.

## النشر من جهازك

من جذر المشروع:

```bash
pnpm install
pnpm build:cloudflare
npx wrangler login
npx wrangler deploy --var API_ORIGIN:https://YOUR-BACKEND-DOMAIN
```

استبدل `https://YOUR-BACKEND-DOMAIN` بعنوان Backend المنشور فعلياً. لا تستخدم عنوان `localhost`. إذا كان Backend ما زال على WebDev، استخدم عنوان HTTPS العام له مؤقتاً، والأفضل نشر Backend على خدمة Node دائمة قبل الإنتاج.

بعد النشر سيظهر عنوان `*.workers.dev`. اختبر:

```bash
curl -I https://YOUR-WORKER.workers.dev
curl -i https://YOUR-WORKER.workers.dev/api/webhooks/n8n
```

الطلب الثاني يجب أن يرجع `503` أو `401` بحسب إعداد n8n، وليس `404`؛ هذا يؤكد أن Proxy الـ API يعمل.

## إعدادات Cloudflare Dashboard

إذا نشرت من Cloudflare Dashboard بدلاً من Wrangler:

| الإعداد | القيمة |
|---|---|
| Framework preset | React (Vite) |
| Build command | `pnpm build:cloudflare` |
| Build output directory | `dist/public` |
| Root directory | `/` |
| Worker entry | `cloudflare/worker.ts` |
| Variable | `API_ORIGIN` = عنوان Backend العام |

للنشر كـ Worker مع Static Assets استخدم Wrangler؛ إعداد Pages وحده سيخدم الواجهة فقط ولن يمرر `/api` إلى Backend.

## ملاحظة مهمة عن n8n

بعد ظهور عنوان Cloudflare النهائي، اجعل عنوان Webhook الوارد في n8n هو:

```text
https://YOUR-WORKER.workers.dev/api/webhooks/n8n
```

ويظل Webhook الصادر من الموقع مضبوطاً في جدول `n8n_settings` كما هو موضح في دليل n8n. استخدم HTTPS Production Webhook وليس Test Webhook.

## متغيرات Backend المطلوبة

لا تضع هذه القيم في الواجهة أو داخل Git. يجب أن تكون متاحة على Backend:

- `DATABASE_URL`
- إعدادات OAuth الخاصة بالموقع
- إعدادات التخزين
- `SUPABASE_URL` و`SUPABASE_SERVICE_ROLE_KEY` إذا أردت أن يقرأ الموقع جدول `public.n8n_settings` مباشرة من Supabase

## سبب عدم استخدام `wrangler deploy` مباشرة

الأمر يحتاج أولاً إلى بناء `dist/public`. لذلك استخدم:

```bash
pnpm build:cloudflare && npx wrangler deploy --var API_ORIGIN:https://YOUR-BACKEND-DOMAIN
```
