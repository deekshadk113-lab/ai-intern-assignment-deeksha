# AI Intern Assignment — HSA Team

**Name:** Deeksha
**Email:** deekshadk113@gmail.com
**Date of submission:** 12-09-2026
**USN:** 1DT23CS050

---

## Overview

This repo implements all three parts of the assignment: a vanilla HTML/CSS/JS lead
capture form (Part A), two n8n automation workflows (Part B), and an end-to-end
integration wiring the form to a live n8n webhook (Part C, bonus).

The scenario I built around is a **study-abroad admissions form** — a student
picks a course level and preferred university, and the backend automation
routes postgraduate/PhD leads to the counselling team while everything else
gets logged for follow-up. It gave Part A real fields to validate and Part B a
believable branching rule.

---

## Part A — Student Lead Capture Form

**Location:** `part-a/index.html`

Pure HTML + CSS + vanilla JavaScript — no frameworks or build step.

**Fields:** full name, email, country (dropdown), course level (UG/PG/PhD
radio), preferred university, and a message textarea with a live 300-character
countdown.

**Behaviour:**
- Every field is validated on submit (email via regex, all others for
  presence/length); invalid fields get a red outline and an inline error
  message under the field, and focus jumps to the first invalid field.
- On successful submit, the form is replaced with a thank-you panel with no
  page reload, and the submitted data is logged to the browser console as
  JSON.
- Fully responsive: single-column layout, fluid padding, and a stacked submit
  button below ~480px.

**How to run/test:**
1. Open `part-a/index.html` directly in any browser (no server needed).
2. Try submitting empty/invalid values first to see the inline errors, then
   fill in valid values and submit.
3. Open the browser console (F12) before submitting to see the logged JSON
   payload.
4. Resize the window or use device-toolbar mode to check the mobile layout.

---

## Part B — N8N Automation Workflows

**Location:** `part-b/`

### B1 — Lead Notification Workflow (`workflow-b1-lead-notification.json`)

`Webhook → Set → IF → (Email notification | Google Sheets log)`

- **Webhook** node listens for a `POST` at `/webhook/lead-notification`,
  simulating the form submitting its payload.
- **Set** node pulls out and renames the fields we care about: `name`,
  `email`, `courseLevel`, `preferredUniversity`, `message`.
- **IF** node checks `courseLevel === "PG" OR courseLevel === "PhD"`.
  - **True →** Email Send node notifies the counselling team (swap for a
    Slack node if you'd rather notify over Slack — same inputs).
  - **False →** Google Sheets node appends the lead to a "Leads" sheet for
    manual follow-up later.

**How to run/test:**
1. In n8n, `Import from File` → select `workflow-b1-lead-notification.json`.
2. Add SMTP (or Slack) and Google Sheets credentials, replacing the
   `PLACEHOLDER_*` values in the Email Send and Google Sheets nodes — or swap
   the Google Sheets node for a **No-Op** node if you don't have Sheets
   access, to mock the "else" branch.
3. Activate the workflow and copy its **Test URL** (or Production URL) from
   the Webhook node.
4. Send a test `POST` with a JSON body like:
   ```json
   { "fullName": "Asha Rao", "email": "asha@example.com", "courseLevel": "PG", "preferredUniversity": "University of Melbourne", "message": "Interested in the Jan intake." }
   ```
   via curl, Postman, or the Part C form (see below) and confirm it takes the
   correct branch.

### B2 — Scheduled Data Fetch Workflow (`workflow-b2-scheduled-fetch.json`)

`Cron (daily, 9am) → HTTP Request → Code (JS transform) → Set`

- **Cron/Schedule Trigger** fires once a day at 9am.
- **HTTP Request** calls the [Hipolabs Universities API](http://universities.hipolabs.com/search?country=India),
  a free, no-auth public API that returns university name, country, state,
  domains, and web pages for a given country.
- **Code** node (JavaScript) filters out entries missing a web page or
  state/province, then maps the rest down to a small, clean shape and caps
  the list at 25 results.
- **Set** node labels the final output fields (`universityName`, `region`,
  `domain`, `website`, `fetchedAt`) ready for downstream use (e.g. feeding
  the "preferred university" dropdown in Part A, or a digest email).

**Which API and why:** I used the Hipolabs Universities API because it needs
no API key, returns clean structured JSON, and fits the admissions theme of
this assignment directly — the same "universities" concept that shows up in
Part A's form. That made B2 feel like a real extension of the same product
rather than an unrelated demo call.

**How to run/test:**
1. In n8n, `Import from File` → select `workflow-b2-scheduled-fetch.json`.
2. Click the HTTP Request node → **Execute Node** to fetch live data without
   waiting for the schedule.
3. Step through Code → Set with "Execute Node" on each to confirm the shape
   of the output at every stage.
4. Change the `country` query parameter on the HTTP Request node to test
   other countries.

### Screenshots

`screenshot-b1.png` and `screenshot-b2.png` should show each workflow's
canvas after import, with all nodes connected as described above. **Add these
after importing the JSON into your own n8n instance** — see
`part-b/HOW-TO-CAPTURE-SCREENSHOTS.md` for the exact steps.

---

## Part C — Integration Challenge (Bonus)

**Location:** `part-c/index.html`

Same form as Part A, with one addition: on submit, validated data is `POST`ed
via `fetch()` to the B1 webhook URL, with `Content-Type: application/json`.

**Behaviour:**
- Submit button shows a spinner and switches to "Sending..." while the
  request is in flight, and is disabled to prevent double submits.
- On a successful response, the form is replaced with the thank-you panel
  (same as Part A).
- On a network error or non-2xx response, an inline error banner appears
  above the form ("We couldn't reach the server...") and the button resets
  so the user can retry — nothing is lost.

**How to run/test:**
1. Get your webhook URL from the imported B1 workflow (Test or Production).
2. Open `part-c/index.html` in a text editor and replace `WEBHOOK_URL` near
   the top of the `<script>` block with that URL.
3. Open the file in a browser and submit the form with the workflow active in
   n8n — you should see the execution appear in n8n's execution log, and the
   thank-you panel appear in the browser.
4. To see the error state, point `WEBHOOK_URL` at an inactive/wrong URL and
   submit again.

**Demo recording:** [LOOM_LINK_OR_GIF_PATH_HERE]

---

## Environment Variables / Credentials (placeholders)

None of these are committed — replace them locally before testing:

| Where | Placeholder | What it should be |
|---|---|---|
| `part-b/workflow-b1-lead-notification.json` | `PLACEHOLDER_SMTP_CREDENTIAL_ID` / `PLACEHOLDER_SMTP_ACCOUNT` | Your n8n SMTP (or Slack) credential |
| `part-b/workflow-b1-lead-notification.json` | `PLACEHOLDER_GOOGLE_SHEET_ID` / `PLACEHOLDER_GOOGLE_CREDENTIAL_ID` / `PLACEHOLDER_GOOGLE_ACCOUNT` | Your Google Sheets ID + OAuth credential |
| `part-c/index.html` | `WEBHOOK_URL` (`https://YOUR-N8N-INSTANCE/webhook/lead-notification`) | Your B1 workflow's webhook URL |

---

## Challenges Faced

Getting the IF node's branching logic right for "PG or PhD" took a couple of
tries — my first pass used two chained IF nodes, which worked but was harder
to read than it needed to be, so I collapsed it into a single IF node with an
OR-combined condition instead. The other challenge was keeping Part A and
Part C's validation logic in sync without a shared JS file (per the "no
frameworks/simple structure" spirit of the brief), which I resolved by
keeping both files self-contained but structurally identical, so a reviewer
can diff them directly to see exactly what Part C adds.

---


