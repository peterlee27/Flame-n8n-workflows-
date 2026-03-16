# Flame Centre — n8n Workflow 1: Participant Data Ingestion
# Claude Code Context Brief

---

## Organisation Background
- **Company:** Flame Centre, Singapore (human skills institute)
- **MD:** Wendy Tan
- **Key staff:** Admin person (ops), Client Partnership Person (BD + delivery), Jael (trainer)
- **Trainer pool:** 9 external trainers + Wendy + Jael + Client Partnership Person
- **Survey tool:** Sogolytics
- **Compliance platforms:** Skilleto (SSG/IBF registration), TPG (IBF only)
- **Comms:** Outlook email (confirm before building — may be Gmail)
- **Files:** Google Drive folders per workshop (links stored in dashboard)

---

## Problem This Workflow Solves

Currently:
- Clients send participant lists late via email (Excel/CSV/sometimes in email body)
- Admin manually reads the email, manually types each participant into the workshop sheet
- For SSG/IBF funded workshops, admin then manually inputs the SAME data again into Skilleto AND TPG separately
- This is slow, error-prone, and chaotic when it arrives last minute before a workshop

After this workflow:
- The moment a participant list email arrives, everything processes automatically
- No manual data entry
- No duplicate Skilleto/TPG input
- Admin is notified, client is acknowledged, dashboard is updated

---

## Workflow 1 — Full Step-by-Step Logic

### STEP 1: Trigger — Email Arrives
- n8n monitors Flame Centre's Outlook inbox continuously (or Gmail — confirm)
- Trigger fires when email arrives with:
  - Attachment (Excel .xlsx, .csv) OR
  - Table/list in email body
- Detection heuristic: subject line OR sender domain matches known client
- Keywords to watch: "participant list", "participants", "attendees", "pax list", "name list"
- **If none of these match:** email is ignored — workflow does NOT trigger

### STEP 2: Extract the Data (Claude AI Node)
- n8n sends the email body + attachment to Claude
- Claude's job: extract a clean, structured participant table regardless of input format
- Claude outputs clean JSON array:
```json
[
  {
    "name": "John Tan",
    "email": "john.tan@company.com",
    "organisation": "DBS Bank",
    "designation": "Senior Manager",
    "nric_last4": "567A",
    "citizenship": "Singaporean"
  }
]
```
- Claude also extracts from email body: client name, workshop name hint, any dates mentioned
- NRIC and citizenship fields only populated if present (required for SSG/IBF only)
- If extraction fails or data is too messy: Claude flags "EXTRACTION_FAILED" and workflow routes to manual review

### STEP 3: Match to Workshop (Switch/IF Node)
- n8n reads the MASTER DASHBOARD sheet
- Attempts to match the email to a workshop row using:
  1. Sender email domain → match against Client Contact_Email column
  2. Account name extracted by Claude → fuzzy match against Account Name column (Column B)
  3. Workshop date mentioned in email → match against Start Date column (Column D)
  4. Workshop title keywords → match against Workshop Title column (Column C)
- Match confidence scoring:
  - 3–4 criteria match = HIGH CONFIDENCE → proceed automatically
  - 1–2 criteria match = LOW CONFIDENCE → flag for manual confirmation (see Issue #1)
  - 0 criteria match = NO MATCH → full flag to admin (see Issue #1)

### STEP 4: Workshop File — Where to Write (see Issue #2 and Issue #3)
- Architecture: ONE master dashboard (lean, 20 columns) + SEPARATE files per workshop
- Each confirmed workshop has its own file stored in Google Drive
- The Master Dashboard (Column S) contains the link to each workshop's participant sheet
- n8n reads Column S to get the specific file URL for the matched workshop
- n8n writes participant data into THAT file (not a new file, not a new tab in master)
- Column headers in each workshop file vary by workshop type (see Issue #3):

**Standard (Non-Funded) headers:**
Name | Email | Organisation | Designation | Attendance | Notes

**SSG-funded headers:**
Name | Email | Organisation | Designation | NRIC (last 4) | Citizenship | Attendance | Skilleto Status | Notes

**IBF-funded headers:**
Name | Email | Organisation | Designation | NRIC (last 4) | Citizenship | Attendance | Skilleto Status | TPG Status | Notes

### STEP 5: Funding Type Check (Switch Node)
- n8n reads Column G (Funding Type) from Master Dashboard for the matched workshop
- Three routes:

**Route A — Non-Funded:**
- Write participant data to workshop file
- Skip Skilleto and TPG entirely
- Go to Step 6

**Route B — SSG Funded:**
- Write participant data to workshop file (with NRIC fields)
- Trigger Skilleto sub-workflow (see Sub-Workflow B below)
- Skip TPG
- Go to Step 6

**Route C — IBF Funded:**
- Write participant data to workshop file (with NRIC fields)
- Trigger BOTH Skilleto AND TPG sub-workflows simultaneously
- Go to Step 6

### STEP 6: Update Master Dashboard
- n8n writes back to Master Dashboard row for the matched workshop:
  - Column I (# Participants): update count
  - Column J (Participant List Received): change to "Yes"
  - Column Q (n8n Flag): clear any "🔴 Participant list missing" flag, replace with "✅ List received [timestamp]"

### STEP 7: Send Confirmations
- **To client:** "Thank you — we've received the participant list for [Workshop Name] on [Date]. [X] participants confirmed."
- **To admin:** "Participant list received for [Workshop Name] — [X] participants logged. [If SSG/IBF: Skilleto/TPG entry triggered.]"
- **To client partnership person:** brief CC or separate notification if workshop is within 7 days

---

## Sub-Workflow B — Skilleto Entry
- Skilleto does not have a public API on standard plans
- **Current workaround options:**
  1. n8n formats participant data into the exact CSV template Skilleto accepts → admin uploads manually (semi-automated)
  2. If Skilleto supports webhook/HTTP target (Enterprise plan only): n8n posts directly
  3. Claude fills a browser form via automation (complex, fragile — avoid for now)
- **Recommended for Phase 1:** n8n generates a pre-formatted Skilleto-ready CSV and emails it to admin with instructions: "Skilleto upload ready — download and upload at [link]"
- Column S (Skilleto Done) updated to "Pending Upload" until admin confirms

## Sub-Workflow C — TPG Entry
- Same challenge as Skilleto — check if TPG has API
- Phase 1 same approach: n8n generates TPG-formatted data → admin uploads
- Column N (TPG Done) updated to "Pending Upload"

---

## ISSUES AND HOW TO HANDLE THEM

### Issue #1 — No Match Found / Low Confidence Match

**Scenario:** Participant list email arrives but n8n can't confidently match it to a workshop

**How the system flags it (three simultaneous actions):**

1. **Email to admin:**
   - Subject: "⚠️ Participant list received — workshop not identified"
   - Body: "A participant list was received from [sender email] at [time]. We could not automatically match this to a workshop. Original email and attachment are attached. Please reply to this email with the Workshop ID (format: FC-2026-XXX) to continue processing."
   - Original email and attachment forwarded as part of the notification

2. **Pending row written to Master Dashboard:**
   - New row added at the top of Master Dashboard (above all confirmed rows)
   - Status: "⚠️ PENDING MATCH"
   - Account Name: best guess from Claude extraction
   - n8n Flag: "Unmatched participant list — manual review needed"
   - Row highlighted in orange
   - Admin sees this the moment she opens the dashboard

3. **Teams/WhatsApp notification (optional):**
   - If Flame Centre uses Teams: n8n posts to a designated channel
   - More immediate than email, harder to miss
   - Message: "⚠️ Unmatched participant list from [sender] — check dashboard"

**What happens after admin responds:**
- Admin replies to the notification email with the Workshop ID (e.g., "FC-2026-015")
- n8n detects the reply, reads the Workshop ID, and resumes the workflow from Step 4
- The workflow does NOT die — it pauses in a waiting state until manual confirmation
- The "PENDING MATCH" row is removed from the dashboard once resolved

**n8n implementation note:**
- Use a Webhook node or email reply trigger to listen for admin's confirmation response
- Store the pending data temporarily (n8n's built-in data store or a "Pending" tab in the dashboard)
- Set a timeout: if no reply within 24 hours → escalate to client partnership person

---

### Issue #2 — New File vs New Tab Question

**Decision: No new spreadsheet created at participant list arrival.**

Here is the file architecture:

```
Google Drive/
  └── Flame Centre Operations/
        ├── 📊 Master Dashboard (single file — n8n reads this daily)
        ├── 📁 Active Workshops/
        │     ├── FC-2026-001_TAI_DBS_20Mar/
        │     │     ├── Participant Sheet.xlsx   ← n8n writes HERE
        │     │     ├── Pre-Workshop Survey.link
        │     │     └── Materials/
        │     ├── FC-2026-002_Leadership_GovTech_22Mar/
        │     └── ...
        ├── 📁 Completed Workshops/
        │     └── 2026/Q1/...
        └── 📁 Templates/
              ├── Template_Standard.xlsx
              ├── Template_SSG.xlsx
              └── Template_IBF.xlsx
```

**When is the participant file created?**
- NOT when the participant list arrives
- WHEN the workshop status changes to "Confirmed" in the Master Dashboard (triggers a separate setup sub-workflow)
- By the time the participant list arrives, the file ALREADY EXISTS and is linked in Column S
- n8n simply writes into that pre-existing file

**Why this matters for n8n:**
- n8n reads Column S → gets the exact file URL → writes directly
- No need to search for or create files at ingestion time
- Keeps Workflow 1 simple and fast

---

### Issue #3 — Multiple Sheets, Different Structures Per Workshop Type

**Problem:** Different workshop types need different participant data fields (Standard vs SSG vs IBF)

**Solution: Template-based file creation (handled in setup sub-workflow, before WF1 runs)**

When a workshop is confirmed:
- n8n reads Funding Type (Column G)
- Copies the matching template from Templates folder
- Renames it: FC-2026-XXX_[Client]_[Date]_Participants.xlsx
- Places it in the correct Active Workshop folder
- Pastes the file link into Master Dashboard Column S

By the time Workflow 1 runs, the file already has the correct headers for its funding type. Claude knows which fields to extract based on the template structure it detects when it reads the file.

**n8n Switch Node logic:**
```
Read Column G (Funding Type)
  ↓
IBF   → copy Template_IBF.xlsx   (includes NRIC, Citizenship, Skilleto Status, TPG Status)
SSG   → copy Template_SSG.xlsx   (includes NRIC, Citizenship, Skilleto Status)
Non-Funded → copy Template_Standard.xlsx (Name, Email, Org, Designation only)
```

---

### Issue #4 — Large Dashboard Scale

**Problem:** As Flame Centre grows, one giant workbook with many tabs becomes slow and fragile

**Solution already implemented in the n8n-ready file:**
- Master Dashboard = ONE lean sheet (20 columns, one row per workshop)
- n8n ONLY reads the Master Dashboard daily (fast, always clean)
- Individual workshop files stored separately in Drive (n8n only opens them when acting)
- Completed workshops: status changed to "Archived" → n8n moves file to Completed folder → row hidden in master dashboard
- Master dashboard never grows beyond active + upcoming workshops

**n8n implementation note:**
- Never loop through all tabs looking for data
- Always use Workshop ID as the lookup key
- Always read Master Dashboard first → get file link → open specific file

---

## Dashboard Column Reference (Master Dashboard)

| Column | Letter | n8n Field Name | Type | Notes |
|--------|--------|----------------|------|-------|
| Workshop ID | A | workshop_id | Text | Primary key. Format: FC-2026-XXX |
| Account Name | B | account_name | Text | Used for email sender matching |
| Workshop Title | C | workshop_title | Text | For email personalisation |
| Start Date | D | start_date | Date | Calculate days_to_workshop |
| End Date | E | end_date | Date | |
| Trainer | F | trainer | Text | For briefing routing |
| Funding Type | G | funding_type | Dropdown | IBF / SSG / Non-Funded |
| Status | H | status | Dropdown | Confirmed / In Prep / Ready / Delivered / Archived |
| # Participants | I | participant_count | Number | Updated by WF1 |
| Participant List | J | participant_list_received | Yes/No | WF1 sets to Yes on completion |
| Survey Sent | K | survey_sent | Yes/No | |
| Welcome Letter | L | welcome_letter_sent | Yes/No | |
| Skilleto Done | M | skilleto_done | Yes/No/NA | WF1 sets to Pending/Yes |
| TPG Done | N | tpg_done | Yes/No/NA | WF1 sets to Pending/Yes |
| Feedback Received | O | feedback_received | Yes/No | |
| NPS Score | P | nps_score | Number | |
| n8n Flag | Q | flag | Text | Written by n8n — do not edit manually |
| Workshop Folder | R | folder_link | URL | Google Drive folder |
| Participant Sheet | S | participant_sheet_link | URL | Direct link to participant file |
| Notes | T | notes | Text | Free text — Claude reads for context |

---

## n8n Node Sequence for Workflow 1

```
[EMAIL TRIGGER] — Outlook/Gmail: watch inbox for emails with attachments
        ↓
[HTTP REQUEST / ATTACHMENT EXTRACT] — pull attachment binary or email body
        ↓
[CLAUDE AI NODE] — prompt: extract participant table as JSON array
        System prompt: "You are extracting participant data from emails and attachments. 
        Output ONLY a JSON array with fields: name, email, organisation, designation, 
        nric_last4 (if present), citizenship (if present). Also extract: sender_name, 
        sender_email, workshop_hint (any workshop name mentioned), date_hint (any date mentioned).
        If extraction fails output: {error: 'EXTRACTION_FAILED', reason: '...'}"
        ↓
[IF NODE] — check if Claude returned EXTRACTION_FAILED
        YES → [SEND EMAIL to admin: manual review needed] → STOP
        NO  → continue
        ↓
[GOOGLE SHEETS / EXCEL NODE] — read ALL rows from Master Dashboard
        ↓
[FUNCTION NODE] — match logic:
        score = 0
        if sender_email domain matches client contact email domain → score += 3
        if account_name fuzzy matches → score += 2  
        if date_hint matches start_date → score += 2
        if workshop_hint keyword found in title → score += 1
        
        if score >= 4 → HIGH_CONFIDENCE match
        if score 1-3 → LOW_CONFIDENCE match (flag + proceed with best guess)
        if score = 0 → NO_MATCH
        ↓
[SWITCH NODE] — route by match confidence
        NO_MATCH     → [Issue #1 handler]: email admin + write pending row + Teams ping
        LOW_CONFIDENCE → [flag admin] + [wait for confirmation webhook]
        HIGH_CONFIDENCE → continue
        ↓
[GOOGLE SHEETS NODE] — read matched workshop row, get Column G (funding_type) and Column S (participant_sheet_link)
        ↓
[SWITCH NODE] — route by funding type
        IBF         → [write to IBF template columns] → [trigger Skilleto sub] → [trigger TPG sub]
        SSG         → [write to SSG template columns] → [trigger Skilleto sub]
        Non-Funded  → [write to Standard template columns]
        ↓
[GOOGLE SHEETS / EXCEL NODE] — write participant rows to the workshop participant file (Column S link)
        ↓
[GOOGLE SHEETS NODE] — update Master Dashboard:
        Column I: participant count
        Column J: "Yes"
        Column Q: "✅ List received [timestamp]"
        ↓
[GMAIL / OUTLOOK NODE] — send confirmation to client
[GMAIL / OUTLOOK NODE] — send notification to admin
```

---

## Claude AI Node Prompt Templates

### Extraction Prompt (Step 2)
```
You are a data extraction assistant for Flame Centre, a training company in Singapore.

Your task: Extract participant list data from the email or attachment provided.

Output a JSON object with two keys:
1. "participants" - array of participant objects
2. "meta" - metadata extracted from the email

Participant fields (include if present, leave blank if not):
- name (full name)
- email
- organisation
- designation  
- nric_last4 (last 4 characters of NRIC only, e.g. "567A")
- citizenship (Singaporean / PR / Foreigner)

Meta fields:
- sender_name
- sender_email
- workshop_hint (any workshop or course name mentioned)
- date_hint (any date mentioned in DD Mon YYYY format)
- participant_count (total count if mentioned)

If the data is too messy or unreadable, return:
{"error": "EXTRACTION_FAILED", "reason": "brief explanation"}

Return ONLY the JSON object. No preamble, no markdown, no explanation.
```

### Match Confirmation Prompt (for low confidence)
```
You are reviewing a participant list received by Flame Centre admin.

The participant list was sent by: [sender_email]
They mentioned: [workshop_hint] and [date_hint]

Here are the active workshops in the system:
[paste top 5 closest matches from dashboard]

Which workshop does this participant list most likely belong to?
Return the Workshop ID (format: FC-2026-XXX) and your confidence (High/Medium/Low).
If none match, return "NO_MATCH".
```

---

## Things to Confirm Before Building

1. **Email platform:** Outlook or Gmail? (determines which trigger node to use)
2. **File storage:** Google Drive or OneDrive? (determines file management nodes)
3. **Skilleto plan:** Does Flame Centre have Enterprise plan with API/HTTP Targets?
4. **TPG:** Does TPG have an API or is it web-form only?
5. **Teams/WhatsApp:** Which does the team use for urgent pings?
6. **n8n setup:** Cloud (n8n.cloud) or self-hosted?
7. **Claude API key:** Already set up in n8n credentials?

---

## Test Scenarios to Validate Before Go-Live

| Test | Input | Expected Output |
|------|-------|----------------|
| Clean Excel attachment, known client | DBS email with .xlsx | Participants written to FC-2026-012 file, Column J → Yes |
| CSV attachment | GOvTech CSV | Same as above but parsed from CSV |
| Participant list in email body (no attachment) | Table pasted in email body | Claude extracts correctly |
| Unknown sender | Random email with Excel | NO_MATCH flagged, admin emailed |
| Partially matching sender | Known domain, unknown workshop | LOW_CONFIDENCE flag, admin asked to confirm |
| SSG funded workshop | MBS email | Participants written + Skilleto CSV generated |
| IBF funded workshop | DBS email | Participants written + both Skilleto and TPG CSVs generated |
| Messy/unreadable attachment | Password-protected Excel | EXTRACTION_FAILED → admin notified |
| Duplicate submission | Same client sends list twice | n8n detects count already filled, asks admin to confirm overwrite |

---

## Build Order Recommendation

1. Build the email trigger + Claude extraction node first — test it reads emails correctly
2. Add the Master Dashboard read node — test it pulls the right rows
3. Build the match logic function node — test all three confidence levels
4. Add the Issue #1 handler (no match flow) — test the admin email and pending row
5. Add the participant file write node — test it writes to the correct file
6. Add the Master Dashboard update node — test Column J and Q update correctly
7. Add confirmation emails — test client and admin receive correct content
8. Add Skilleto/TPG sub-workflows last — these depend on what API access is available

---
*Context file generated from planning session with Claude on 16 March 2026*
*Flame Centre — Internal Use Only*
