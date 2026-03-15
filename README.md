# Flamecentre n8n Workflows

Comprehensive n8n workflow suite for Flamecentre business automation.

## Workflows

| # | Workflow | Description | Trigger |
|---|---------|-------------|---------|
| 01 | Morning Briefing | Daily briefing with schedule, emails, proposals & AI headlines | Daily @ 7am |
| 02 | Post-Workshop Follow-Up | 3-email nurture sequence (Day 1, 7, 30) for workshop participants | Workshop end date |
| 03 | LinkedIn Content Engine | AI-generated LinkedIn posts with approval loop & auto-publish | Webhook |
| 04 | Lead Routing & Outreach | Sector-based outreach (public/private) with auto follow-up | Webhook |
| 05 | Book Download Nurture | 3-email sequence for Smarter book downloads, sector-tailored | Webhook |
| 06 | Research Insight Extractor | Extract insight cards & digest from interview transcripts | Manual |

## Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- Google Workspace OAuth2 credentials (Calendar, Gmail, Sheets)
- Anthropic API key

### Import Instructions

1. Open your n8n instance
2. Go to **Workflows → Import from File**
3. Select the `.json` file from the `workflows/` folder
4. Configure credentials:
   - **Google Calendar OAuth2** – connect your Google account
   - **Gmail OAuth2** – connect your Google account
   - **Google Sheets OAuth2** – connect your Google account
5. Set the `ANTHROPIC_API_KEY` environment variable in n8n Settings → Variables
6. Update the Google Sheets node with your actual Proposals spreadsheet ID and sheet name
7. Activate the workflow

### Google Sheets – Proposals Format

The proposals sheet should have these columns:

| client | proposal_name | amount | status | date |
|--------|--------------|--------|--------|------|
| Acme Corp | Website Redesign | 15000 | open | 2026-03-10 |

The `status` column must contain `open` for rows to be picked up by the workflow.

### Google Sheets – Workshop Participants Format (Workflow 02)

The participants sheet should have these columns:

| participantName | participantEmail | workshopName | workshopDate | workshopTopic | resourceLinks | status |
|----------------|-----------------|--------------|-------------|--------------|---------------|--------|
| Jane Doe | jane@example.com | Leadership Ignite | 2026-03-15 | Leadership & Team Dynamics | https://link1.com, https://link2.com | completed |

- `workshopDate` should match the date the workshop ends (format: `yyyy-MM-dd`)
- `status` must be `completed` for participants to enter the follow-up sequence
- After the full 30-day sequence, the workflow automatically updates `status` to `nurtured`

### Google Sheets – Content Calendar Format (Workflow 03)

| date | topic | angle | targetAudience | contentType | status | publishedDate |
|------|-------|-------|---------------|-------------|--------|--------------|
| 2026-03-15 10:30 | AI in Leadership | Practical tips | C-suite executives | post | draft | |

The workflow auto-logs entries on request and updates `status` to `published` with a timestamp after posting.

### Webhook – LinkedIn Content Request (Workflow 03)

Send a POST request to trigger content generation:

```json
{
  "topic": "AI in Leadership",
  "angle": "3 practical ways leaders can use AI today",
  "targetAudience": "C-suite executives and founders",
  "contentType": "post"
}
```

### Webhook – New Lead (Workflow 04)

```json
{
  "leadName": "John Smith",
  "organisation": "Ministry of Education",
  "sector": "public",
  "leadEmail": "john@moe.gov.sg"
}
```

### Webhook – Book Download (Workflow 05)

```json
{
  "name": "Sarah Lee",
  "email": "sarah@company.com",
  "sector": "private",
  "downloadSource": "LinkedIn ad"
}
```

### Google Sheets – Lead Tracking Format (Workflows 04 & 05)

| leadName/name | organisation | sector | leadEmail/email | status | createdDate |
|--------------|-------------|--------|----------------|--------|-------------|
| John Smith | Ministry of Education | public | john@moe.gov.sg | new | 2026-03-15 |

Status progression: `new` → `contacted` → `followed up` or `engaged`  (Workflow 04)
Status progression: `new` → `nurtured` (Workflow 05)

### Google Sheets – Master Insights Format (Workflow 06)

| theme | quote | workshopApplication | interviewee | interviewDate |
|-------|-------|-------------------|-------------|--------------|
| Cognitive Sloth | "People default to AI answers..." | Use as opening exercise in Thinking with AI workshop | Dr. Jane Doe | 2026-03-10 |

### Environment Variables

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Your Anthropic API key for Claude access |
| `LINKEDIN_ACCESS_TOKEN` | LinkedIn OAuth2 access token for publishing |
| `LINKEDIN_PERSON_URN` | Your LinkedIn person URN (e.g., `abc123def`) |
