# تركيب سكيلز معايير NGP

بالملف المضغوط في 5 سكيلز، كل وحدة بمجلد جوّا `skills/`:

| السكيل | لشو |
|---|---|
| `ngp-general` | قواعد عامة لكل المشاريع: git، commits، CI، env، RTL |
| `ngp-laravel` | الباك إند Laravel |
| `ngp-react` | فرونت React + Vite |
| `ngp-nextjs` | مواقع Next.js العامة |
| `ngp-flutter` | تطبيقات Flutter (GetX) |

كل سكيل هو ملف `SKILL.md` عادي، فيك تفتحه وتقراه متل أي ملف md.

اختار طريقة وحدة من التلاتة:

---

## الطريقة 1: على جهازك لكل المشاريع (الأسهل)

السكيلز بتشتغل بأي مشروع بتفتحه بـ Claude Code على هالجهاز.

1. فك الضغط.
2. انسخ المجلدات يلي جوّا `skills/` (مو مجلد `skills` نفسه) لهون:
   - **Windows:** `C:\Users\<اسمك>\.claude\skills\`
   - **Mac / Linux:** `~/.claude/skills/`
3. لازم يطلع الشكل هيك:
   ```
   .claude/skills/ngp-general/SKILL.md
   .claude/skills/ngp-laravel/SKILL.md
   .claude/skills/ngp-react/SKILL.md
   .claude/skills/ngp-nextjs/SKILL.md
   .claude/skills/ngp-flutter/SKILL.md
   ```
4. سكّر Claude Code وافتحه من جديد.

بـ PowerShell (بعد فك الضغط بمجلد `ngp-standards`):
```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse -Force ".\ngp-standards\skills\*" "$env:USERPROFILE\.claude\skills\"
```

---

## الطريقة 2: جوّا مشروع معيّن (لكل الفريق عبر git)

السكيلز بتنحفظ مع المشروع، وأي حدا بيعمل clone بياخدها تلقائياً.

1. بجذر المشروع اعمل مجلد `.claude/skills/`.
2. انسخ بس السكيلز يلي بتلزم هالمشروع، مثلاً لمشروع Flutter:
   ```
   .claude/skills/ngp-general/SKILL.md
   .claude/skills/ngp-flutter/SKILL.md
   ```
3. اعمل commit و push.
4. ضيف هالسطر لـ `CLAUDE.md` بالمشروع:
   ```
   Follow the NGP standards: ngp-general + ngp-flutter.
   ```

---

## الطريقة 3: كـ plugin (الأفضل للشركة على المدى الطويل)

بس ترفع محتوى الملف المضغوط على ريبو `ngp-techworld/ngp-standards`، كل مبرمج بيكتب جوّا Claude Code:

```
/plugin marketplace add ngp-techworld/ngp-standards
/plugin install ngp-standards@ngp
```

ولما تتعدّل المعايير، كل واحد بيعمل:
```
/plugin marketplace update ngp
```

قبل ما ترفعه فيك تجربه من جهازك مباشرة:
```
/plugin marketplace add C:\path\to\ngp-standards
/plugin install ngp-standards@ngp
```

---

## كيف بتتأكد إنها شغالة

افتح Claude Code بمشروع Flutter مثلاً واسأله:
```
شو السكيلز المتاحة عندك؟ واشرحلي هيكلية مشروع Flutter حسب معايير NGP
```
لازم يذكر `ngp-flutter` ويشرح core / data / features مع GetX.

لما تطلب منه ميزة جديدة، بيمشي على المعايير لحاله. وإذا بدك تجبره، اذكر السكيل بالاسم:
```
اعمل ميزة orders حسب ngp-flutter
```

---

## ملاحظات

- لا تغيّر اسم المجلد ولا السطر `name:` بأول الـ SKILL.md، هنن يلي بيعرّفوا السكيل.
- إذا عدّلت ملف SKILL.md، أعد فتح Claude Code ليقرا النسخة الجديدة.
- بالطريقة 1 و 2، أي تعديل لازم تنسخه يدوياً لكل جهاز أو مشروع. لهيك الطريقة 3 أحسن لما يصير الفريق أكبر.
