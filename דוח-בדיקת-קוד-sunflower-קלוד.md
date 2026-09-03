# דוח בדיקת קוד — Sunflower Intelligence (Lovable)

**תאריך:** 3 בספטמבר 2026
**נבדק ע"י:** Claude Code
**פרויקט Lovable:** `gemini-genesis-engine` — "Sunflower Strategic BizDev projectile analysis"
**Project ID:** `d47c4603-563b-4790-a151-b899fd9b4ee5`
**Commit שנבדק:** `ed77784fcef245ef4c3f93fdf1aefb9442d429b8`
**Supabase ref:** `jtofvaetlpfmlzgiqwzj`
**סטאק:** Vite + React + TypeScript + shadcn/ui + Supabase (Edge Functions + Postgres)

---

## 0. תקציר מנהלים

נבדקה ליבת הלוגיקה העסקית, פונקציות ה-backend ומדיניות ההרשאות של האפליקציה.
**הממצא המרכזי: קיים פער שיטתי בין הקוד בפועל לבין ההחלטות הקנוניות שתועדו במסמך המסירה (v5.0).**

הפער אינו נקודתי אלא חוזר בשלוש שכבות בלתי-תלויות — פרומפט ה-AI, טריגר מסד הנתונים, וקוד הלקוח — כאשר כל שכבה מיישמת גרסה אחרת של אותה לוגיקה. התוצאה: אותו משפיען יכול לקבל שלושה ציונים שונים ושלוש תוויות גיאופוליטיות שונות, בהתאם למי חישב אחרון.

**סיכום כמותי:** 17 ממצאים — 5 קריטיים, 5 בינוניים, 7 קלים.

### שורש הבעיה

הפרויקט עבר אבולוציה מגרסת HTML/Gemini מקורית → ארכיטקטורת v2 מפוצלת לשלבים. **הקוד הישן לא הוסר** — הוא רץ במקביל לחדש. בנוסף, החלטות קנוניות שהתקבלו (החלטות 1, 3, 4, 5, 8 במסמך המסירה) **לא הוטמעו** ב-`analyze-entity` ובטריגרי ה-DB.

---

## 1. ממצאים קריטיים 🔴

### ממצא 1 — חסר `humanitarian_concern`, ועדיין קיים `unknown`

**קובץ:** `supabase/functions/analyze-entity/index.ts` (שני הפרומפטים: bulk + single)

הפרומפט מגדיר את רמות הסיכון כ:
```
pro_israel | neutral | unknown | suspected | confirmed
```

הקאנון (החלטה 3 + החלטה 4 במסמך המסירה):
```
pro_israel | neutral | humanitarian_concern | suspected | confirmed_anti
```

**המשמעות:**
- רמת הביניים `humanitarian_concern` — כל מהותה של החלטה 3 — **לעולם לא נוצרת ע"י ה-AI**. מקרי הגבול (תוכן הומניטרי על עזה ללא BDS) מסווגים בטעות כ-`neutral` או כ-`suspected` — בדיוק שתי הטעויות שהחלטה 3 נועדה למנוע.
- `unknown` בוטל בהחלטה 4 אך עדיין בשימוש.
- שם הרמה `confirmed` במקום `confirmed_anti`.

**השלכה נוספת:** הדגל הכתום `SENSITIVE` נבדק ב-`verdictCalculation.ts` (`orangeFlags.includes('SENSITIVE')`), אך מכיוון שה-AI לעולם לא מייצר `humanitarian_concern` — הענף הזה הוא **קוד מת**.

---

### ממצא 2 — VETO עוצר את המחקר, בניגוד להחלטה 5

**קובץ:** `supabase/functions/analyze-entity/index.ts`

הפרומפט קובע מפורשות:
```
confirmed: Hard evidence found → AUTOMATIC VETO - DO NOT PROCEED TO STEP 2
```
והקוד מממש זאת — בזיהוי `vetoTriggered` הוא מחזיר מיד `HARD_NO` ללא המשך ניתוח עסקי.

**החלטה 5 קובעת את ההפך:** VETO **אינו** עוצר. המערכת חייבת להשלים דוסייה מלא (ציונים בכל 6 הדימנשנים) לצורך:
1. סקירה אנושית עם ראיות
2. יכולת override של BD
3. audit trail ל-compliance

זהו בדיוק הכיוון ש**נוסה ונזנח** (חלק ד' §2 במסמך המסירה).

---

### ממצא 3 — טריגר ה-DB מחשב משקלים שגויים ומתעלם ממשקלי המשתמש

**קובץ:** `supabase/migrations/20260129153137_*.sql`, פונקציה `calculate_weighted_score`

המשקלים **מקודדים קשיחות** בפונקציה, ושונים מהקאנון:

| דימנשן | טריגר ה-DB | קאנוני (החלטה 1) | פער |
|---|---|---|---|
| relevance | 0.25 | 0.25 | ✅ |
| reach | **0.15** | 0.20 | ❌ −5 |
| partnership_readiness | **0.20** | 0.15 | ❌ +5 |
| competition | **0.10** | 0.15 | ❌ −5 |
| brand_safety | 0.15 | 0.15 | ✅ |
| strategic_fit | **0.15** | 0.10 | ❌ +5 |

**שתי בעיות נפרדות:**
1. **המשקלים פשוט שגויים** ביחס להחלטה 1.
2. **הפונקציה מתעלמת לחלוטין** מהמשקלים שהמשתמש מגדיר ב-`app_settings.score_weights` — כל מסך "הגדרות משקולות" (`WeightsSettingsPage`) לא משפיע על החישוב הזה.

**חמור מכך — דריסה הדדית:** הטריגר `trigger_update_calculated_scores` רץ `BEFORE INSERT OR UPDATE` על עמודות הציונים ודורס את `score_weighted_total`. במקביל, `useVerdictRecalculation` (לקוח) כותב לאותה עמודה ערך אחר עם המשקלים הנכונים. **מי שכתב אחרון מנצח** → הערך אינו דטרמיניסטי.

---

### ממצא 4 — שלוש מערכות ניקוד/דירוג מקבילות ולא תואמות

| # | מנגנון | היכן | מה מחשב |
|---|---|---|---|
| 1 | `useWeightedScoring.ts` | דשבורד ראשי (`ScoringControls`) | `totalScore` מ-5 מדדים ישנים: followers / affinity / virality / engagement / cost + פריסטים (balanced/exposure/roi/authenticity) |
| 2 | `calculate_weighted_score` (DB) | טריגר על `influencers` | `score_weighted_total` — 6 דימנשנים, משקלים שגויים וקשיחים |
| 3 | `verdictCalculation.ts` + `useVerdictRecalculation` | לקוח, לחיצה ידנית | `score_weighted_total` — 6 דימנשנים, משקלים נכונים ומוגדרים |

בנוסף קיים `priority_score` — טריגר DB נפרד (`calculate_research_priority_score`) עם מכפילי verdict (PRIORITY_PARTNER ×1.5 … HARD_NO ×0) ובונוס +10 לפרו-ישראל. זהו השדה שלפיו `list_influencers` ב-MCP ממיין.

**המשמעות:** הדשבורד הראשי מדרג משפיענים לפי המערכת **הישנה** (#1), שמבוססת על שדות (`affinityScore`, `viralityPotential`) שכלל לא מיוצרים ע"י ה-pipeline החדש. הדירוג שהמשתמש רואה מנותק מהניקוד הקאנוני.

**אישוש:** `ScoringControls.tsx` מייבא `SCORING_PRESETS` ו-`ScoringWeights` מ-`useWeightedScoring` — המערכת הישנה חיה ומחוברת ל-UI.

---

### ממצא 5 — ריבוי אוצרות-מילים לאותם מושגים

**Verdict — שלוש גרסאות:**

| מקור | ערכים |
|---|---|
| `verdictCalculation.ts` | 6 verdicts + 4 drivers כולל `company_sentiment` ✅ |
| `analyze-entity` → `verdict.primaryDriver` | `geopolitical｜business｜combined` — **חסר `company_sentiment`** (החלטה 8) |
| `analyze-entity` → `executiveSummary.recommendation` | `PRIORITY_YES｜STRONG_YES｜YES｜CONDITIONAL｜CAUTION｜NO｜HARD_NO` — סט שלישי ושונה |

**גיאופוליטיקה — שלוש גרסאות:**

| מקור | ערכים |
|---|---|
| קאנון (מסמך המסירה) | `pro_israel｜neutral｜humanitarian_concern｜suspected｜confirmed_anti` |
| `analyze-entity` | `pro_israel｜neutral｜unknown｜suspected｜confirmed` |
| הערת עמודה `geopolitical_level` ב-DB | `PRO_ISRAEL｜NEUTRAL｜UNKNOWN｜CONCERNING｜HOSTILE` |

חוסר עקביות זה שובר כל מיפוי בין השכבות ומקשה על תצוגה נכונה ב-UI.

---

## 2. ממצאים בינוניים 🟠

### ממצא 6 — `analyze-entity` ללא אכיפת JSON schema

הפונקציה שולחת ל-`google/gemini-3-pro-preview` בלי `response_format` / schema enforcement, ואז:
```js
const jsonContent = content.replace(/```json\n?|\n?```/g, '').trim();
parsed = JSON.parse(jsonContent);
```
בכשל פרסינג יש **fallback שקט** ששומר את הטקסט הגולמי לתוך `researchNotes` ומחזיר `confidence: 'low'` — כלומר כשל נראה כמו הצלחה חלקית.

זהו בדיוק הכשל שתועד כ"נוסה ונזנח" (חלק ד' §7): *"Gemini עם prompt control בלבד יוצר שגיאות פרסינג ב-production"*. ההחלטה הייתה **Gemini + JSON schema**.

---

### ממצא 7 — `research_status` מתעדכן רק ל-influencer

**קובץ:** `supabase/functions/receive-research-results/index.ts`, `handleComplete`

```js
if (entity_type === 'influencer') {
  entityUpdate.research_status = researchStatus;
  entityUpdate.research_stage = ...;
}
```
ארגונים (`organizations`) ומדינות (`country_strategies`) **לא מקבלים** מעבר סטטוס בסיום מחקר — למרות ש-`list_organizations` ב-MCP בוחר `research_status`. הסטטוס שלהם יישאר תקוע.

---

### ממצא 8 — טיפול שונה ב-null בין DB ללקוח

| | התנהגות | תוצאה |
|---|---|---|
| טריגר DB | `COALESCE(score, 0)` | דימנשן חסר נספר כ־0 |
| `verdictCalculation.ts` | מחזיר `null` אם דימנשן כלשהו חסר | הישות מדולגת (`skipped`) |

ישות מנוקדת חלקית מקבלת ב-DB ציון נמוך מלאכותי שנראה לגיטימי, בעוד הלקוח מסמן אותה כלא-ניתנת-לחישוב. סתירה ישירה.

---

### ממצא 9 — שער האישור מנוטרל + חשש לנעילת משתמשים רגילים

שני ממצאים משלימים וסותרים זה את זה:

**א. השער עוקף את עצמו.** `handle_new_user` (migration `20251225115740`) מקצה אוטומטית תפקיד `user` לכל הרשמה:
```sql
INSERT INTO public.user_roles (user_id, role) VALUES (NEW.id, 'user');
```
לכן ברמת ה-DB **כל מי שמתחבר עם Google מקבל גישה מיידית לכל נתוני המחקר**. כל תשתית האישור (`AwaitingApprovalPage`, `notify-admin-new-user`, `notify-user-approved`, `usePendingUsersCount`, `AdminUsersPage`) מיותרת בפועל.

**ב. אך משתמשים רגילים עלולים בכל זאת להיחסם.** מדיניות ה-SELECT היחידה שנמצאה על `user_roles` היא:
```sql
CREATE POLICY "Admins can read user roles" ON public.user_roles
FOR SELECT TO authenticated USING (public.has_role(auth.uid(), 'admin'));
```
אך `useUserRole` קורא את התפקיד של המשתמש עצמו. ללא מדיניות self-read, משתמש שאינו אדמין יקבל 0 שורות (RLS מסנן בשקט, ללא שגיאה) → `roles = []` → `ProtectedRoute` מפנה ל-`/awaiting-approval` לצמיתות.

**נטו:** ייתכן שרק האדמין (Itay) יכול להשתמש באפליקציה בפועל, בעוד ה-DB מעניק גישה לכולם.

> ⚠️ **דורש אימות מול המסד החי.** נקראו כ-8 מתוך ~30 migrations; ייתכן שמדיניות self-read נוספה במקום שלא נבדק.

---

### ממצא 10 — חישוב-מחדש N+1 בצד לקוח

**קובץ:** `src/hooks/useVerdictRecalculation.ts`

```js
for (const update of updates) {
  await supabase.from('influencers').update({...}).eq('id', update.id);
}
```
UPDATE נפרד לכל משפיען, ברצף, מהדפדפן. לא אטומי, לא טרנזקציוני, ואם המשתמש סוגר את הטאב באמצע — חלק מהישויות מעודכנות וחלק לא. עדיף Edge Function עם עדכון מרוכז.

---

## 3. ממצאים קלים 🟡

| # | ממצא | קובץ |
|---|---|---|
| 11 | `app_settings` ו-`app_config` קריאים לכולם (`USING (true)`) — כולל אנונימיים. לא סוד כרגע, אך דלת פתוחה לקונפיג רגיש עתידי | migrations `20251223085357`, `20260115210741` |
| 12 | `.env` קיים כקובץ בריפו — **לא נקרא** (נחסם). יש לוודא שמכיל רק מפתחות ציבוריים (`VITE_*`/publishable) ולא service-role | `.env` |
| 13 | השוואת סוד webhook לא בזמן-קבוע (`providedSecret !== expectedSecret`) — ערוץ צד תזמוני זניח | `receive-research-results` |
| 14 | אין ולידציה ל-`entity_type` ב-`handleComplete` לפני בניית שם הטבלה `${entity_type}s` | `receive-research-results` |
| 15 | `calculateWeightedScore` מחלק ב-100 קבוע (מניח סכום משקלים = 100). ה-UI אוכף `total === 100`, אך עדיף לחלק בסכום בפועל לחוסן | `verdictCalculation.ts` |
| 16 | באג קוסמטי: `PRESET_STYLES` מכיל מפתח `reach`, אך הפריסט נקרא `exposure` — הפריסט מקבל אייקון ברירת מחדל | `ScoringControls.tsx` |
| 17 | `research_jobs` — מדיניות SELECT היא `USING (true)` לכל authenticated: כל משתמש רואה את כל העבודות של כולם | migration `20260204124714` |

---

## 4. מה נעשה נכון ✅

לצורך איזון — הבסיס בנוי היטב במספר מקומות:

- **`has_role` מוגדרת כ-`SECURITY DEFINER` עם `SET search_path = public`** — הדרך הנכונה למנוע רקורסיית RLS.
- **תפקידים בטבלה נפרדת** (`user_roles`) ולא כעמודה על `profiles` — עמיד בפני הסלמת הרשאות.
- **RLS מופעל** על כל הטבלאות שנבדקו.
- **`trigger-research` מאמת משתמש דרך `getUser()`** ושולף את נתוני הישות מהשרת, לא מהלקוח — מונע זיוף.
- **`receive-research-results` דורש סוד webhook** ומסרב לפעול אם `N8N_WEBHOOK_SECRET` לא מוגדר (`503`) — ברירת מחדל בטוחה.
- **`analyze-entity` חוסם קלט > 50,000 תווים** ודורש אימות — הגנה מפני credit-drain.
- **פונקציית ה-MCP קוראת בלבד** ומשתמשת ב-token של המשתמש (`supabaseForUser`) — כל שאילתה כפופה ל-RLS שלו.

---

## 5. סדר תיקון מומלץ

### שלב 1 — עצירת דימום נתונים (לפני כל שינוי אחר)
1. **ממצא 3** — לתקן את המשקלים בטריגר ה-DB, או להסיר את חישוב `score_weighted_total` מהטריגר ולהשאיר מקור אמת יחיד. **זו ההחלטה הארכיטקטונית הראשונה שצריך לקבל.**
2. **ממצא 8** — להחליט על מדיניות null אחידה (מומלץ: `NULL` = לא ניתן לחשב, לא 0).

### שלב 2 — יישור הקאנון
3. **ממצא 1** — לעדכן את `analyze-entity` ל-5 הרמות הקנוניות כולל `humanitarian_concern`.
4. **ממצא 2** — להסיר את ה-hard-stop; VETO מסמן ולא עוצר.
5. **ממצא 5** — אוצר מילים אחד ל-verdict ואחד לגיאופוליטיקה, בקובץ טיפוסים משותף.

### שלב 3 — ניקוי חוב טכני
6. **ממצא 4** — להחליט על גורל `useWeightedScoring`: להסיר, או להגדיר במפורש כ"מיון תצוגה" נפרד מהניקוד הקאנוני.
7. **ממצא 6** — אכיפת JSON schema ב-`analyze-entity`.
8. **ממצא 9** — לאמת מול המסד החי ואז להכריע: שער אישור אמיתי, או גישה פתוחה מוצהרת.

### שלב 4 — ליטוש
9. ממצאים 7, 10, 11–17.

---

## 6. היקף הבדיקה ומגבלות

**נבדק לעומק:**
- `src/lib/verdictCalculation.ts`, `src/lib/costCalculator.ts`
- `src/hooks/useWeightedScoring.ts`, `useScoringWeights.ts`, `useVerdictRecalculation.ts`, `useUserRole.ts`
- `src/components/ProtectedRoute.tsx`, `src/components/dashboard/ScoringControls.tsx`
- `src/pages/settings/WeightsSettingsPage.tsx`
- `supabase/functions/`: `analyze-entity`, `trigger-research`, `receive-research-results`, `mcp`
- 8 מתוך ~30 קבצי migration

**לא נבדק:**
- ~130 קומפוננטות UI (`src/components/detail/`, `dashboard/`, `research/`)
- ~22 migrations נוספים — **כולל מדיניות ה-RLS על `influencers` / `organizations` / `intelligence_reports` עצמן** (פער משמעותי: לא אומת מי יכול לקרוא ולכתוב את נתוני המחקר)
- `schemaMapper.ts`, `researchFieldMapping.ts`, `importSchema.ts`, `validationSchemas.ts`
- שאר ה-Edge Functions (`seed-*`, `migrate-influencers`, `toggle-maintenance`, `track-visit`, `notify-*`)
- `.env` (חסום לקריאה)

**מגבלה טכנית:** ה-Supabase של הפרויקט (`jtofvaetlpfmlzgiqwzj`) שייך לחשבון/ארגון שאינו מחובר ל-MCP בסשן זה, ולכן **לא ניתן היה להריץ את מנתח האבטחה של Supabase** (`get_advisors`) — כלי שהיה מאתר אוטומטית טבלאות ללא RLS ופערי מדיניות. מומלץ לחבר גישה זו ולהריץ אותו.

---

*הופק ע"י Claude Code. כל הממצאים אומתו מול קוד המקור בפועל ב-commit הנקוב.*
