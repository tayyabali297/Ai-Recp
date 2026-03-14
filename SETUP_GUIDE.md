# Dental Care Ireland — AI Receptionist Booking System
## Complete Setup Guide

---

## OVERVIEW

This guide walks you through setting up the complete AI receptionist booking workflow:

```
Vapi (Emma) → n8n Webhook → Validate → Google Sheets → Google Calendar → Gmail → Response to Vapi
```

**Files in this project:**
- `n8n-workflow.json` — Import this into n8n
- `vapi-tool-schema.json` — Reference for configuring Vapi's Custom Tool
- `SETUP_GUIDE.md` — This file

---

## STEP 1: Set Up Google Sheets

### 1a. Create the Spreadsheet

1. Go to [sheets.google.com](https://sheets.google.com) and create a new spreadsheet
2. Name it: **"Dental Bookings"**
3. Create **Sheet 1** and rename it to **"Bookings"**
4. Add these headers in Row 1 (exactly as shown, no typos):

| A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|
| Name | Phone | Email | Service | Date | Time | Status | Timestamp | Booking ID |

5. Create **Sheet 2** and rename it to **"Logs"**
6. Add these headers in Row 1:

| A | B | C | D | E | F | G | H | I | J |
|---|---|---|---|---|---|---|---|---|---|
| Timestamp | Booking ID | Name | Phone | Email | Service | Date | Time | Status | Raw Valid |

### 1b. Get the Spreadsheet ID

Your spreadsheet URL looks like:
```
https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms/edit
```

The Spreadsheet ID is the long string between `/d/` and `/edit`:
```
1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms
```

**Copy this ID — you'll need it in Step 3.**

---

## STEP 2: Set Up n8n Credentials

You need to create **3 credentials** in n8n before importing the workflow.

### 2a. Google Sheets OAuth2

1. In n8n, go to **Settings → Credentials → New Credential**
2. Search for **"Google Sheets OAuth2 API"**
3. Follow the OAuth flow to connect your Google account
4. Name it: **"Google Sheets account"**
5. **Copy the credential ID** (visible in the URL when editing it)

### 2b. Google Calendar OAuth2

1. Create another new credential
2. Search for **"Google Calendar OAuth2 API"**
3. Connect the **same Google account** (so it accesses your primary calendar)
4. Name it: **"Google Calendar account"**
5. **Copy the credential ID**

> **Important:** The Google account you connect must own the calendar you want events added to.

### 2c. Gmail OAuth2

1. Create another new credential
2. Search for **"Gmail OAuth2"**
3. Connect the Gmail account that will **send** the confirmation emails
4. Name it: **"Gmail account"**
5. **Copy the credential ID**

> **Tip:** This can be the same Google account as above, or a dedicated clinic email account.

---

## STEP 3: Import the n8n Workflow

1. In n8n, click **"+"** to create a new workflow (or go to Workflows → New)
2. Click the **three-dot menu (⋮)** in the top right → **"Import from File"**
3. Upload `n8n-workflow.json`
4. The workflow will appear with all 11 nodes connected

### 3a. Update the Spreadsheet ID (do this in both Sheets nodes)

1. Click **"Sheets - Log to Logs"** node
2. In the **Document** field, replace `YOUR_SPREADSHEET_ID_HERE` with your actual Spreadsheet ID
3. Click **"Sheets - Append to Bookings"** node
4. Replace `YOUR_SPREADSHEET_ID_HERE` again

### 3b. Update Credentials (do this in all service nodes)

For each node below, open it and select the matching credential:

| Node | Credential to Select |
|------|---------------------|
| Sheets - Log to Logs | Google Sheets account |
| Sheets - Append to Bookings | Google Sheets account |
| Calendar - Create Event | Google Calendar account |
| Gmail - Patient Confirmation | Gmail account |
| Gmail - Admin Alert | Gmail account |

### 3c. Set Workflow Timezone

1. Click the **Settings** icon (gear) in the top toolbar
2. Set **Timezone** to: `Europe/Dublin`
3. Save settings

### 3d. Activate the Workflow

1. Click **"Save"** (top right)
2. Toggle the **Active** switch to ON (top right, turns green)

---

## STEP 4: Get Your Webhook URL

1. Click the **"Webhook - Receive Booking"** node
2. Copy the **Production URL** (not the test URL)

It will look like one of these:
```
https://your-n8n-instance.cloud/webhook/dental-booking
https://your-domain.com:5678/webhook/dental-booking
```

> **Note:** The webhook URL only works when the workflow is **Active**. If the workflow is inactive, Vapi will get a connection error.

**Keep this URL — you'll paste it into Vapi next.**

---

## STEP 5: Configure Vapi Custom Tool

### 5a. Create the Tool in Vapi

1. Log into your Vapi dashboard at [dashboard.vapi.ai](https://dashboard.vapi.ai)
2. Go to **Tools** → **Create Tool**
3. Select **"Custom Tool"** (or "Function")

### 5b. Fill in the Tool Settings

| Field | Value |
|-------|-------|
| Tool Name | `book_appointment` |
| Description | `Books a dental appointment for a patient at Dental Care Ireland. Call this after collecting patient name, phone, email, service needed, and appointment date.` |
| Server URL | Your n8n webhook URL from Step 4 |
| Method | POST |
| Timeout | 20 seconds |

### 5c. Add Parameters

Add these 6 parameters one by one:

**Parameter 1:**
- Key: `name`
- Type: `string`
- Description: `Patient's full name (first and last name)`
- Required: ✅ Yes

**Parameter 2:**
- Key: `phone`
- Type: `string`
- Description: `Patient's phone number in any format`
- Required: ✅ Yes

**Parameter 3:**
- Key: `email`
- Type: `string`
- Description: `Patient's email address for confirmation`
- Required: ✅ Yes

**Parameter 4:**
- Key: `service`
- Type: `string`
- Description: `Type of dental service: checkup, cleaning, filling, extraction, whitening, consultation, emergency, orthodontic, root canal`
- Required: ✅ Yes

**Parameter 5:**
- Key: `date`
- Type: `string`
- Description: `Appointment date in YYYY-MM-DD format (e.g. 2024-03-15)`
- Required: ✅ Yes

**Parameter 6:**
- Key: `time`
- Type: `string`
- Description: `Appointment time in HH:MM 24-hour format (e.g. 14:30). Optional - defaults to 14:00 if not provided.`
- Required: ❌ No

### 5d. Save the Tool

Click **Save** / **Create Tool**.

---

## STEP 6: Attach Tool to Emma (Your Vapi Assistant)

1. In Vapi, go to **Assistants** → click on Emma (your dental receptionist)
2. Go to the **Tools** section / tab
3. Add the **"book_appointment"** tool you just created
4. Save the assistant

### 5e. Update Emma's System Prompt

Add this section to Emma's system prompt (paste at the end or in the instructions section):

```
## Booking Appointments

When a patient wants to book an appointment, collect information in this order:
1. Full name (first and last)
2. Phone number
3. Email address — ask them to spell it out to confirm
4. Service needed (checkup, cleaning, filling, whitening, extraction, etc.)
5. Preferred date — confirm you heard it correctly
6. Preferred time — optional, tell them default is 2:00 PM

IMPORTANT: Only call the book_appointment tool after you have confirmed ALL required fields.

After the tool responds:
- If status is "success": Say "Your appointment is confirmed! I've sent a confirmation email to [email]. Your booking reference is [appointment_id]."
- If you receive an error: Apologise and say "I'm sorry, there seems to be a technical issue. Please call us directly at 046 902 1372 and we'll get you booked right away."
```

---

## TESTING CHECKLIST

Use this checklist to verify everything is working before going live.

### Phase 1: Test n8n Webhook Directly

Send a test POST request using curl or Postman:

```bash
curl -X POST https://YOUR-N8N-URL/webhook/dental-booking \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Patient",
    "phone": "086 111 2222",
    "email": "YOUR-EMAIL@gmail.com",
    "service": "checkup",
    "date": "2024-12-20",
    "time": "10:00"
  }'
```

**Expected result:**
```json
{
  "status": "success",
  "message": "Booking confirmed",
  "appointment_id": "DCN-XXXXXXXXX-XXXX",
  "booking_details": {
    "name": "Test Patient",
    "date": "2024-12-20",
    "time": "10:00"
  }
}
```

**Verify in n8n:**
- [ ] Workflow execution shows green (success) in the Executions tab
- [ ] All nodes are green

**Verify in Google Sheets:**
- [ ] New row appears in "Bookings" sheet with all columns filled
- [ ] New row appears in "Logs" sheet
- [ ] Timestamp is in Dublin timezone
- [ ] Booking ID looks like `DCN-XXXXXXXX-XXXX`

**Verify in Google Calendar:**
- [ ] New event appears on your primary calendar
- [ ] Event title: "Test Patient - checkup"
- [ ] Event time: 10:00 - 10:30
- [ ] Event location: 15 Watergate Street, Navan, Co. Meath, Ireland
- [ ] Description includes phone, email, service, booking ID

**Verify in Gmail:**
- [ ] Patient receives confirmation email at YOUR-EMAIL@gmail.com
- [ ] Email subject: "Appointment Confirmed - Dental Care Ireland"
- [ ] Email contains correct date, time, service, booking ID
- [ ] Email looks professional (HTML styled with logo/colors)

---

### Phase 2: Test Validation (Error Cases)

**Test missing email:**
```bash
curl -X POST https://YOUR-N8N-URL/webhook/dental-booking \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Patient",
    "phone": "086 111 2222",
    "service": "checkup",
    "date": "2024-12-20"
  }'
```
- [ ] Returns HTTP 400 status
- [ ] Response body contains `"status": "error"` and error details

**Test invalid date format:**
```bash
curl -X POST https://YOUR-N8N-URL/webhook/dental-booking \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Patient",
    "phone": "086 111 2222",
    "email": "test@test.com",
    "service": "checkup",
    "date": "20/12/2024"
  }'
```
- [ ] Returns HTTP 400 with "Date must be in YYYY-MM-DD format" error

**Test invalid email:**
```bash
curl -X POST https://YOUR-N8N-URL/webhook/dental-booking \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Patient",
    "phone": "086 111 2222",
    "email": "not-an-email",
    "service": "checkup",
    "date": "2024-12-20"
  }'
```
- [ ] Returns HTTP 400 with "Invalid email format" error

---

### Phase 3: Test Without Time (Default Time)

```bash
curl -X POST https://YOUR-N8N-URL/webhook/dental-booking \
  -H "Content-Type: application/json" \
  -d '{
    "name": "No Time Patient",
    "phone": "086 999 8888",
    "email": "YOUR-EMAIL@gmail.com",
    "service": "cleaning",
    "date": "2024-12-21"
  }'
```
- [ ] Returns 200 success
- [ ] Calendar event is at 14:00 - 14:30 (default)
- [ ] Confirmation email shows time as "14:00"

---

### Phase 4: Test Vapi Integration

1. Call your Vapi assistant (Emma)
2. Say: "I'd like to book a dental appointment"
3. Provide all details when asked:
   - Name: "Sarah Test"
   - Phone: "086 555 6666"
   - Email: your real email
   - Service: "checkup"
   - Date: a future date
   - Time: "11:00"
4. Listen for Emma to confirm the booking

- [ ] Emma collects all information naturally
- [ ] Emma calls the `book_appointment` tool
- [ ] Emma confirms the booking with the booking ID
- [ ] Confirmation email arrives in your inbox
- [ ] Calendar event appears in Google Calendar
- [ ] Row appears in Google Sheets

---

## TROUBLESHOOTING

### "Webhook not found" error
- Ensure the workflow is **Active** (toggle is green)
- Check you're using the **Production URL** not the Test URL
- The workflow must be active for the production webhook to respond

### Google Sheets errors
- Verify the Spreadsheet ID is correct (check it in the URL of your spreadsheet)
- Ensure the sheet names are exactly "Bookings" and "Logs" (case-sensitive)
- Ensure column headers in Row 1 match exactly what's in the node configuration
- Re-authenticate your Google Sheets credential if expired

### Google Calendar errors
- Ensure the credential is connected to the right Google account
- Make sure the calendar is set to "primary" in the node
- Check that the Google account has Calendar access

### Gmail not sending
- Re-authenticate the Gmail credential
- Check Gmail hasn't blocked the app (check Google Account → Security → Third-party apps)
- Try using a Google Workspace account if Gmail personal is restricted

### Admin alert emails not sending
- Verify the admin email `majesticson297@gmail.com` is correct
- Check the Gmail - Admin Alert node credential is set

### Vapi not calling the tool
- Ensure the `book_appointment` tool is attached to Emma's assistant
- Check Emma's system prompt instructs her to collect all fields before calling the tool
- Verify the Server URL in the tool matches your actual n8n webhook URL

---

## WORKFLOW ARCHITECTURE DIAGRAM

```
                          ┌─────────────────────┐
                          │  Vapi (Emma)         │
                          │  AI Phone Receptionist│
                          └──────────┬──────────┘
                                     │ POST /webhook/dental-booking
                                     ▼
                          ┌─────────────────────┐
                          │ Webhook - Receive    │
                          │ Booking              │
                          └──────────┬──────────┘
                                     │
                                     ▼
                          ┌─────────────────────┐
                          │ Code - Process &    │
                          │ Validate            │
                          │ • Validate fields   │
                          │ • Generate Booking  │
                          │   ID (DCN-xxx-xxxx) │
                          │ • Build email HTML  │
                          │ • Calculate times   │
                          └──────────┬──────────┘
                                     │
                              ┌──────┴───────┐
                              │ IF - Valid?  │
                              └──┬───────┬──┘
                         TRUE ◄──┘       └──► FALSE
                             │                 │
                             ▼                 ▼
                   ┌─────────────────┐  ┌─────────────┐
                   │ Sheets - Log to │  │ Respond 400 │
                   │ Logs (debug)    │  │ (Error JSON)│
                   └────────┬────────┘  └─────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │ Sheets - Append │
                   │ to Bookings     │
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │ Calendar -      │
                   │ Create Event    │
                   │ (30 min event)  │
                   └────────┬────────┘
                            │
                     ┌──────┴───────┐
                     │ IF - Error?  │
                     └──┬───────┬──┘
               ERROR ◄──┘       └──► SUCCESS
                   │                     │
                   ▼                     ▼
          ┌─────────────────┐  ┌─────────────────┐
          │ Gmail - Admin   │  │ Gmail - Patient  │
          │ Alert Email     │  │ Confirmation     │
          └────────┬────────┘  └────────┬─────────┘
                   │                    │
                   ▼                    ▼
          ┌─────────────────┐  ┌─────────────────┐
          │ Respond 500     │  │ Respond 200      │
          │ (System Error)  │  │ (Success + ID)   │
          └─────────────────┘  └─────────────────┘
                   │                    │
                   └─────────┬──────────┘
                             ▼
                   ┌─────────────────┐
                   │  Vapi receives  │
                   │  response and   │
                   │  tells patient  │
                   └─────────────────┘
```

---

## CREDENTIALS SUMMARY

| Credential Type | n8n Name | Used By |
|----------------|----------|---------|
| Google Sheets OAuth2 | Google Sheets account | Sheets - Log to Logs, Sheets - Append to Bookings |
| Google Calendar OAuth2 | Google Calendar account | Calendar - Create Event |
| Gmail OAuth2 | Gmail account | Gmail - Patient Confirmation, Gmail - Admin Alert |

---

## MAINTENANCE NOTES

- **Workflow executions** are saved in n8n for debugging. Check `Executions` tab if something goes wrong.
- **Booking IDs** format: `DCN-{unix_timestamp}-{4_random_digits}`. Unique per booking.
- **Default appointment time**: 14:00 (2:00 PM) when patient doesn't specify
- **Appointment duration**: Always 30 minutes (hardcoded in the Code node)
- **Admin alert email**: `majesticson297@gmail.com` — change this in the `Gmail - Admin Alert` node if needed
- **Clinic phone**: `046 902 1372` — appears in patient emails and error messages
- **Clinic address**: `15 Watergate Street, Navan, Co. Meath, Ireland` — appears in calendar events and patient emails

To change any of these, edit the **"Code - Process & Validate"** node's JavaScript.

---

*Built for Dental Care Ireland | Powered by n8n + Vapi + Google Workspace*
