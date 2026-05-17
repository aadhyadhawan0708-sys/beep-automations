# Daily Beep Report — Founder's Office Teams Chat

Pull data from Raj Chauhan's Outlook email, Mixpanel, and Madgicx, then post a
formatted report with AI insights to the Founder's Office Teams chat every morning.

Founder's Office chat ID: 19:3f7d59cc89644e78afb85e4f671ec00d@thread.v2
Composio account: microsoft_teams_helot-laigh
Mixpanel project ID: 3512236

---

## STEP 1 — Fetch Raj Chauhan's email

Use `outlook_email_search` with sender "raj chauhan", limit 5. Pick the most
recent email whose subject includes words like "report", "daily", "update", or
"EOD". Prefer yesterday's email.

Then call `read_resource` with URI `mail:///messages/{messageId}` to fetch the
full body. The email will be too large to read inline — it will be saved to disk
automatically. Use bash + python3 to parse the file at the path shown in the
error message. Strip all HTML tags by replacing them with a pipe `|`, replace
`&nbsp;` with space, split by `|`, strip whitespace, and remove empty tokens.
This gives you a clean list of tokens to scan.

Extract the following fields from yesterday's date row (format: DD-Mon-YY,
e.g. 16-May-26):

**From "May'26 Month High Ticket Course Achievement Summary" table:**
- `adm_count` → column "ADM Count", yesterday's row
- `vc_done` → column "VC Done", yesterday's row
- `cib_mtd` → column "Rev Collected" from the **Total row** (MTD cumulative).
  CIB target is ₹40,00,000 — calculate % achieved as
  `(cib_mtd / 4000000) * 100`.

**From "Retention Report Summary May'26" table:**
- `day1_retention` → column "Day 1", most recent row that has a value.
  Yesterday's Day 1 may not be available yet — use the day-before-yesterday's
  value in that case.

**From "Overall Report Summary May'26" table:**
- `pro_plan_revenue_mtd` → column "Total Pro Plan Rev", Total row (MTD
  cumulative).

Use "—" for any field that is missing or ambiguous.

---

## STEP 2 — Fetch Mixpanel data for yesterday

Use **absolute** date range for all queries — yesterday's date only:
`{ "type": "absolute", "from": "YYYY-MM-DD", "to": "YYYY-MM-DD" }`

**A. New Signups**
Run-Query, report_type: `insights`, event: `sign_up_success`,
measurement: `{ "type": "basic", "math": "total" }`, dateRange: absolute
(yesterday). Extract total count → `new_signups`.

**B. New Pro Plan Payments**
Run-Query, report_type: `insights`, event: `pro_autopay_success`,
measurement: `{ "type": "basic", "math": "total" }`, dateRange: absolute
(yesterday). Extract total count → `new_pro_payments`.

**C. Conversion Rate**
Run-Query, report_type: `funnels`, step 1: `sign_up_success`, step 2:
`pro_autopay_success`, dateRange: **last 7 days (relative)**. Extract
`overall_conv_ratio` from step 2 and convert to % → `conversion_rate`.
If `overall_conv_ratio` is 0 but step counts are available, calculate manually:
`(step2_count / step1_count) * 100`.

---

## STEP 3 — Fetch Madgicx data for yesterday (Ads Manager 2.0)

This step pulls Facebook Ads performance data scoped to the **Certifications
campaign** for yesterday. Run these sub-steps sequentially:

**3a. Verify auth**
Call `fb_whoiam` (tool slug: `MADGICX_WHOIAM`) to confirm authentication and
get context.

**3b. Discover ad account**
Call `list_ad_accounts` (tool slug: `MADGICX_LIST_AD_ACCOUNTS`) to get the
available Facebook Ad Account ID(s). Use the first active account ID.
Store it as `{account_id}` (format: `act_XXXXXXXXX`).

**3c. Find the Certifications campaign**
Call `get_ad_account_insights` (tool slug: `MADGICX_GET_AD_ACCOUNT_INSIGHTS`)
with:
- `account_id`: `{account_id}`
- `level`: `"campaign"`
- `date_preset`: `"yesterday"`
- `fields`: `["campaign_name", "spend", "purchase_roas", "actions",
  "action_values"]`
- `time_increment`: `"all_days"`
- `default_summary`: `false`

From the results, find the row(s) where `campaign_name` contains "certif"
(case-insensitive). If multiple rows match, sum their values.

Extract:
- `certs_spend` → `spend` field (sum across matching campaigns, in ₹)
- `roas` → `purchase_roas[0].value` (weighted average if multiple campaigns;
  round to 2 decimal places). If `purchase_roas` is absent, try
  `website_purchase_roas[0].value`.
- `certs_bought` → from `actions` array, find the entry where
  `action_type` is `"offsite_conversion.fb_pixel_purchase"` or
  `"purchase"` or `"omni_purchase"` — use whichever is present — and sum
  the `value` field across matching campaigns (round to nearest integer).

If no campaign matching "certif" is found, try broader name patterns:
"certificate", "cert", "learn". If still not found, use "—" for all three
fields and note: ⚠️ Certifications campaign not found in Madgicx.

---

## STEP 4 — Generate AI Insights

Write **3 concise bullet points** based on all data collected. Key benchmarks
and context to apply:

- Beep Learn CIB MTD target is ₹40,00,000 — always state % achieved and
  whether pace is on track (pace = % of month elapsed vs % of target achieved).
- Day 1 retention May benchmark is ~5.45% — flag clearly if below this.
- Flag conversion rate if signup-to-pro is under 1% over the last 7 days.
- For Madgicx: flag if ROAS < 1.5x (unprofitable territory); highlight if
  ROAS > 3x (strong). Comment on whether certifications spend is driving
  meaningful purchase volume.
- Highlight anything notably strong or weak (e.g. high admissions day, spike
  in pro payments, retention improvement, certs spike).

Each insight should be 1–2 sentences, specific, and actionable.

---

## STEP 5 — Post to Teams

Use `COMPOSIO_SEARCH_TOOLS` to find the Teams message tool if needed, then
`COMPOSIO_MULTI_EXECUTE_TOOL` with tool slug
`MICROSOFT_TEAMS_TEAMS_POST_CHAT_MESSAGE`, account
`microsoft_teams_helot-laigh`,
chatId `19:3f7d59cc89644e78afb85e4f671ec00d@thread.v2`.

**IMPORTANT — HTML formatting required.** Pass two fields: `content` (the HTML
body below) and `contentType` with value `html`. This ensures Teams renders the
message with proper line breaks and bold headings instead of a wall of text.

Build the `content` string as HTML, substituting all values. Use this exact
structure:

```html
<b>📊 Report Analysis — {DD/MM/YYYY of yesterday}</b><br>
━━━━━━━━━━━━━━━━━━━━━━━━<br>
<br>
<b>1. Beep Learn</b><br>
&nbsp;&nbsp;&nbsp;• Admissions: {adm_count}<br>
&nbsp;&nbsp;&nbsp;• VC Done: {vc_done}<br>
&nbsp;&nbsp;&nbsp;• CIB MTD: ₹{cib_mtd} / ₹40,00,000 target ({%_achieved}%)<br>
<br>
<b>2. Beep App</b><br>
&nbsp;&nbsp;&nbsp;• New Signups: {new_signups}<br>
&nbsp;&nbsp;&nbsp;• New Pro Plan Payments: {new_pro_payments}<br>
&nbsp;&nbsp;&nbsp;• Day 1 Retention: {day1_retention}<br>
&nbsp;&nbsp;&nbsp;• Conversion Rate: {conversion_rate}<br>
&nbsp;&nbsp;&nbsp;• Pro Plan Revenue MTD: ₹{pro_plan_revenue_mtd}<br>
<br>
<b>3. Certifications (Ads Manager)</b><br>
&nbsp;&nbsp;&nbsp;• Certificates Bought: {certs_bought}<br>
&nbsp;&nbsp;&nbsp;• ROAS: {roas}x<br>
&nbsp;&nbsp;&nbsp;• Marketing Spend on Certs: ₹{certs_spend}<br>
<br>
━━━━━━━━━━━━━━━━━━━━━━━━<br>
<b>AI Insights —</b><br>
<br>
• {insight_1}<br>
<br>
• {insight_2}<br>
<br>
• {insight_3}
```

---

## Error Handling

- If Raj Chauhan's email is not found, use "—" for all email fields and add:
  ⚠️ Raj Chauhan's daily report email not yet received.
- If Mixpanel queries fail, use "—" for those fields and add:
  ⚠️ Mixpanel data unavailable.
- If Madgicx auth fails or no data is returned, use "—" for all Madgicx fields
  and add: ⚠️ Madgicx data unavailable.
- **Always post to Teams even with partial data** — partial data is better than
  silence.
