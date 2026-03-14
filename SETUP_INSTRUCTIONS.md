# Dental Care Ireland — AI Receptionist Booking System
## Complete Setup Guide

---

## WHAT YOU'RE BUILDING

```
Vapi (Emma AI) → n8n Webhook → Validate → Google Sheets (log) → Google Calendar → Gmail (patient) → 200 OK to Vapi
                                              ↓ (on fail)
                                         Admin Alert Email → 500 Error to Vapi
```

---

## PART 1: GOOGLE SHEETS SETUP

### Step 1 — Create the spreadsheet

1. Go to [Google Sheets](https://sheets.google.com) and create a new spreadsheet
2. Name it exactly: **Dental Bookings**
3. Create two sheets (tabs at the bottom):

**Sheet 1 — "Bookings"** — Add these headers in Row 1:
| A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|
| Name | Phone | Email | Service | Date | Time | Status | Timestamp | BookingID |

**Sheet 2 — "Logs"** — Add these headers in Row 1:
| A | B | C | D | E | F | G | H | I | J |
|---|---|---|---|---|---|---|---|---|---|
| Timestamp | BookingID | Name | Phone | Email | Service | Date | Time | LogType | RawTimestamp |

### Step 2 — Get the Spreadsheet ID

The URL of your sheet looks like:
```
https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms/edit
```
The long string between `/d/` and `/edit` is your **Spreadsheet ID**.

Copy it — you'll need it in n8n.

---

## PART 2: GOOGLE CREDENTIALS IN N8N

You need **3 credentials** in n8n. Set them up before importing the workflow.

### Credential 1: Google Sheets OAuth2

1. In n8n: **Settings → Credentials → New Credential → Google Sheets OAuth2 API**
2. Follow the OAuth flow — sign in with the Google account that owns the spreadsheet
3. Name it: `Google Sheets account`

### Credential 2: Google Calendar OAuth2

1. In n8n: **Settings → Credentials → New Credential → Google Calendar OAuth2 API**
2. Follow the OAuth flow — sign in with the Google account that owns the calendar
3. Name it: `Google Calendar account`

### Credential 3: Gmail OAuth2

1. In n8n: **Settings → Credentials → New Credential → Gmail OAuth2 API**
2. Follow the OAuth flow — sign in with the Gmail account that will send emails
3. Name it: `Gmail account`

> **Tip:** All three can be the same Google account. Use the clinic's Google Workspace account for production.

---

## PART 3: IMPORT THE WORKFLOW INTO N8N

1. Open your n8n instance
2. Click **"+ New Workflow"** (or open an existing one)
3. Click the **three-dot menu (⋮)** in the top-right → **"Import from File"**
4. Select `dental_booking_workflow.json` from this repository
5. The workflow will appear with all nodes connected

### After Import — Update These Values

Open each node and replace the placeholder values:

#### In "Log Request to Sheets (Logs Tab)" node:
- Click on the node → find **Document** field
- Replace `SPREADSHEET_ID_HERE` with your actual Spreadsheet ID
- Set credential to `Google Sheets account`

#### In "Log Booking to Google Sheets" node:
- Same as above — replace `SPREADSHEET_ID_HERE`
- Set credential to `Google Sheets account`

#### In "Create Google Calendar Event" node:
- Set credential to `Google Calendar account`
- Calendar is already set to `primary` — change if needed

#### In "Send Confirmation Email to Patient" node:
- Set credential to `Gmail account`

#### In "Send Admin Alert Email" node:
- Set credential to `Gmail account`
- Verify the **To** field is `majesticson297@gmail.com`

6. Click **"Save"** (Ctrl+S)
7. Click the **"Active"** toggle (top right) to enable the workflow

---

## PART 4: GET YOUR WEBHOOK URL

1. Click on the **"Webhook - Receive Vapi Booking"** node
2. You'll see two URLs:
   - **Test URL** (use during testing): `https://your-n8n-domain/webhook-test/dental-booking`
   - **Production URL** (use in Vapi): `https://your-n8n-domain/webhook/dental-booking`
3. Copy the **Production URL** for Vapi

> If using n8n Cloud, the domain is something like `https://yourname.app.n8n.cloud`
> If self-hosted, it's your server domain.

---

## PART 5: CONFIGURE VAPI CUSTOM TOOL

### Step 1 — Create the tool

1. Log into [Vapi Dashboard](https://dashboard.vapi.ai)
2. Go to **Tools** in the left sidebar
3. Click **"+ Create Tool"**
4. Select **"Function"** type

### Step 2 — Fill in tool settings

| Field | Value |
|-------|-------|
| Tool Name | `book_appointment` |
| Description | `Books a patient appointment at Dental Care Ireland. Call this after collecting name, phone, email, service, and date.` |
| Server URL | *(paste your n8n Production Webhook URL here)* |
| Method | POST |
| Timeout | 20 seconds |

### Step 3 — Paste the Parameters Schema

In the **Parameters** section, paste this JSON:

```json
{
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "The patient's full name, e.g. 'John Murphy'."
    },
    "phone": {
      "type": "string",
      "description": "The patient's phone number, e.g. '+353 87 123 4567'."
    },
    "email": {
      "type": "string",
      "description": "The patient's email address for the confirmation."
    },
    "service": {
      "type": "string",
      "description": "Type of dental appointment: Checkup, Cleaning, Filling, Extraction, Whitening, Consultation, or Emergency."
    },
    "date": {
      "type": "string",
      "description": "Appointment date in YYYY-MM-DD format, e.g. '2026-04-15'."
    },
    "time": {
      "type": "string",
      "description": "Appointment time in HH:MM 24-hour format, e.g. '14:00'. Optional — defaults to 14:00."
    }
  },
  "required": ["name", "phone", "email", "service", "date"]
}
```

### Step 4 — Add Tool Messages (optional but recommended)

| State | Message |
|-------|---------|
| Request Start | `Let me book that appointment for you right now.` |
| Request Complete | `Your appointment has been confirmed! You'll receive a confirmation email shortly.` |
| Request Failed | `I'm sorry, I wasn't able to book that. Please call us at 046 902 1372.` |
| Request Delayed | `I'm still processing your booking, please bear with me for just a moment.` |

### Step 5 — Add the tool to Emma's assistant

1. Go to your **Assistant** configuration in Vapi
2. Scroll to **Tools** section
3. Click **"+ Add Tool"** and select `book_appointment`
4. Save the assistant

---

## PART 6: TESTING CHECKLIST

Work through these tests in order.

### Test 1 — Webhook Validation (400 errors)

Send these requests using curl or Postman to the **Test URL**:

```bash
# Test 1a: Missing required fields (expect 400)
curl -X POST "YOUR_TEST_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Patient"}'

# Expected response:
# {"status":"error","message":"Validation failed: phone is required, email is required..."}
```

```bash
# Test 1b: Invalid email format (expect 400)
curl -X POST "YOUR_TEST_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","phone":"087123","email":"notanemail","service":"Checkup","date":"2026-04-15"}'
```

```bash
# Test 1c: Invalid date format (expect 400)
curl -X POST "YOUR_TEST_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","phone":"087123","email":"t@t.com","service":"Checkup","date":"15/04/2026"}'
```

### Test 2 — Full Successful Booking (200)

```bash
curl -X POST "YOUR_TEST_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sarah O Brien",
    "phone": "+353 87 555 1234",
    "email": "YOUR_TEST_EMAIL@gmail.com",
    "service": "Cleaning",
    "date": "2026-04-20",
    "time": "10:30"
  }'
```

**Verify all 5 things happened:**
- [ ] Response is HTTP 200 with `"status": "success"` and a `appointment_id`
- [ ] Row appeared in Google Sheets "Bookings" tab
- [ ] Row appeared in Google Sheets "Logs" tab
- [ ] Event created in Google Calendar for April 20 at 10:30
- [ ] Confirmation email received at your test email

### Test 3 — Booking without time (defaults to 14:00)

```bash
curl -X POST "YOUR_TEST_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Michael Walsh",
    "phone": "046 902 5678",
    "email": "YOUR_TEST_EMAIL@gmail.com",
    "service": "Checkup",
    "date": "2026-04-21"
  }'
```

**Verify:** Calendar event is at 14:00.

### Test 4 — Vapi end-to-end

1. Call your Vapi phone number
2. Speak to Emma: *"I'd like to book a cleaning for next Monday at 2pm. My name is Test Patient, phone 087 123 4567, email test@example.com"*
3. Emma should call the tool and confirm the booking verbally
4. Check all 5 items from Test 2 above

---

## ARCHITECTURE REFERENCE

```
┌─────────────────────────────────────────────────────────────┐
│                    n8n Workflow Flow                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Webhook] → [Validate+PrepareData]                         │
│                      ↓              ↓                       │
│               [Valid=true]    [Valid=false]                  │
│                      ↓              ↓                       │
│              [PrepareLog]    [Return 400]                    │
│                      ↓                                      │
│              [SheetsLogs]   ← continueOnFail=true           │
│                      ↓                                      │
│            [SheetsBookings] ← continueOnFail=true           │
│                      ↓                                      │
│          [GoogleCalendar] ──error──→ [PrepareAlert]         │
│                      ↓                       ↓              │
│           [GmailPatient]          [GmailAdminAlert]         │
│           continueOnFail                     ↓              │
│                      ↓              [Return 500]            │
│              [Return 200]                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## TROUBLESHOOTING

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Webhook returns 404 | Workflow not activated | Toggle "Active" switch in n8n |
| Google Sheets error | Wrong Spreadsheet ID | Copy ID from URL bar in Sheets |
| Calendar event not created | OAuth scope missing | Re-authenticate Google Calendar credential |
| Email not sending | Gmail credential expired | Re-authenticate Gmail credential in n8n |
| Vapi tool times out | n8n workflow taking >20s | Check n8n server load; increase Vapi timeout to 30s |
| 400 on valid data | Body parsing issue | Check n8n node reads `body.name` not just `name` |
| No rows in Logs sheet | Sheet name mismatch | Ensure tab is named exactly "Logs" (capital L) |
