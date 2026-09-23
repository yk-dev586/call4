# Cloud Video Call — النسخة المُصلحة 3.1.0

هذه النسخة تُبقي الواجهة الحالية ونظام غرف المكالمات، وتُصلح طبقة ICE/WebRTC وتدعم Cloudflare TURN بشكل آمن من جهة الخادم.

## ما تم إصلاحه

- إصلاح فحص Origin في `/api/ice` حتى لا يفشل طلب same-origin صحيح عندما لا يرسل المتصفح Origin في بعض طلبات GET.
- إضافة 3 خوادم STUN احتياطية.
- دعم Cloudflare TURN تلقائيًا عند ضبط `TURN_KEY_ID` و`TURN_API_TOKEN`.
- عدم إرسال أي مفتاح TURN طويل الأجل أو API token إلى المتصفح.
- استخدام بيانات TURN قصيرة العمر وإعادة جلبها من Worker.
- منع المنفذ 53 من قائمة TURN لأن Cloudflare توضح أنه لا يعمل في المتصفحات.
- إضافة ICE candidate pool.
- إضافة إعادة تفاوض/ICE restart تلقائية مرتين عند فشل المسار.
- إضافة `/health` لمعرفة هل TURN مضبوط (`turnConfigured`) بدون كشف الأسرار.
- الإبقاء على الاتصال المباشر P2P أولًا، واستخدام TURN كـ relay عند الحاجة.

## النشر باستخدام Wrangler

> مهم: TURN لا يمكن تفعيله فعليًا من الكود وحده؛ يجب إنشاء TURN key وAPI token في حساب Cloudflare ثم وضعهما كأسرار في Worker. Cloudflare توصي بأن تبقى هذه القيم في الخادم ولا تُكشف للواجهة.

### 1. تثبيت الاعتمادات

```bash
npm install
```

### 2. تسجيل الدخول

```bash
npx wrangler login
```

### 3. ضبط TURN_KEY_ID

ضع `uid`/المعرّف الخاص بمفتاح TURN في متغير Worker باسم:

```text
TURN_KEY_ID
```

يمكن وضعه كمتغير عادي لأنه معرّف المفتاح؛ لا تضع القيمة السرية لمفتاح TURN داخل الكود أو الواجهة.

### 4. ضبط TURN_API_TOKEN كسرّ

أنشئ API token في Cloudflare بصلاحية Calls Write الخاصة بإنشاء/استخدام TURN، ثم:

```bash
npx wrangler secret put TURN_API_TOKEN
```

سيطلب منك Wrangler القيمة محليًا؛ لا تكتبها داخل Git أو داخل `CloudVideoCall_Dashboard_Worker.js`.

### 5. النشر

```bash
npx wrangler deploy
```

## التحقق بعد النشر

افتح:

```text
https://YOUR-WORKER-DOMAIN/health
```

ويفترض ظهور:

```json
{
  "ok": true,
  "service": "cloud-video-call",
  "version": "3.1.0",
  "turnConfigured": true
}
```

ثم افتح:

```text
https://YOUR-WORKER-DOMAIN/api/ice
```

ويجب أن ترى `iceServers` وبداخلها STUN، وعند تفعيل TURN تظهر كذلك عناوين `turn:`/`turns:` مع بيانات اعتماد مؤقتة.

## ملاحظة مهمة عن Cloudflare Dashboard

إذا كان Worker الحالي موجودًا بالفعل ولديه Durable Object باسم `CallRoom`، لا تُنشئ Namespace جديدًا عشوائيًا ولا تحذف الموجود. استخدم نفس binding باسم `CALL_ROOM` ونفس class باسم `CallRoom`، أو احتفظ بملف migration الحالي الخاص بالمشروع.

## معلومات تقنية

الاتصال الإعلامي نفسه لا يمر عبر Worker في الحالة الطبيعية؛ Worker يستخدم للإشارة (signaling)، بينما WebRTC يحاول الاتصال المباشر أولًا. TURN يصبح relay عندما تمنع الشبكة الاتصال المباشر.

النسخة لا تعتبر وجود TURN شرطًا لبدء المكالمة: تعمل بالمسار المباشر عندما يكون ممكنًا، وتستخدم TURN تلقائيًا عندما تكون بيانات TURN متاحة.
