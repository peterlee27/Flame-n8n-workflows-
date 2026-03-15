# Flamecentre n8n Workflows

Comprehensive n8n workflow suite for Flamecentre business automation.

## Workflows

| # | Workflow | Description | Trigger |
|---|---------|-------------|---------|
| 01 | Morning Briefing | Daily briefing with schedule, emails, proposals & AI headlines | Daily @ 7am |

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

### Environment Variables

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Your Anthropic API key for Claude access |
