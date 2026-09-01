# חיבור MCP לפרויקט Lovable

מסמך הגדרה לחיבור בין Lovable לבין קלוד דרך Model Context Protocol.
יש שני כיווני חיבור שונים לחלוטין — חשוב לא לבלבל ביניהם.

---

## כיוון א׳ — Lovable כשרת MCP (קלוד שולט בפרויקט)

השרת הרשמי של Lovable מאפשר לסוכן AI ליצור פרויקטים, לשלוח הודעות לסוכן
של Lovable, לקרוא קוד, לשאול את מסד הנתונים ולפרוס — בלי לצאת מקלוד.

| פרט | ערך |
|---|---|
| כתובת השרת | `https://mcp.lovable.dev` |
| טרנספורט | Streamable HTTP |
| אימות | OAuth 2.1 בלבד (אין API key) |
| OAuth Client ID | `6d465f583e1e4ce5801b1616f735670c` — נדרש רק ללקוחות שאינם קלוד |
| זמינות | כל המסלולים של Lovable |

### 1. claude.ai ו-Claude Code בדפדפן (המסלול המומלץ)

מוסיפים את Lovable כ-Connector דרך ההגדרות של claude.ai:

`Settings → Connectors → Browse connectors → Lovable → Connect`

אישור ה-OAuth נפתח בדפדפן. אחרי החיבור צריך גם לוודא שה-connector
מסומן כפעיל בצ׳אט עצמו (תפריט ה-connectors של השיחה) — חיבור ברמת
החשבון לא מפעיל אותו אוטומטית בכל שיחה.

> סשן ריצה מרוחק (Claude Code on the web) לא יכול להשלים OAuth בעצמו,
> כי אין בו דפדפן. לכן החיבור חייב להתבצע מצד המשתמש בממשק claude.ai.

### 2. Claude Code CLI מקומי

```sh
claude mcp add --transport http lovable https://mcp.lovable.dev
```

ואז מריצים `/mcp` בתוך הסשן ומאשרים את ה-OAuth בדפדפן.
לחלופין, בגרסאות 2.1.186 ומעלה: `claude mcp login lovable`.

בחירת scope:

```sh
claude mcp add --transport http lovable --scope local   https://mcp.lovable.dev  # ברירת מחדל, אישי
claude mcp add --transport http lovable --scope user    https://mcp.lovable.dev  # בכל הפרויקטים שלך
claude mcp add --transport http lovable --scope project https://mcp.lovable.dev  # נשמר ב-.mcp.json ומשותף לצוות
```

### 3. הגדרה ידנית בקבצי קונפיגורציה

Claude Desktop (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "lovable": {
      "type": "http",
      "url": "https://mcp.lovable.dev"
    }
  }
}
```

Cursor / Windsurf (`~/.cursor/mcp.json`) — כאן כן צריך את ה-Client ID:

```json
{
  "mcpServers": {
    "lovable": {
      "type": "http",
      "url": "https://mcp.lovable.dev",
      "auth": { "CLIENT_ID": "6d465f583e1e4ce5801b1616f735670c" }
    }
  }
}
```

VS Code — אותו מבנה, תחת המפתח `servers` במקום `mcpServers`.

Codex CLI:

```sh
codex mcp add lovable --url https://mcp.lovable.dev
codex mcp login lovable
```

### בדיקת תקינות אחרי החיבור

1. להריץ `/mcp` (או לפתוח את תפריט ה-connectors) ולוודא סטטוס `connected`.
2. לבקש מהסוכן להריץ `list_workspaces` ואז `list_projects` — אם חוזרת
   רשימת הפרויקטים, החיבור והאימות תקינים.
3. `get_project` על הפרויקט הרלוונטי כדי לאמת שהוא נגיש.

---

## כיוון ב׳ — MCP בתוך Lovable (Lovable כלקוח)

זה הכיוון ההפוך: חיבור שרתי MCP חיצוניים אל תוך הצ׳אט של Lovable, כדי
שסוכן הבנייה של Lovable יוכל להשתמש בהם בזמן פיתוח.

הנתיב בממשק: בתוך הפרויקט → פאנל ה-Integrations / Connectors → בוחרים
קונקטור מוכן, או `Custom` / `New MCP server` לשרת משלך. בהגדרה ידנית
מזינים את כתובת השרת ובוחרים שיטת אימות (בדרך כלל OAuth).

Lovable מפרידה בין שני משטחי אינטגרציה, וכדאי לדעת במה בוחרים:

- **App Connectors** — אינטגרציות שרצות באפליקציה עצמה בזמן ריצה.
- **Chat Connectors (MCP)** — כלים שזמינים לסוכן של Lovable בזמן בנייה בלבד.

---

## אזהרות תפעוליות

- `create_project` ו-`send_message` צורכים קרדיטים של Lovable בכל קריאה.
  כלים לקריאה בלבד לא עולים כלום.
- `deploy_project` מפרסם כתובת ציבורית — לא להריץ בלי כוונה מפורשת.
- `query_database` מריץ SQL ישירות מול מסד הנתונים של הפרויקט. להימנע
  מהרצת שאילתות כתיבה דרך הסוכן בלי בדיקה.
- כדאי להשאיר את מצב ההרשאות כך שכלי כתיבה ידרשו אישור ידני, לפחות
  עד שמתייצבים על תהליך עבודה.

---

## מצב הרפו הזה

נכון לכתיבת המסמך, `Sunflower-Intelligence` ריק — אין קומיטים ואין
ברנצ׳ים ב-remote. כלומר סנכרון GitHub של פרויקט ה-Lovable עדיין לא בוצע.
אם המטרה היא שהקוד של הפרויקט יישב כאן, צריך להפעיל ב-Lovable את
`GitHub → Connect` ולכוון אותו לרפו הזה. זהו חיבור נפרד מ-MCP.

---

## מקורות

- https://lovable.dev/mcp
- https://docs.lovable.dev/integrations/lovable-mcp-server
- https://github.com/lovablelabs/mcp
- https://code.claude.com/docs/en/mcp
