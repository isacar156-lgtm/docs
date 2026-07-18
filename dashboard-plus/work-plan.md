# GANI ADS Dashboard — תוכנית עבודה · יולי 2026

> **מסמך עצמאי לשיחה/מודל אחר, ללא הקשר מוקדם.** נתיבים יחסיים לשורש הריפו `gani-ads`.
> קוד/שדות באנגלית כמו ב-codebase; הסברים בעברית. עבוד שלב-שלב, קומיט אחרי כל משימה.
> נוצר מתוך סריקת קוד מלאה ב-2026-07-18 (קומיט `0912b4f`). כל שורה שמצוינת אומתה מול הקוד.

---

## 0. הקשר טכני (לקרוא פעם אחת)

- **Backend:** FastAPI, שורש `gani_agent/dashboard/`, שרת `server.py`, ראוטרים ב-`routers/`, שירותים ב-`services/`. כל endpoint חדש חייב `Depends(verify_api_key)` מ-`dashboard.auth`.
- **Guardian Agent:** `gani_agent/rules_engine.py` + `start.py` (סקדולר) + `bot_listener.py` (טלגרם) + `pending_actions.py`.
- **Frontend:** React 18 + Vite + Tailwind, עברית RTL, dark mode. שורש `gani_agent/dashboard/frontend/src/`. טאבים ב-`tabs/`, ניווט `components/Sidebar.jsx`, ראוטינג טאבים ב-`App.tsx` (`_validTabs`).
- **Cache/DB:** SQLite דרך `dashboard/cache.py`, DB ב-`/data` על Railway. WAL.
- **קונפיג עסקי:** `gani_agent/config.py` — `CPA_TARGET=70`, `CPA_KILL=175`, `GROSS_MARGIN=190`, `WINNERS_ROAS_THRESHOLD=8.0`, `SELLING_PRICE=330`.
- **עיקרון BellaPOINT:** המדד הוא **רווח שולי**, לא ROAS. NC-ROAS + MER לפני כל החלטה. CPA מקסימלי = מרווח גולמי × 0.5. "95% מהקריאייטיבים לא יעבדו".
- **גארדריילים לכל שינוי:** לא להאט את `/api/creatives`; לא לשבור RTL/dark-mode; ספים מ-`config`, לא hardcode; טסטים קיימים חייבים להמשיך לעבור (`pytest`, `npm run lint`).

---

## שלב 1 — תיקוני מתמטיקה ודאטה (קריטי, לעשות ראשון)

### 1.1 גרף טרנד קטום ל-25 ימים
- **קובץ:** `gani_agent/fb_api.py:305-327` (`fetch_account_daily_trends`)
- **בעיה:** משתמש ב-`_get` בלי `limit` ובלי מעקב paging — Meta מחזירה 25 שורות ברירת-מחדל, אז טווחי 30/60/90 יום קטומים.
- **תיקון:** להשתמש ב-`_get_all` (קיים באותו קובץ, שורה 20) + `"limit": 100` בפרמטרים; אם התשובה מכילה `"error"` — להעלות חריגה במקום להחזיר גרף ריק.
- **קבלה:** בקשת 90 יום מחזירה ~90 שורות; טסט יחידה עם paging מדומה של 2 עמודים.

### 1.2 NC-ROAS מומצא ב-Margin Lab
- **קובץ:** `gani_agent/dashboard/services/margin_elasticity.py:87` (`nc_roas_proxy`)
- **בעיה:** `nc_roas_proxy = nc_roas_account * spend_weight` — חסר משמעות; מעניש קונספטים קטנים.
- **תיקון:** למחוק את השדה מהפלט ומה-UI (`tabs/MarginLabTab.jsx`). להציג במקומו את ה-NC-ROAS החשבוני פעם אחת בכותרת הטאב עם תווית "רמת חשבון". (תחליף אמיתי פר קריאייטיב — שלב 3.)
- **קבלה:** אין שדה `nc_roas_proxy` בתשובת ה-API; ה-UI לא מציג NC-ROAS פר קונספט.

### 1.3 hit_rate מוטה ב-Creative P&L
- **קובץ:** `gani_agent/dashboard/services/creative_pnl.py:52-57`
- **בעיה:** `hit_rate = winners / with_purchases` — מחלק רק במודעות שרכשו; צריך לחלק בכל מה שנבדק. וגם `cost_to_first_winner = total_spend / max(len(winners),1)` מציג את כל ההוצאה כ"עלות ווינר" כשאין ווינרים.
- **תיקון:** `hit_rate = len(winners) / count * 100` (count = כל המודעות בקונספט שהוציאו כסף). `cost_to_first_winner`: אם `len(winners) == 0` להחזיר `None` ולהציג ב-UI "אין ווינר עדיין (הושקעו X₪)".
- **קבלה:** קונספט עם 10 מודעות, 1 ווינר → hit_rate 10% (לא 100%); קונספט בלי ווינרים לא מציג עלות-לווינר.

### 1.4 lift בלי רצפת מדגם בגנום
- **קובץ:** `gani_agent/dashboard/services/genome_attribution.py:96-120`
- **בעיה:** lift מחושב גם לקבוצות של מודעה אחת — רעש מוצג כתובנה, ומזין את ניבויי ה-Pre-Launch.
- **תיקון:** קבוע `_MIN_GROUP = 3` (מודעות) ו-`_MIN_GROUP_SPEND = 300` (₪); קבוצה מתחת לרף מקבלת `"low_sample": true` ולא נכללת בחישובי ניבוי; ה-UI (`tabs/GenomeTab.jsx`, `tabs/PreLaunchTab.jsx`) מציג אותה באפור עם "מדגם קטן".
- **קבלה:** אלמנט עם 2 מודעות לא מופיע בטבלת ה-lift הפעילה ולא משפיע על ציון Pre-Launch.

### 1.5 מובהקות מנופחת בטסטים A/B
- **קובץ:** `gani_agent/dashboard/frontend/src/tabs/TestingTab.jsx:364-367`
- **בעיה:** מבחן ה-Z מקבל CVR לפי impressions — חשיפות אינן ניסויים בלתי-תלויים; N ענק → "מובהק" על רעש.
- **תיקון:** לשנות את שדות הקלט ל-**clicks** (או unique link clicks) בתור N, ולתייג בטופס "קליקים" במקום "חשיפות". להוסיף הערת מינימום: לא להכריז ווינר מתחת ל-100 המרות מצטברות או 7 ימים.
- **קבלה:** אותם נתונים עם N=קליקים נותנים מובהקות נמוכה משמעותית; הטופס דורש קליקים.

### 1.6 ספי ווינר לא אחידים
- **קבצים:** `frontend/src/tabs/AnglesTab.jsx:167, 829-859` (קשיח `roas >= 9`), `gani_agent/config.py:111` (`WINNERS_ROAS_THRESHOLD=8.0`)
- **תיקון:** להוסיף את הסף לתשובת `/api/color_tiers` (ראוטר `routers/metrics.py` או איפה שהוא ממומש) כ-`winner_roas_threshold`, ולצרוך אותו ב-AnglesTab במקום ה-9 הקשיח (4 מופעים).
- **קבלה:** שינוי ה-env `WINNERS_ROAS_THRESHOLD` משנה את הסף בזוויות בלי דיפלוי פרונט.

### 1.7 אזורי זמן ו-dedup בהתראות
- **קבצים:** `gani_agent/dashboard/alerts.py:94-145` (`check_fatigue`), `gani_agent/dashboard/routers/creatives.py:231`
- **בעיות:** (א) `check_fatigue` שולח התראה על אותו קריאייטיב כל יום מחדש — אין dedup; (ב) שני המקומות משתמשים ב-`date.today()` (שעון שרת) במקום שעון ישראל.
- **תיקון:** (א) KV חדש `seen_fatigue` ב-`cache.py` (מפתח `ad_id`, תפוגה 7 ימים) — לחקות את דפוס `seen_winners` הקיים באותו קובץ (שורות 30-52); (ב) להחליף ל-`_israel_today()` (קיים ב-`fb_creatives.py:30-37`).
- **קבלה:** הרצת `check_fatigue` פעמיים ברצף שולחת התראה אחת בלבד; אין `date.today()` בשני הקבצים.

### 1.8 ממוצע ATC לא משוקלל
- **קובץ:** `frontend/src/App.tsx:526-532` (`avgAtc`)
- **תיקון:** לשקלל לפי הוצאה כמו `avgRoas` שמעליו (שורות 512-525).
- **קבלה:** מודעה עם 5₪ הוצאה לא מזיזה את הממוצע כמו מודעה עם 5,000₪.

---

## שלב 2 — צמצום 17 → 7 טאבים (הסרת הסרבול)

**עיקרון: קודם מסתירים מאחורי דגל, מודדים חודש, ואז מוחקים קוד.** ככה אין חרטות.

### 2.1 מדידת שימוש בטאבים (לפני הכל)
- **קבצים:** `frontend/src/App.tsx` (ה-`useEffect` של סנכרון ה-URL, שורות 135-145), ראוטר חדש קטן או הרחבת `routers/analytics.py`
- **תיקון:** בכל החלפת `activeTab` לשלוח `POST /api/tab_usage {tab}` (fire-and-forget); לשמור מונה יומי ב-KV (`cache.py`). endpoint קריאה `GET /api/tab_usage` מחזיר מונים.
- **קבלה:** אחרי גלישה בין 3 טאבים המונים עולים; אין האטה בניווט.

### 2.2 דגל הסתרה לטאבים
- **קבצים:** `frontend/src/components/Sidebar.jsx`, `frontend/src/App.tsx` (`_validTabs`)
- **תיקון:** קבוע `HIDDEN_TABS` (נשלט מ-`/api/settings` או קובץ config פרונט) שמסתיר מה-Sidebar ומ-`_validTabs` את: **גנום, ניבוי לפני השקה, מסלול המראה, טסטים A/B, Margin Lab, Creative P&L, Intelligence (כל הסקשן)**. ניווט ישיר ב-URL לטאב מוסתר נופל ל"בסיכון".
- **קבלה:** ה-Sidebar מציג: יומי/סקירה, בסיכון, קריאייטיבים, קונספטים (ר' 2.3), טבלה, חוקים, הגדרות (+מחשבון ככלי קטן).

### 2.3 טאב "קונספטים" מאוחד
- **קבצים:** `tabs/AnglesTab.jsx` (בסיס), לקלוט מ-`tabs/CreativePnlTab.jsx` ומ-`tabs/MarginLabTab.jsx`
- **תיקון:** להוסיף ל-AnglesTab שתי עמודות/כרטיסים פר קונספט: `hit_rate` המתוקן (1.3) ו-`gross_margin_contribution` (מ-`services/margin_elasticity.py` — החישוב הזה תקין). את שני הטאבים הישנים להסתיר (2.2).
- **קבלה:** טאב אחד עונה על "איזה קונספט מרוויח, מה שיעור הפגיעה, וכמה מרווח הוא תרם".

### 2.4 מיזוג גארדריילים + תובנות
- **תיקון:** את כרטיס ה-EMQ/CAPI מ-`tabs/GuardrailsTab.jsx` להעביר ל-`tabs/SettingsTab.jsx` (סקשן "איכות סיגנל"); מה-`tabs/InsightsTab.jsx` להעביר לסקירה רק כרטיסים שלא קיימים בה כבר; להסתיר את שני הטאבים.
- **קבלה:** אין אובדן מידע — כל כרטיס שהיה בשימוש קיים במקום החדש.

### 2.5 מחיקה סופית (אחרי ~30 יום)
- אם `GET /api/tab_usage` מראה אפס שימוש: למחוק את קבצי הטאבים המוסתרים, את `services/persona_panel.py`, `services/lp_coherence.py`, `services/prelaunch_scorer.py`, `services/fatigue_forecast.py`, ואת הראוטרים שלהם (`routers/intelligence.py`, חלקים מ-`routers/genome.py`, `routers/scoring.py`), כולל הרישום ב-`server.py`.
- **קבלה:** `npm run build` + `pytest` ירוקים; `scripts/check_router_imports.py` עובר; אין ייבוא יתום.

---

## שלב 3 — NC-ROAS אמיתי פר קריאייטיב (פיצ'ר הכסף של 2026)

**הרעיון:** לחבר הזמנות Shopify למודעה שהביאה אותן, ולקבל רווח-שולי ולקוחות-חדשים **פר קריאייטיב** — המדד שכל שיטת BellaPOINT בנויה עליו. כל התשתית קיימת:
- סניפט התמה כבר אוסף `fbp/fbc`: `shopify_fbp_fbc_theme_snippet.js` (שורש הריפו)
- הזמנות נשמרות בטבלת `shopify_orders` ב-`dashboard/cache.py` (כולל payload מלא + דגל לקוח-חדש נגזר מ-`orders_count==1`)
- Webhooks: `dashboard/shopify_webhooks.py` + `routers/webhooks.py` (HMAC מאומת)

### 3.1 שכבת ייחוס הזמנה→מודעה
- **קובץ חדש:** `gani_agent/dashboard/services/order_attribution.py`
- **תיקון:** לכל הזמנה ב-`shopify_orders`: לחלץ `fbc` מה-payload (note_attributes/attributes לפי הסניפט) → לפרסר `fbclid` → לשמור בטבלה חדשה `order_attribution (order_id, ad_id, fbclid, is_new_customer, revenue, created_at)`. מיפוי `fbclid→ad_id` דרך UTM: לוודא שתבנית ה-UTM במודעות כוללת `utm_content={{ad.id}}` ולחלץ מ-`landing_site` של ההזמנה כ-fallback.
- **קבלה:** על 30 הימים האחרונים ≥50% מההזמנות מיוחסות למודעה (השאר "לא מיוחס" — לא להמציא).

### 3.2 חשיפה ב-API וב-UI
- **קבצים:** `routers/metrics.py` או ראוטר חדש; `tabs/TableTab.jsx`, `components/DetailPanel.jsx`
- **תיקון:** `GET /api/creative_nc?days=&account=` מחזיר פר `ad_id`: `orders, new_customer_orders, nc_revenue, nc_roas = nc_revenue/spend, real_margin = Σ(line_items margin) - spend`. עמודות חדשות בטבלת הרווח + בפאנל הפירוט, עם תווית "מבוסס X% ייחוס".
- **קבלה:** לכל מודעה עם רכישות מוצג NC-ROAS אמיתי; סכימת השורות מתכנסת (±20%) ל-NC-ROAS החשבוני הקיים ב-`/api/daily_metrics`.

### 3.3 חיווט להחלטות
- **קובץ:** `gani_agent/dashboard/fb_creatives.py:687-688` (באדג' `scale_signal`)
- **תיקון:** באדג' הסקייל יסומן מלא רק אם גם `nc_roas` פר-קריאייטיב ≥ סף (ברירת מחדל 2.0, מ-config); אחרת באדג' "סקייל?" עם טולטיפ "ROAS גבוה אבל NC-ROAS נמוך — לקוחות חוזרים".
- **קבלה:** מודעה עם ROAS 4 אבל NC-ROAS 0.8 לא מקבלת המלצת סקייל מלאה.

---

## שלב 4 — מקור אמת אחד לספים + הקשחת Guardian

### 4.1 ספים נגזרים ממרווח
- **קובץ:** `gani_agent/config.py:78-115`
- **בעיה:** המחשבון מלמד CPA-מקס = מרווח×0.5 (95₪), האכיפה בפועל 175₪ קשיח. **דורש החלטת בעלים לפני מימוש** — לשאול את המשתמש: לגזור, או לתעד את 175 כ"רצח מכוון".
- **תיקון (אם הוחלט לגזור):** `CPA_MAX = GROSS_MARGIN * 0.5`, `CPA_KILL = GROSS_MARGIN * 0.9` (ברירות מחדל הניתנות לדריסה ב-env); `rules_engine.py` ו-`fb_creatives.py` צורכים אותם במקום קבועים; פר-מותג כשLANO יחזור.
- **קבלה:** שינוי `GROSS_MARGIN` משנה את כל הספים בכל המערכת בעקביות.

### 4.2 בריחת rate-limit של ה-Guardian (N+1)
- **קובץ:** `gani_agent/rules_engine.py` (לולאות פר-ישות עם קריאת insights לכל אחת)
- **תיקון:** קריאה אחת ברמת חשבון: `GET act_X/insights?level=adset&time_increment=1&time_range=...` (עמוד-שניים) → מילון `adset_id → rows`; החוקים קוראים מהמילון. אותו דבר ל-level=ad.
- **קבלה:** ריצת stop-loss מלאה ≤ 6 קריאות Graph (היום: מאות).

### 4.3 מפתח API מחוץ ל-localStorage
- **קבצים:** `frontend/src/api.js:3-31`, `dashboard/auth.py`
- **תיקון:** אחרי `/api/login` מוצלח — לא לשמור את המפתח; להסתמך על עוגיית `gani_session` (httpOnly, קיימת). `X-API-Key` נשאר רק לקריאות שרת-לשרת. לוודא שה-cookie נקבע עם `secure=True, max_age=86400`.
- **קבלה:** אחרי לוגין אין `gani_api_key` ב-localStorage וכל הקריאות עוברות עם העוגייה בלבד.

### 4.4 `/api/health` ציבורי
- **קובץ:** `dashboard/routers/observability.py` + החרגה ב-`server.py`
- **תיקון:** לפצל: `/api/ping` ציבורי שמחזיר `{"ok": true}` בלבד (ל-Railway healthcheck); `/api/health` המלא מאחורי אימות ובלי קריאת Graph חיה בכל פגיעה (cache של 5 דקות).
- **קבלה:** גישה אנונימית ל-`/api/health` מחזירה 401; ל-`/api/ping` — 200.

---

## שלב 5 — פיצ'רים 2026 (אחרי שהבסיס נקי)

לפי סדר עדיפות; כל אחד עומד בפני עצמו:

1. **מציע הקצאת תקציב שבועי** — ג'וב שבועי שמחשב: 3 המודעות עם המרווח השלילי הגדול ביותר ו-2 עם החיובי הגדול ביותר (מנתוני שלב 3), מציע "העבר X₪" בטלגרם ובדשבורד, ומבצע דרך `routers/mutations.py` הקיים אחרי אישור.
2. **מרכז התראות בדשבורד** — טבלת `alerts_log` ב-SQLite (cache.py), כל התראת טלגרם נרשמת גם בה; טאב/פאנל "התראות" עם סטטוס טופל/לא; digest יומי אחד בטלגרם במקום זרם.
3. **מרווח פר SKU** — מ-line_items של הזמנות Shopify (כבר בטבלה): טבלת `sku_margins` שניתנת לעריכה ב-Settings; `real_margin` בשלב 3.2 עובר להשתמש בה במקום קבוע יחיד.
4. **דוח P&L שבועי אוטומטי** — הרחבת `daily_summary` הקיים ב-`alerts.py`: פעם בשבוע סיכום ווינרים/מפסידים/NC%/MER + המלצות ההקצאה, בפורמט קבוע.

---

## סדר עבודה מומלץ למודל המבצע

```
שלב 1 (יום-יומיים): 1.1 → 1.3 → 1.7 → 1.2 → 1.4 → 1.6 → 1.5 → 1.8   [כל אחד = קומיט]
שלב 2 (יום):        2.1 → 2.2 → 2.3 → 2.4   (2.5 רק אחרי 30 יום)
שלב 3 (2-4 ימים):   3.1 → 3.2 → 3.3
שלב 4 (יום-יומיים): 4.2 → 4.3 → 4.4   (4.1 רק אחרי החלטת בעלים)
שלב 5:              לפי הצורך
```

**כללי ברזל:** קומיט קטן אחרי כל משימה עם מספרה (`fix(1.3): hit_rate over tested creatives`); להריץ `pytest` ו-`npm run lint` לפני כל push; לא לגעת ב-`hookScore/clickScore/convertScore`; כל טקסט UI חדש בעברית; לא למחוק קוד בשלב 2 — רק להסתיר.
