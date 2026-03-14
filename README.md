# n8n MCP Voice Receptionist Backend

A modular n8n workflow that acts as an **MCP (Model Context Protocol) server** for a Vapi Voice AI agent. Handles CRM lookups, calendar availability checking, appointment booking, rescheduling, and cancellation — all backed by Google Sheets and Google Calendar.

---

## Repository Structure

```
├── workflows/
│   ├── mcp_voice_receptionist.json      # Main MCP workflow (7 tool branches)
│   └── vapi_post_call_webhook.json      # Post-call report logger
├── vapi/
│   └── mcp_tool_config.json             # Tool schemas to paste into Vapi
├── docs/
│   └── SETUP_GUIDE.md                   # Full step-by-step setup instructions
└── README.md
```

---

## Seven MCP Tools

| Tool | Description | Inputs | Storage |
|---|---|---|---|
| `client_lookup` | Search CRM by email | `email` | Google Sheets (CRM) |
| `new_client_crm` | Add new client to CRM | `name`, `email`, `phone` | Google Sheets (CRM) |
| `check_availability` | List free/busy slots | `startTime`, `endTime` | Google Calendar |
| `book_event` | Create appointment (1 hr) | `email`, `name`, `startTime` | Google Calendar + Booking Log |
| `lookup_appointment` | Find appointment by email | `email`, `startTime?` | Google Sheets (Booking Log) |
| `update_appointment` | Reschedule by eventId | `eventId`, `newStartTime` | Google Calendar + Booking Log |
| `delete_appointment` | Cancel by eventId | `eventId` | Google Calendar + Booking Log |

---

## Post-Call Webhook

A separate **Webhook Trigger** receives Vapi's End-of-Call Report and appends the call summary, outcome, transcript, and metadata to a **Call History** Google Sheet.

---

## Quick Start

1. **Import** `workflows/mcp_voice_receptionist.json` into n8n (v1.22+).
2. **Import** `workflows/vapi_post_call_webhook.json`.
3. Set up **Google Sheets** and **Google Calendar** OAuth2 credentials in n8n.
4. Replace all `REPLACE_WITH_...` placeholders in the nodes with your spreadsheet IDs and credential IDs.
5. **Activate** both workflows.
6. Copy the **MCP Server URL** from the MCP Server Trigger node.
7. Paste into **Vapi → Tools → MCP Server**.
8. Use the tool schemas in `vapi/mcp_tool_config.json` if using Vapi Custom Tools instead.

See [`docs/SETUP_GUIDE.md`](docs/SETUP_GUIDE.md) for the full walkthrough.

---

## Design Principles

- **No AI nodes** inside sub-flows — pure logic nodes (Switch, If, Set, Code) for minimal latency.
- **ISO 8601** enforced on all date/time fields.
- **Lean error handling** — failed lookups return clear string errors (e.g., `"Error: Appointment not found"`) so the Voice AI can relay them to the caller.
- **Single MCP Trigger** entry point with a Switch router for clean, maintainable branching.
- **Atomic operations** — `book_event` writes to Calendar and Booking Log in the same branch before responding.
