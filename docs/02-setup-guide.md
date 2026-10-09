# 02 · מדריך הקמה, צעד אחר צעד

כל המערכת רצה בענן, ואין מה להתקין על המחשב. הזמן המשוער להקמה מלאה הוא 60–90 דקות.

## 0. חשבונות
| שירות | למה | הערה |
|---|---|---|
| n8n Cloud | מריץ את 10 התהליכים והסוכנים | נותן כתובת HTTPS ציבורית, ולכן טלגרם ו-Webhooks עובדים בלי מנהרה (Tunnel) |
| Airtable | בסיס הנתונים | צריך Personal Access Token עם ההרשאות `data.records:read`, `data.records:write` ו-`schema.bases:read` |
| OpenAI | מודל הצ'אט והמודל של ה-Embeddings | מפתח API עם יתרה |
| Telegram | שני בוטים מ-@BotFather | בוט אחד למנהל ובוט אחד ללקוחות |
| Google | Gmail ו-Drive | OAuth דרך n8n |
| Lovable או Base44 | ממשק הניהול | מספיק לבחור אחת מהשתיים |

## 1. Credentials ב-n8n
**Credentials → Add credential.** יוצרים את החיבורים הבאים. השמות המומלצים זהים לשמות שמופיעים בקובצי התהליכים, כך שאחרי הייבוא קל לזהות איזה Credential לבחור בכל צומת:

| Credential | סוג |
|---|---|
| `Airtable - Kilohertz` | Airtable Personal Access Token |
| `OpenAI - Kilohertz` | OpenAI |
| `Telegram - Manager Bot` | Telegram (הטוקן של בוט המנהל) |
| `Telegram - Customer Bot` | Telegram (הטוקן של בוט הלקוחות) |
| `Gmail - Kilohertz Sales` | Gmail OAuth2 |
| `Google Drive - Kilohertz Docs` | Google Drive OAuth2 |
| `App Webhook - Header Auth` | Header Auth. השם הוא `x-api-key`, והערך הוא מחרוזת אקראית ארוכה |

> הסודות נשמרים רק ב-Credentials של n8n. הם לא נכנסים לקוד, לריפו או לאפליקציה.

### יצירת Personal Access Token (אסימון גישה אישי) ב-Airtable
1. נכנסים ל-[airtable.com/create/tokens](https://airtable.com/create/tokens) ולוחצים **Create token**.
2. נותנים לאסימון שם מזהה, למשל `n8n - Kilohertz`.
3. ב-**Scopes** (הרשאות) מוסיפים שלוש: `data.records:read`, `data.records:write`, `schema.bases:read`.
4. ב-**Access** בוחרים רק את הבסיס של קילוהרץ, לא את כל הבסיסים בחשבון. לכן כדאי ליצור קודם את הבסיס לפי סעיף 2, ורק אחר כך את האסימון.
5. לוחצים **Create token** ומעתיקים את האסימון. הוא מוצג פעם אחת בלבד; אם לא הועתק, יוצרים אסימון חדש.
6. ב-n8n פותחים את ה-Credential `Airtable - Kilohertz` (מסוג Airtable Personal Access Token), מדביקים את האסימון ושומרים. n8n בודק את החיבור בשמירה.

## 2. בסיס Airtable
1. יוצרים בסיס חדש בשם "Kilohertz ERP-AI".
2. יוצרים את 4 טבלאות הליבה לפי `data/schema.json` (`core: true`), **עם שמות שדות זהים בדיוק**:
   - **Invoices**: InvoiceNumber · DocType · CustomerId · CustomerName · CustomerEmail · IssueDate · Amount · VatRate · VatAmount · Total · Status · PdfUrl · Notes · Created
   - **Leads**: Name · Email · Phone · Company · Source · Interest · Status · LastContacted · Notes · Created
   - **Products**: Sku · Name · Category · Price · Description · InStock
   - **Tasks**: Title · Status · Priority · DueDate · Source · Created
3. **שני כללים שחוסכים תקלות:**
   - השדה `Created` הוא מסוג **Created time** בטבלאות Invoices, Leads ו-Tasks. בלעדיו הטריגרים לא עובדים.
   - שדות `Status` הם **Single line text**, ולא Single select.
4. מייבאים את `data/products.csv` לטבלה Products (**Add or import → CSV file**, ומוודאים שכל עמודה ממופה לשדה בעל אותו שם).

**למה דווקא המוצרים מיובאים מראש?** העסק מתחיל בלי נתוני פעילות: הטבלאות Invoices, Leads ו-Tasks נשארות ריקות ומתמלאות רק מהשימוש במערכת, כפי שהמטלה דורשת. קטלוג המוצרים הוא נתוני אב (Master Data - נתוני בסיס קבועים) ולא נתוני פעילות: סוכן שירות הלקוחות וסוכן המנהל צריכים אותו כדי לענות על שאלות מוצר דרך RAG (WF07 בונה ממנו את המאגר הווקטורי). לכן הייבוא שלו הוא חלק מההקמה.

## 3. ייבוא התהליכים
עבור כל קובץ ב-`workflows/`: **Create Workflow → ⋯ → Import from File**. אפשר גם להעתיק את תוכן הקובץ ולהדביק על הקנבס.

אחרי הייבוא, בכל תהליך:
1. בכל צומת Airtable בוחרים את הבסיס שלכם, במקום `appXXXXXXXXXXXXXX`, ואז בוחרים מחדש את הטבלה.
2. בכל צומת שמסומן באזהרה, פותחים את הצומת ובוחרים ידנית את ה-Credential שיצרתם בסעיף 1. השם שמופיע בצומת אחרי הייבוא מציין איזה Credential מתאים לו (למשל `Telegram - Manager Bot` ב-WF09). החיבור לא נוצר אוטומטית, כי בחשבון שלכם יש מזהים אחרים.
3. ב-**WF09**, בצומת `Is Owner?`: מחליפים את `123456789` ב-Chat ID של הבעלים. אפשר לברר אותו דרך הבוט @userinfobot.
4. ב-**WF08**: כדאי לבחור תיקייה ייעודית ב-Drive, למשל "Kilohertz - מסמכים". ברירת המחדל היא שורש הדרייב.

## 4. מילוי המאגר הווקטורי
מריצים ידנית את **WF06** (מדיניות) ואחריו את **WF07** (מוצרים), בלחיצה על **Execute workflow**.

> המאגר יושב בזיכרון של n8n. אחרי כל הפעלה מחדש של n8n צריך להריץ אותם שוב.

## 5. הפעלה
מפעילים (Activate / Publish) את WF01, WF03, WF04a, WF04b, WF05, WF08, WF09 ו-WF13.

> **WF01 מגיב רק לרשומות חדשות.** הטריגר סורק לפי השדה `Created`, ולכן עריכה של רשומה קיימת לא מפעילה אותו מחדש. חשבונית שסומנה `Invalid` (הסיבה כתובה בשדה Notes) מתקנים כך: מזינים אותה מחדש כרשומה חדשה עם הנתונים הנכונים, או משכפלים את השורה ב-Airtable (**Duplicate record**), מתקנים את העותק ומחזירים בו את Status ל-New. WF01 יקלוט את העותק כרשומה חדשה ויחשב אותו מחדש. הרשומה הפסולה נשארת לתיעוד.

## 6. האפליקציה
1. מעתיקים את ה-**Production URL** של WF13.
2. ב-Lovable או ב-Base44 מדביקים את הפרומפט מתוך `app/build-prompt.md`.
3. שומרים את הכתובת ואת המפתח כ-Secrets בצד השרת: `N8N_APP_WEBHOOK_URL` ו-`N8N_APP_API_KEY`. **לא** בקוד שרץ בדפדפן.
4. בדיקה: הדשבורד נטען, טופס יוצר רשומה ב-Airtable, והצ'אט מחזיר תשובה.

כדי לראות איך הממשק אמור להיראות לפני הבנייה: `app/index.html` הוא לקוח ייחוס שעובד במצב הדגמה. פותחים אותו בדפדפן.

## 7. בדיקה מקצה לקצה
ראו [04-testing.md](04-testing.md).
