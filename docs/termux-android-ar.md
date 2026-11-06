# explainroo على Termux / أندرويد (Samsung A10)

خلاصة سريعة: **التحرير وكتابة المشاريع يعمل على الهاتف، أما الرندر الكامل فلا** —
على الأقل على A10 بنسخة الشركة. هذا الملف يشرح السبب بالأرقام، وما الذي يمكن
تشغيله فعلاً، وكيف تُخرج فيديو كاملاً وأنت على الهاتف.

---

## 1. لماذا لا يكفي Termux وحده

المستودع يحتاج ثلاث حزم فيها **ملفات ثنائية أصلية**، وكلها مبنية لـ Linux glibc:

| الحزمة | الدور | ما تتوقعه | حالة Termux |
|---|---|---|---|
| `onnxruntime-node` 1.21 | Kokoro (الصوت) + Whisper (التوقيت) | `linux-x64` / `linux-arm64` glibc | أندرويد يستخدم bionic لا glibc، ولا يوجد build لـ `armv7` |
| Chromium (يرشّده `playwright-core`) | رسم الإطارات | Chromium لـ Linux x64/arm64 | لا حزمة رسمية على Termux، وبالكاتب `pkg install chromium` فهي من `x11-repo` وغالباً `aarch64` فقط |
| `sharp` (تبعية لـ transformers.js) | معالجة صور | prebuild glibc / musl | لا يوجد build لأندرويد |

فالنتيجة المتوقَّعة عند `npm install` ثم التشغيل:

```
Error: Cannot find module '.../onnxruntime_binding.node'
# أو
invalid ELF header
```

### وضع جهاز A10 تحديداً

- المعالج: Exynos 7884/7884B (Cortex‑A53) — **المستخدم 32‑بت فقط** (`armeabi-v7a`)
  على نواة arm64. أي أن Termux سيثبّت حزماً 32‑بت.
- الذاكرة: 2 غيغابايت (بعض النسخ 3).
- التخزين: 32 غيغابايت، والمتاح عملياً أقل بكثير.

مشكلة الـ32‑بت قاتلة هنا: كل الملفات الثنائية المطلوبة (Chromium، ONNX، sharp)
تُبنى لـ64‑بت فقط في عالم Linux. حتى مع `proot-distro` تبقى الحزم 32‑بت
لأن المستخدم نفسه 32‑بت.

### تحقّق بنفسك

```bash
uname -m                       # armv7l / armv8l = 32-bit
getprop ro.product.cpu.abilist # armeabi-v7a,armeabi
free -m | head -2
node -v                        # يجب 20.11 أو أحدث
node --input-type=module -e "import('onnxruntime-node').then(()=>console.log('ORT ok')).catch(e=>console.log('ORT FAIL:',e.message))"
ls $PREFIX/bin | grep -i chrom
```

---

## 2. ما يعمل فعلاً على الهاتف اليوم

هذه أوامر JavaScript خالصة بلا متصفح ولا ONNX، وقد جرّبتها وتعمل:

```bash
pkg install nodejs git
git clone https://github.com/vincentsch/explainroo.git
cd explainroo

node bin/explainroo.js --help
node bin/explainroo.js doctor          # سيُظهر FAIL لـ chrome وmodels، وهذا متوقع
node bin/explainroo.js themes
node bin/explainroo.js formats
node bin/explainroo.js icons arrow
node bin/explainroo.js init videos/first --theme paper --size vertical --title "فكرتي"
```

ما لا يعمل محلياً: `voice` و`check` و`still` و`sheet` و`render` و`export` و`capture`
(كلها تحتاج إمّا ONNX أو Chromium).

بهذا تستطيع على الهاتف: كتابة `script.md`، كتابة `scenes.js`، قراءة `AGENTS.md`،
التحقق من صحة النص عبر `check` (لا، يحتاج كروم) — عملياً: التحرير والمراجعة النصية
وGit فقط، ثم الرندر في السحابة.

---

## 3. الحل العملي: التحرير على الهاتف، الرندر في السحابة

أضفت سير عمل جاهزاً: **`.github/workflows/render.yml`**.

### الطريقة (كلها من متصفح الهاتف)

1. ارفع المشروع إلى المستودع. **مهم:** مجلد `videos/` مُستثنى في `.gitignore`،
   فضع المشروع في `examples/<name>/` أو `projects/<name>/` ليُرفع فعلاً.
2. افتح المستودع على GitHub ← تبويب **Actions** (قد تحتاج تفعيل Actions مرة واحدة).
3. اختر **render** ← **Run workflow**، واملأ:
   - `project`: مثل `projects/my-video`
   - `size`: `vertical` أو `tiktok` أو اتركه فارغاً
   - `draft`: `true` للتجربة السريعة، ثم `false` للنسخة النهائية
   - `seconds`: مثلاً `20` لتجربة أول 20 ثانية
4. بعد انتهاء التشغيل، افتح الـrun وحمّل الـartifact باسم `video-<رقم>`
   (فيه MP4 + SRT + VTT + الغلاف + الفصول).

السير عمل يقوم بـ: `npm ci` ← `doctor --fetch` ← `check` ← `render` ← `verify`،
ويحفظ نماذج الصوت في الكاش (400MB) بين التشغيلات. مدة التشغيل على عدّاء GitHub
(4 أنوية، 16 غيغابايت، Chrome مثبّت مسبقاً) أقرب إلى حاسوب مكتبي.

### بدائل أخرى

| الخيار | متى يكون أفضل |
|---|---|
| GitHub Codespaces | تريد `preview` تفاعلياً وتعديلاً مباشراً من متصفح الهاتف |
| حاسوب شخصي/VPS | مشاريع طويلة أو كثيرة، أو تحتاج `capture` لموقع حي |
| GitHub Actions | الأبسط: بلا إعداد، ومجاني ضمن حدود الحساب |

---

## 4. إن كان جهازك 64‑بت (ROM مخصّص) وتريد المحاولة محلياً

هذا **لا ينصح به** على 2 غيغابايت، لكنه المسار الوحيد الممكن:

```bash
pkg install proot-distro
proot-distro install debian
proot-distro login debian
apt update && apt install -y curl git ffmpeg chromium fonts-noto
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
```

ثم داخل Debian:

```bash
export EXPLAINROO_CHROME=$(which chromium)
export EXPLAINROO_TTS_DTYPE=q8
node bin/explainroo.js doctor
node bin/explainroo.js render projects/first --draft --workers 1 --scale 0.5 \
     --size 854x480 --to 20
```

### إعدادات تخفيف الضغط

| الإعداد | القيمة | السبب |
|---|---|---|
| `EXPLAINROO_TTS_DTYPE` | `q8` | يقلّل ذاكرة Kokoro من ~330MB إلى ~90MB |
| `--workers` | `1` | متصفح واحد فقط |
| `--scale` | `0.5` | نصف الأبعاد = ربع العمل |
| `--draft` | مطلوب | يتخطى motion blur (6 عيّنات لكل إطار) |
| `size` | `854x480` أو `1280x720` | لا تحاول 1080p على A53 |
| `fps` | `24` | أخف من 30 (المسموح: 24/25/30/50/60) |
| `--to` | 20–30 ثانية | اختبر مقطعاً قبل الفيديو الكامل |

نصائح إضافية: `termux-wake-lock` لمنع النوم، أغلق التطبيقات الأخرى، امسح
`build/segments` بين المحاولات، وضع الكاش على التخزين الداخلي لا بطاقة SD.
توقّع أن يستغرق مقطع قصير عشرات الدقائق، مع بطء شديد وحرارة عالية.

---

## 5. ملاحظات للعربية

سواء رندرت محلياً أو في السحابة، تذكّر أن **العربية غير مدعومة في التعليق
أو في النص على الشاشة** (لا صوت، لا خط، لا RTL). لغات التعليق هي:
`en, es, fr, it, pt-br, hi, ja, zh`. التفاصيل وخطة الإضافة في
`docs/explainroo-study-ar.md`، القسم 9.

إن احتجت فيديو عربياً اليوم: اكتب النص بالإنجليزية، أو أضف دعم العربية في fork
(خط عربي + `direction: 'rtl'` + مصدر صوت عبر ElevenLabs + تمرير `ar` إلى Whisper).
