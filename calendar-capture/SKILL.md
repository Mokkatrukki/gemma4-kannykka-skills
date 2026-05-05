---
name: calendar-capture
description: Extracts calendar events from photos, screenshots, or text descriptions and saves them to the device calendar. Use when the user wants to add an event to their calendar.
---

# Calendar Capture

Extracts event details from an image or text and saves them to the local calendar, checking for duplicates first.

## Instructions

This skill does not call `run_js` directly. It coordinates between the user and the `calendar-storage` skill.
When saving or searching events, call the `calendar-storage` skill using the `run_js` tool:
- script name: index.html
- data: JSON string as described in the actions below.

## When to use

- User shares a photo (invitation, appointment reminder, flyer, screen capture)
- User describes an event verbally ("dentist appointment Tuesday at 2pm")
- User pastes text containing event information

## Step-by-step process

### 1. Extract event details

From the image or text, identify:

| Field | Format | Required | Notes |
|-------|--------|----------|-------|
| `name` | string | yes | Event title |
| `date` | YYYY-MM-DD | yes | Convert relative dates using today's date |
| `startTime` | HH:MM | yes | 24h format |
| `endTime` | HH:MM | no | Omit if not found |
| `location` | string | no | Address or place name |
| `description` | string | no | Extra notes from source |

If date or time is ambiguous, ask the user to clarify before proceeding.

### 2. Check for duplicates (RAG)

Call `calendar-storage` with:
```json
{ "action": "search", "query": "<event name>", "date": "<YYYY-MM-DD>" }
```

- If results contain an event with same name and date → ask user: "Found existing event '<name>' on <date>. Update it?" 
- If user confirms → call with `action: "update"` using the found `id`
- If no match → proceed to step 3

### 3. Save the event

**New event:**
```json
{
  "action": "add",
  "name": "Hammaslääkäri",
  "date": "2026-05-10",
  "startTime": "14:00",
  "endTime": "15:00",
  "location": "Keskusklinikka",
  "description": "Muista ottaa vakuutuskortti"
}
```

**Update existing:**
```json
{
  "action": "update",
  "id": "evt_1746873600000",
  "startTime": "15:00"
}
```

### 4. Confirm to user

After successful save, confirm:
- Event name and date/time
- Whether it was added or updated
- Mention that `calendar-sync` skill can push it to Google Calendar

## Error handling

- Missing required fields → ask user before calling storage
- Storage returns `{ "error": "..." }` → show error, offer to retry
- Ambiguous date ("next Friday") → resolve using today's date, confirm with user
