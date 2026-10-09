# פרומפט הבנייה לאפליקציית הניהול

המסמך הזה מכיל את מה שמדביקים ב-Lovable או ב-Base44 כדי לבנות את ממשק הניהול של קילוהרץ. הפרומפט עצמו כתוב באנגלית, כי שתי הפלטפורמות מבינות הנחיות טכניות באנגלית בצורה המדויקת ביותר. הממשק שייווצר יהיה כולו בעברית.

**לפני שמתחילים:**

1. WF13 מיובא ופעיל ב-n8n, ויש בידכם את כתובת ה-Production URL שלו (מסתיימת ב-`/webhook/kilohertz/app`).
2. יש בידכם את ערך המפתח שהוגדר ב-Credential מסוג Header Auth ב-n8n (שם הכותרת: `x-api-key`).
3. את שני הערכים האלה **לא מדביקים בצ'אט הבנייה**. מזינים אותם רק במסך ה-Secrets של הפלטפורמה (פירוט בסעיף "סודות ואבטחה" למטה).
4. מסכי **לקוחות** ו**הזמנות** נבנים כבר עכשיו, אבל קוראים מהטבלאות המורחבות `Customers` ו-`Orders` שב-`data/schema.json`. עד שיוצרים אותן ב-Airtable ומוסיפים אותן לרשימה המותרת בצומת `Route By Action` ב-WF13 (שינוי של שורה אחת, פירוט ב-[`docs/05-app.md`](../docs/05-app.md)), שני המסכים מציגים מצב ריק.

---

## 1. פרומפט הבנייה הראשי

```text
You are building "Kilohertz Admin", a private back-office web app for Kilohertz
Electronics, a small Israeli consumer-electronics retailer. One user (the owner)
will use it every morning to see the state of the business and enter records.

LANGUAGE AND FORMAT
- The entire UI is in Hebrew. Set <html dir="rtl" lang="he">, right-aligned text,
  and mirror the layout (sidebar on the right, icons that imply direction flipped).
- Use a clean font that supports Hebrew well (e.g. Heebo or Assistant).
- Money: Israeli shekel, formatted with Intl.NumberFormat('he-IL',
  { style: 'currency', currency: 'ILS' }), e.g. "‏1,475.00 ₪".
- Dates: display as dd/mm/yyyy. When sending dates to the API, use ISO YYYY-MM-DD.

BACKEND - VERY IMPORTANT
- Do NOT create a database, tables, or any data model of your own.
- The ONLY data source is one HTTPS endpoint (an n8n webhook). Never call
  Airtable or any other service directly.
- The browser must never see the endpoint URL or the API key. Create ONE
  server-side function named "kilohertz-api" that:
    1. reads two secrets: N8N_APP_WEBHOOK_URL and N8N_APP_API_KEY,
    2. accepts a JSON body from the frontend and forwards it unchanged with
       POST to N8N_APP_WEBHOOK_URL, adding the header "x-api-key: <N8N_APP_API_KEY>"
       and "Content-Type: application/json",
    3. returns the JSON response and HTTP status to the frontend as-is,
    4. rejects any request from a user who is not signed in to the app.
- The frontend calls only this server function, never the webhook directly.

API CONTRACT (the server function forwards exactly these bodies)
- List records (max 100 per table):
    request  {"action":"list","table":"Invoices" | "Leads" | "Products" | "Tasks"
              | "Customers" | "Orders"}
    response {"ok":true,"table":"Invoices","records":[{"id":"rec...","fields":{...}}]}
  Customers and Orders are optional extended tables. Until the owner enables
  them on the backend, list returns ok:false for them. In that case do NOT show
  an error: show the Hebrew empty state "הטבלה עדיין לא הופעלה" on that screen.
- Create a record (only these three tables):
    request  {"action":"create","table":"Invoices" | "Leads" | "Tasks","payload":{...fields}}
    response {"ok":true,"id":"rec..."}
  Creating a lead whose Email already exists returns HTTP 409 with
  {"ok":false,"error":"..."} and no record is created. Show the Hebrew message
  "ליד עם המייל הזה כבר קיים" and keep the form open.
- Chat with the business assistant (read-only: it can look up tasks, store
  policies and invoice figures, but cannot create or change records):
    request  {"action":"chat","message":"<text>","sessionId":"<id>"}
    response {"ok":true,"reply":"<text>"}
- Any failure: {"ok":false,"error":"<message>"} with HTTP 400.
  Show the error text in a Hebrew toast; never crash the screen.
- Field names inside "fields" and "payload" are case-sensitive and must be
  used exactly as listed below.

DATA FIELDS
- Invoices: InvoiceNumber, DocType ("חשבונית" | "חשבונית מס" | "קבלה"),
  CustomerId, CustomerName, CustomerEmail, IssueDate, Amount (before VAT),
  VatRate (decimal, 0.18 = 18%), VatAmount, Total, Status
  ("New" | "Pending" | "Invalid" | "Done"), PdfUrl, Notes, Created
- Leads: Name, Email, Phone, Company, Source, Interest,
  Status ("New" | "Contacted" | "Replied" | "Unsubscribed"), LastContacted, Notes, Created
- Products: Sku, Name, Category, Price, Description, InStock (boolean)
- Tasks: Title, Status ("Open" | "Done"), Priority ("נמוכה" | "רגילה" | "גבוהה"),
  DueDate, Source, Created
- Customers: CustomerId, Name, Email, Phone, TaxId, City, Created
- Orders: OrderId, CustomerId, OrderDate, Items, Total (incl. VAT),
  Status ("Received" | "Packed" | "Shipped" | "Delivered" | "Cancelled"), Created
- Any field may be missing from a record; render an empty cell, not "undefined".

SCREENS (right-side navigation, Hebrew labels in parentheses)
1. Dashboard ("לוח בקרה") - four cards, computed in the frontend from list calls:
   - Revenue this month ("הכנסות החודש"): sum of Total for Invoices with
     Status "Done" and IssueDate in the current calendar month.
   - Open invoices ("חשבוניות פתוחות"): count of Invoices whose Status is not "Done",
     with a small breakdown New / Pending / Invalid.
   - Leads by status ("לידים לפי סטטוס"): a simple bar chart of counts per Status.
   - Open tasks ("משימות פתוחות"): count of Tasks with Status "Open", plus a short
     list of the 5 nearest DueDate values.
   Add a "רענון" button. Load the four tables in parallel, once per page load.
2. Invoices ("חשבוניות") - table with columns: InvoiceNumber, DocType,
   CustomerName, IssueDate, Amount, VatAmount, Total, Status, Document.
   The Document column shows a link "פתיחת מסמך" that opens PdfUrl in a new tab
   when PdfUrl exists, otherwise a grey "טרם הופק".
   Status is a colored badge: New grey, Pending amber, Invalid red, Done green.
3. Leads ("לידים") - columns: Name, Company, Email, Phone, Source, Interest,
   Status, LastContacted, Created.
4. Products ("מוצרים") - columns: Sku, Name, Category, Price, InStock (check icon).
   Read-only screen.
5. Tasks ("משימות") - columns: Title, Priority, DueDate, Status, Source.
   Highlight overdue open tasks in red.
6. Customers ("לקוחות") - columns: CustomerId, Name, Email, Phone, TaxId, City.
   Read-only screen.
7. Orders ("הזמנות") - columns: OrderId, CustomerId, OrderDate, Items, Total,
   Status. Status is a colored badge: Received grey, Packed amber, Shipped blue,
   Delivered green, Cancelled red. Read-only screen.
   Screens 6 and 7 load their data only when opened (not on the Dashboard).
Every table screen has: a free-text search box that filters across all visible
columns, a Status (or Category) filter dropdown, sorting by clicking a header,
newest first by default, and an empty state in Hebrew.

CREATE FORMS (open from a "+ חדש" button in a side panel)
- New lead: Name*, Email*, Phone, Company, Source (default "App"), Interest, Notes.
  Send Status "New".
- New invoice: DocType* (dropdown of the three values), CustomerId* (pattern
  CUST-0000), CustomerName*, CustomerEmail, IssueDate* (date picker, default today),
  Amount* (positive number, before VAT), Notes. Send Status "New".
  Do NOT send InvoiceNumber, VatRate, VatAmount, Total or PdfUrl; the automation
  fills them. Show a note: "המע״מ והמספור יחושבו אוטומטית תוך כדקה".
- New task: Title*, Priority (default "רגילה"), DueDate, Source fixed to "App".
  Send Status "Open".
- Validate required fields and email format before sending. After a successful
  create, close the panel, show a success toast, and reload that table.
- Each create starts a backend automation, so tell the user what happens next
  in the success toast:
    invoice -> "החשבונית נקלטה. המע״מ, המספר והמסמך יופיעו תוך כמה דקות"
    lead    -> "הליד נקלט ויקבל פנייה ראשונה מצוות המכירות"
    task    -> "המשימה נוספה"
  An invoice row may later show Status "Invalid": the reason is in Notes.
  Show Notes as a tooltip on the Invalid badge.

CHAT PANEL
- A collapsible panel on the left edge titled "העוזר של קילוהרץ".
- Generate one sessionId (random UUID) when the app loads and keep it for the
  browser session.
- On send: append the user message, show a typing indicator, call the chat
  action, then append "reply" as the assistant message. Messages are RTL.
- Disable the send button while waiting. Timeout after 30 seconds with a Hebrew
  error message.

STYLE
- Calm, professional look: white background, dark slate text, one accent color
  (deep teal). Cards with soft shadows, generous spacing. Responsive down to
  tablet width.
```

---

## 2. פרומפטים לשיפור (שלב אחר שלב)

אחרי שהגרסה הראשונה עולה, ממשיכים בשינויים קטנים, כל אחד בנפרד. כך קל לבדוק כל שינוי ולחזור אחורה אם משהו נשבר.

1. `Check every screen for RTL issues: table column order should start from the right, number and currency cells should stay left-to-right inside the cell, and the date picker should open in Hebrew with Sunday as the first day of the week.`
2. `On the Dashboard, make the four cards a 2x2 grid on desktop and a single column on narrow screens. Add a subtitle under "הכנסות החודש" showing the month name in Hebrew.`
3. `In the Invoices screen, add a filter by DocType next to the Status filter, and a summary row at the bottom showing the sum of Amount, VatAmount and Total for the rows currently visible.`
4. `In the Leads screen, add quick filter chips for each Status with the count in each chip, and make the Email cell a mailto link.`
5. `Show a skeleton loader while a list call is in progress, and if the call returns ok:false show the error text inside the table area with a "נסה שוב" button.`
6. `In the chat panel, add three suggested questions as clickable chips when the conversation is empty: "אילו משימות פתוחות בעדיפות גבוהה?", "מה מדיניות ההחזרות?", "איך לקוח מבקש עותק של חשבונית?".`
7. `After creating an invoice, poll the Invoices list every 20 seconds for up to 3 minutes and update that row's Status badge in place, so I can watch it move from New to Pending to Done.`
8. `Add a small footer with the app version and the text "נתונים מוצגים: עד 100 רשומות לכל טבלה".`

---

## 3. סודות ואבטחה

- **שני סודות בלבד:** `N8N_APP_WEBHOOK_URL` ו-`N8N_APP_API_KEY`. מזינים אותם במסך הסודות של הפלטפורמה (ב-Lovable: Cloud ואז Secrets, לשימוש ב-Edge Function; ב-Base44: הגדרות האפליקציה ואז Secrets / Environment variables, לשימוש בפונקציית Backend).
- **אף מפתח לא מגיע לדפדפן.** אחרי הבנייה פותחים את כלי המפתחים בדפדפן (לשונית Network), מבצעים פעולה, ומוודאים שהבקשה יוצאת לפונקציית השרת בלבד ושאין בה כותרת `x-api-key`.
- **אין Personal Access Token (אסימון גישה אישי) של Airtable באפליקציה.** הגישה ל-Airtable קיימת רק ב-n8n.
- **האפליקציה פרטית.** מפעילים התחברות (חשבון אחד של נועה), ופונקציית השרת דוחה בקשות ממשתמש לא מחובר. אחרת כל מי שמגיע לכתובת האפליקציה יכול להשתמש בה כמתווך ל-n8n.
- **רוטציה (החלפת מפתח):** אם המפתח נחשף, מחליפים אותו ב-Credential של WF13 ב-n8n ובסוד של האפליקציה. אין צורך לבנות מחדש.
- **לא להדביק סודות בצ'אט הבנייה.** מה שנכתב בצ'אט נשמר בהיסטוריית הפרויקט.

---

## 4. רשימת בדיקות קבלה

- [ ] כל הממשק בעברית, מיושר לימין, סכומים ב-₪ ותאריכים ב-dd/mm/yyyy.
- [ ] הדשבורד נטען ומציג ארבעה כרטיסים; "הכנסות החודש" שווה לסכום ידני של Total בחשבוניות Done מהחודש הנוכחי ב-Airtable.
- [ ] מסכי החשבוניות, הלידים, המוצרים והמשימות נטענים, והחיפוש, הסינון והמיון עובדים.
- [ ] מסכי הלקוחות וההזמנות מציגים "הטבלה עדיין לא הופעלה" כל עוד הטבלאות לא הופעלו, ונתונים אחרי שהופעלו.
- [ ] בעמודת המסמך מופיע קישור שנפתח בדרייב לכל חשבונית שיש לה PdfUrl.
- [ ] יצירת ליד מהטופס יוצרת רשומה ב-Leads עם Status = New.
- [ ] יצירה חוזרת של ליד עם אותו מייל מציגה "ליד עם המייל הזה כבר קיים", ולא נוצרת רשומה שנייה.
- [ ] יצירת חשבונית מהטופס יוצרת רשומה, ותוך כמה דקות WF01 ו-WF08 משלימים מע"מ, מספר וקישור.
- [ ] יצירת משימה מופיעה ב-Tasks עם Status = Open.
- [ ] הצ'אט מחזיר תשובה בעברית, ושאלה שנייה באותו סשן מתייחסת לשאלה הראשונה.
- [ ] בלשונית Network אין כתובת webhook ואין מפתח `x-api-key`.
- [ ] בקשה עם table שלא ברשימה המותרת (למשל `Suppliers`) מציגה הודעת שגיאה ולא שוברת את המסך.
- [ ] כניסה לאפליקציה ממכשיר אחר בלי התחברות נחסמת.
