# CLAUDE.md — Sunflower Intelligence / Evinature BD

מסמך זיכרון לסשנים של Claude Code בריפו הזה. קרא אותו לפני שאתה נוגע במשהו.

---

## 1. מה זה הריפו הזה

`itayarieli-ctrl/Sunflower-Intelligence` — ריפו **תיעוד ותשתית**, לא ריפו הקוד של האפליקציה.

| קובץ | תוכן |
|---|---|
| `handoff-evinature-bd-intelligence-קלוד.md` | מסמך המסירה המלא (18k שורות) — **מקור האמת לקאנון** |
| `דוח-בדיקת-קוד-sunflower-קלוד.md` | דוח בדיקת קוד, 17 ממצאים, ספטמבר 2026 |
| `skills-קלוד/sovereign-architect/` | סקיל שהוטמע כ-project knowledge ב-Lovable |
| `אופטימיזציית-קרדיטים-קלוד.md`, `הגדרת-MCP-קלוד.md`, `מבנה-חיבור-חסכוני-קלוד.md` | תשתית Claude/Lovable/MCP |

**קוד האפליקציה נמצא ב-Lovable, לא כאן.**

---

## 2. הפרויקט

**Evinature** — חברת נוטרסוטיקלס ישראלית. מוצר: **CurQD®**, פרוטוקול כורכומין ל-UC/Crohn's עם RCT אמיתי. מייסדים: פרופ' Shomron Ben-Horin (שיבא), Nir Salomon.

**המערכת:** פלטפורמת BD Intelligence שממכנת הערכת שותפים פוטנציאליים (משפיענים / ארגונים / מדינות) בעולם ה-IBD.
עיקרון מנחה: **המערכת נותנת ציונים אובייקטיביים → המשתמש קובע משקלים → ההחלטה הסופית אנושית.**

### היכן מה נמצא

| רכיב | מיקום | מזהה |
|---|---|---|
| קוד האפליקציה | Lovable | project `d47c4603-563b-4790-a151-b899fd9b4ee5` (`gemini-genesis-engine`) |
| Workspace | Lovable | `E2yEcBUG95ByH0SbuHTG` ("Itay's Lovable") |
| מסד נתונים + Edge Functions | Supabase | ref `jtofvaetlpfmlzgiqwzj` |
| Backend workflows | n8n | ארכיטקטורה מוגדרת (v3.1), **workflow עדיין לא נבנה** |
| תיעוד | GitHub | הריפו הזה |

⚠️ קיים גם פרויקט Lovable שני — `cfce4136-...` ("Remix of...") — **ישן, לא בשימוש.** אל תיגע בו.

⚠️ ה-Supabase של הפרויקט **אינו** בחשבון המחובר ל-MCP כברירת מחדל. `list_projects` מחזיר 4 פרויקטים אחרים. ללא חיבור נכון: אי אפשר להריץ migrations או `get_advisors`.

---

## 3. הקאנון — אל תמציא מחדש

מקור האמת: **Project Instructions v5.0**, משורשר בתוך מסמך המסירה. כל סטייה ממנו היא באג עד שיוכח אחרת.

### ארכיטקטורת ה-pipeline (v2)

```
Stage 0A (מקבילי):  Perplexity Deep + Gemini 3 Flash    → זהות + קשר + סנטימנט מוקדם
Stage 0C:            Claude Opus                         → סינתזת זהות (Iron Laws)
Stage 1 (מקבילי):    Perplexity Deep + Perplexity Pro + Gemini 3 Pro (JSON schema)
Stage 1C:            Claude Opus                         → סינתזת מחקר
Stage 2:             Claude Opus                         → ציונים + verdict + flags
```
עלות ~$2.41 לישות · ~24 דקות · 9 webhooks.

### 6 הדימנשנים והמשקלים (החלטה 1)

| דימנשן | משקל |
|---|---|
| Relevance | 25% |
| Reach | 20% |
| Partnership Readiness | 15% |
| Competition | 15% |
| Brand Safety | 15% |
| Strategic Fit | 10% |

המשקלים ניתנים לעריכה ע"י המשתמש, **חייבים להסתכם ל-100**.

### הסולם הגיאופוליטי — 5 רמות, ואין אחרות

```
pro_israel → neutral → humanitarian_concern → suspected → confirmed_anti
   50/50      25/50          5/50              0/50         0/50 + VETO
```

- **`humanitarian_concern` חייב להתקיים** (החלטה 3). זו רמת הביניים למי שמפרסם תוכן הומניטרי על עזה **בלי** BDS/Hamas. מפעיל דגל 🟠 SENSITIVE ודורש סקירה אנושית.
- **`unknown` בוטל** (החלטה 4) — מוזג לתוך `neutral`. אל תחזיר אותו.
- Brand Safety מתחלק 50/50: גיאופוליטי (0-50) מול שאר (0-50: professionalism 20 / controversy 20 / legal 10).

### VETO לא עוצר את המחקר (החלטה 5)

`confirmed_anti` או `HOSTILE` → המערכת **ממשיכה** את כל השלבים, מפיקה דוסייה מלא ומנקדת הכול. רק ה-verdict נקבע ל-`HARD_NO` וה-UI מציג אזהרה בולטת.
הסיבה: סקירה אנושית עם ראיות, יכולת override, ו-audit trail ל-compliance.
**Hard-stop אחרי VETO נוסה ונזנח.** אל תציע אותו שוב.

### Verdicts ו-Drivers

```
verdict:  HARD_NO | NO | CAUTION | CONDITIONAL | CLEAR_YES | PRIORITY_PARTNER
driver:   geopolitical | company_sentiment | business | combined
```
`company_sentiment` הוא driver מלא (החלטה 8) — למי שאמר משהו שלילי על Evinature/CurQD. שונה מ-competitor.

### Iron Laws לסינתזה (החלטה 10)

1. **Hierarchy of Truth** — מקורות ראשוניים > אחרים; עדכני > ישן.
2. **Zero Averaging** — אסור לממצע מספרים סותרים. 500K ו-600K ≠ 550K. בחר מקור, או הצג טווח, או סמן conflict.
3. **Anti-Circular Reporting** — 3 מקורות שמצטטים כתבה אחת = נקודת נתון אחת.
4. **Evidence Tracking** — כל ערך מצביע על URL ספציפי.
5. **Completeness Over Speed**.

### עוד החלטות נעולות

- **Competition** — סולם קופצני 100/70/10/0, לא רציף (החלטה 7).
- **Company/Product Awareness = 30 מתוך 100** ב-Relevance (החלטה 6) — מודעות = מחזור מכירה קצר.
- **פלט תלת-חלקי** לכל שלב: Structured JSON + Narrative Markdown + Data Quality Block (החלטה 11).
- **Absence of data ≠ safety** — אם לא נמצא מידע, מקסימום 6-7/10 עם MEDIUM confidence, לא 10/10.

---

## 4. מה חסר (נכון לספטמבר 2026)

הפרומפטים האלה **מעולם לא נכתבו** — הם לא "אבדו":

- ❌ **פרומפט Stage 2** (ניקוד + verdict) — הכי דחוף. הכי קרוב: `comprehensive_research_prompt_v1_1_CLEAN.md` בתוך מסמך המסירה, אך זו טיוטה שקדמה לפיצול לשלבים.
- ❌ **3 פרומפטים של Stage 1** (Perplexity Deep / Perplexity Pro / Gemini 3 Pro + JSON schema).
- ❌ **פרומפט Stage 1C**.
- ❌ n8n workflow בנוי בפועל.
- ❌ טסטים End-to-End.

קיים ומוכן: Project Instructions v5.0, Stage 0A v11.4, Stage 0C Fusion, `stage1_briefing_schema_v1.json`, ארכיטקטורת n8n v3.1.

**שאלה פתוחה חשובה:** מיפוי סולם 0-10 (Stage 0A) → 0-100 (v5.0). ההמלצה במסמך: חישוב-מחדש מ-sub-components, לא כפל ב-10.

---

## 5. באגים ידועים — אל תחקור מחדש

פירוט מלא ב-`דוח-בדיקת-קוד-sunflower-קלוד.md`. תמצית הקריטיים:

| # | באג | מיקום |
|---|---|---|
| 1 | חסר `humanitarian_concern`, עדיין קיים `unknown`, `confirmed` במקום `confirmed_anti` | `supabase/functions/analyze-entity/index.ts` |
| 2 | VETO עוצר את המחקר (סותר החלטה 5) | `analyze-entity` |
| 3 | טריגר DB מחשב משקלים **שגויים** (reach .15, partnership .20, competition .10, strategic .15) ומתעלם ממשקלי המשתמש; דורס את הלקוח | `calculate_weighted_score`, migration `20260129153137` |
| 4 | 3 מערכות ניקוד מקבילות; הדשבורד מדרג לפי הישנה | `useWeightedScoring.ts` ↔ טריגר DB ↔ `verdictCalculation.ts` |
| 5 | 3 אוצרות-מילים ל-verdict, 3 לגיאופוליטיקה | פרומפט / DB / לקוח |

**התובנה המרכזית:** הקוד הישן (מגרסת ה-HTML/Gemini המקורית) לא הוסר — הוא רץ **במקביל** לחדש. זה שורש רוב הבאגים.

---

## 6. איך לעבוד כאן

### כלים
- **קריאת קוד:** `mcp__Lovable__read_file` / `list_files` עם `ref` = commit sha מ-`get_project`.
- **שינוי קוד:** `mcp__Lovable__send_message` — סוכן ה-Lovable כותב. `plan_mode=true` לדיון לפני. `get_diff` לביקורת אחרי. **צורך קרדיטים.**
- **DB:** דורש חיבור Supabase לחשבון הנכון. בלעדיו — רק דרך סוכן ה-Lovable.

### כללי בית
- **אל תשנה קבצים קיימים בריפו הזה.** אם המקור לא ודאי — צור קובץ חדש.
- **שמות קבצים חדשים חייבים להכיל "קלוד" בדיוק פעם אחת.** (חריג: `CLAUDE.md` — שם מחייב פונקציונלית.)
- ענף פיתוח: לפי מה שהוגדר בסשן. `git push origin HEAD`.
- אם `git push` נחסם ע"י ה-classifier — נסה שוב, החסימה לרוב זמנית.

### לפני שאתה מציע שינוי ארכיטקטוני
קרא את **חלק ד' במסמך המסירה** ("מה נוסה ונזנח"). 8 כיוונים כבר נדונו ונדחו:
provider יחיד ל-Stage 1 · hard-stop אחרי VETO · סולם רציף ל-Competition · הפרדת `unknown` · averaging · multi-source scoring בלי synthesis · Gemini Deep Research Agent · renaming של sections.

### הערה על מודלים
Claude Sonnet נטה לשנות שמות של sections בפלט (`=== 2. CONTACTS ===` → `PROFESSIONAL STANDING`), מה ששבר את ה-parser. יש אזהרה מפורשת נגד זה ב-Stage 0A v11.4. שלבי הסינתזה (0C, 1C, 2) רצים על Opus בכוונה — שם הטעויות הכי יקרות.

---

## 7. מוסכמות תיעוד

- 🚧 טיוטה = לא סופי · ✅ קנוני = מקור אמת.
- העברית היא שפת העבודה מול המשתמש. הקוד, הפרומפטים והקומיטים באנגלית.
