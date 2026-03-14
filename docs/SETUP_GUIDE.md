# Setup Guide — n8n MCP Voice Receptionist Backend

## Overview

This system exposes **seven MCP tools** to a Vapi voice AI agent via an n8n workflow. Google Sheets acts as the CRM and booking log; Google Calendar handles scheduling.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| n8n v1.22+ | MCP Server Trigger requires this version minimum |
| Google Cloud project | For Sheets + Calendar OAuth2 |
| Vapi account | With MCP or Custom Tools support |
| Two Google Spreadsheets | CRM sheet + Booking Log (+ optional Call History) |

---

## Step 1 — Create Google Spreadsheets

### CRM Spreadsheet
- Create a new Google Sheet and name it **"CRM"** (or any name you prefer).
- Add a tab named **`CRM`** with these exact headers in row 1:

```
A: Name | B: Email | C: Phone | D: Status | E: CreatedAt
```

### Booking Log Spreadsheet
- Create another Google Sheet (can be in the same file as a second tab, or a separate file).
- Add a tab named **`BookingLog`** with these headers in row 1:

```
A: Email | B: Name | C: StartTime | D: EventID | E: Status | F: CreatedAt | G: UpdatedAt
```

### Call History Spreadsheet  *(for the post-call webhook)*
- Add a tab named **`CallHistory`** with these headers:

```
A: CallID | B: PhoneNumber | C: StartedAt | D: EndedAt | E: DurationSec | F: EndedReason | G: Summary | H: Outcome | I: Transcript | J: RecordingURL | K: LoggedAt
```

Record the **Spreadsheet IDs** from each URL:
```
https://docs.google.com/spreadsheets/d/SPREADSHEET_ID_HERE/edit
```

---

## Step 2 — Configure Google OAuth2 Credentials in n8n

1. In n8n, go to **Settings → Credentials → Add Credential**.
2. Select **Google Sheets OAuth2 API** — follow the OAuth2 setup wizard.
3. Add a second credential: **Google Calendar OAuth2 API**.
4. Make sure both credentials grant access to the Google account that owns the sheets and calendar.

> **Tip:** The same Google OAuth2 credential set can often be reused for both Sheets and Calendar if you grant both scopes.

---

## Step 3 — Import the Workflows

### Main MCP Workflow
1. In n8n, go to **Workflows → Import from File**.
2. Upload `workflows/mcp_voice_receptionist.json`.
3. The workflow will open in the editor.

### Post-Call Webhook Workflow
1. Repeat: **Import from File** → `workflows/vapi_post_call_webhook.json`.

---

## Step 4 — Configure Placeholder Values

In each workflow, update every node that contains `REPLACE_WITH_...`:

### MCP Workflow — Google Sheets nodes (6 nodes total)
| Placeholder | Replace with |
|---|---|
| `REPLACE_WITH_CREDENTIAL_ID` | Your Google Sheets credential ID (visible in n8n credentials) |
| `REPLACE_WITH_CRM_SPREADSHEET_ID` | The Spreadsheet ID from the CRM Google Sheet URL |
| `REPLACE_WITH_BOOKING_LOG_SPREADSHEET_ID` | The Spreadsheet ID from the Booking Log URL |

### MCP Workflow — Google Calendar nodes (4 nodes total)
| Placeholder | Replace with |
|---|---|
| `REPLACE_WITH_CREDENTIAL_ID` | Your Google Calendar credential ID |

> In n8n's UI, clicking any Google Sheets/Calendar node and selecting the credential from the dropdown is the easiest way — you don't need to edit the JSON directly.

### Post-Call Webhook
| Placeholder | Replace with |
|---|---|
| `REPLACE_WITH_CREDENTIAL_ID` | Google Sheets credential ID |
| `REPLACE_WITH_CALL_HISTORY_SPREADSHEET_ID` | Call History spreadsheet ID |

---

## Step 5 — Activate and Get the MCP URL

1. Open the **MCP Voice Receptionist** workflow.
2. Click **Activate** (toggle in the top-right).
3. Click on the **MCP Server Trigger** node.
4. Copy the **MCP Server URL** shown in the node panel — it will look like:
   ```
   https://your-n8n-instance.com/mcp/voice-receptionist
   ```
   or for n8n Cloud:
   ```
   https://app.n8n.cloud/webhook/mcp-voice-receptionist
   ```
5. Keep this URL — you will paste it into Vapi.

For the **Post-Call Webhook**, activate that workflow too and copy the Webhook URL from the Vapi End-of-Call Webhook node.

---

## Step 6 — Configure Vapi

### Option A: MCP Server (recommended if your Vapi plan supports it)
1. In your Vapi assistant, go to **Tools → Add MCP Server**.
2. Set the URL to the MCP Server URL from Step 5.
3. The 7 tools will be discovered automatically.

### Option B: Custom Tools (REST fallback)
1. In Vapi, go to **Tools → Add Custom Tool** for each of the 7 functions.
2. Use the definitions in `vapi/mcp_tool_config.json`.
3. Set **Server URL** to your n8n webhook URL.
4. Set **Method** to `POST`.
5. Paste the JSON schema for each tool's parameters.

### Post-Call Webhook
1. In your Vapi assistant settings, find **Server URL / End-of-Call Report**.
2. Enter the webhook URL from the Post-Call Webhook workflow.

---

## Step 7 — System Prompt for the Voice AI

Add these instructions to your Vapi assistant's system prompt:

```
You are a professional voice receptionist. You have access to these tools:
- client_lookup: Always call this first with the caller's email to check if they are an existing client.
- new_client_crm: Add new clients to the system after confirming they are new.
- check_availability: Check open appointment slots before booking.
- book_event: Book a 1-hour appointment. Store the returned eventId in memory.
- lookup_appointment: Find an existing appointment before modifying it.
- update_appointment: Reschedule using the eventId from lookup_appointment.
- delete_appointment: Cancel using the eventId from lookup_appointment.

Always use ISO 8601 format for dates and times (e.g., 2025-03-15T14:00:00Z).
If a tool returns an error string starting with "Error:", relay that message to the caller politely.
```

---

## Step 8 — Testing Checklist

- [ ] **client_lookup** — Call with a known email; confirm name + status returned.
- [ ] **client_lookup** (new) — Call with unknown email; confirm "New Client" returned.
- [ ] **new_client_crm** — Add a test client; verify row appears in CRM sheet.
- [ ] **check_availability** — Pass a time window with some existing calendar events; verify busy/free slots.
- [ ] **book_event** — Book a test appointment; verify Calendar event created + Booking Log row added.
- [ ] **lookup_appointment** — Look up the test booking by email; confirm eventId returned.
- [ ] **update_appointment** — Reschedule using the eventId; verify Calendar updated + Booking Log shows "Rescheduled".
- [ ] **delete_appointment** — Cancel using the eventId; verify Calendar event deleted + Booking Log shows "Cancelled".
- [ ] **Post-call webhook** — Send a test POST to the webhook URL; verify row appended to Call History sheet.

---

## Google Sheet Column Reference

### CRM Sheet
| Column | Field | Notes |
|---|---|---|
| A | Name | Client full name |
| B | Email | Primary key for lookups |
| C | Phone | Phone number |
| D | Status | Active / Inactive |
| E | CreatedAt | ISO 8601 timestamp |

### Booking Log Sheet
| Column | Field | Notes |
|---|---|---|
| A | Email | Links back to CRM |
| B | Name | Client full name |
| C | StartTime | ISO 8601 appointment start |
| D | EventID | Google Calendar event ID |
| E | Status | Confirmed / Rescheduled / Cancelled |
| F | CreatedAt | When booked |
| G | UpdatedAt | When last modified |

### Call History Sheet
| Column | Field | Notes |
|---|---|---|
| A | CallID | Vapi call ID |
| B | PhoneNumber | Caller's number |
| C | StartedAt | Call start (ISO 8601) |
| D | EndedAt | Call end (ISO 8601) |
| E | DurationSec | Duration in seconds |
| F | EndedReason | Why call ended |
| G | Summary | AI-generated summary |
| H | Outcome | Success evaluation |
| I | Transcript | Full transcript |
| J | RecordingURL | Recording URL if available |
| K | LoggedAt | When logged |

---

## Troubleshooting

| Problem | Solution |
|---|---|
| MCP Trigger URL not visible | Ensure n8n is v1.22+ and the workflow is active |
| Google Sheets returns 0 results | Check the `filtersUI` column name matches your sheet header exactly (case-sensitive) |
| Calendar events not creating | Verify the Google Calendar credential has "calendar.events" write scope |
| Vapi tool not triggering | Check the tool name in Vapi matches exactly: `client_lookup`, `new_client_crm`, etc. |
| "Error: Appointment not found" | Ensure `lookup_appointment` is called before `update_appointment` / `delete_appointment` |
| Post-call webhook 404 | Ensure the Post-Call Webhook workflow is active |

---

## Architecture Diagram

```
Vapi Voice AI
     │
     │  MCP Protocol
     ▼
┌─────────────────────────────────────────────────────────┐
│  MCP Server Trigger                                      │
│       │                                                  │
│  Route by Tool Name (Switch)                             │
│  ┌────┴────┬────────┬────────┬────────┬────────┬──────┐ │
│  │         │        │        │        │        │      │ │
│  ▼         ▼        ▼        ▼        ▼        ▼      ▼ │
│ client  new_crm  avail  book_event lookup update delete │ │
│ _lookup         _check             _appt  _appt  _appt  │
│  │         │        │        │        │        │      │ │
│  ▼         ▼        ▼        ▼        ▼        ▼      ▼ │
│ GSheets GSheets GCal   GCal+    GSheets GCal+  GCal+  │ │
│  CRM     CRM    Events GSheets  BkgLog  GSheets GSheets│ │
└─────────────────────────────────────────────────────────┘

Vapi End-of-Call Report
     │  HTTP POST
     ▼
┌────────────────────────────┐
│  Webhook Trigger            │
│  → Parse Report (Code)      │
│  → Append Call History      │
│  → Respond 200 OK           │
└────────────────────────────┘
```
