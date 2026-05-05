---
name: calendar-sync
description: Syncs local calendar events to Google Calendar using a service account. Use when the user wants to push saved events to Google Calendar or sync their schedule.
metadata:
  require-secret: true
  require-secret-description: Paste your full Google service account JSON (from Cloud Console → IAM → Service Accounts → Keys → Add Key → JSON).
---

# Calendar Sync

Pushes unsynced local events to Google Calendar using a service account — no OAuth token expiry needed.

## Instructions

Call the `run_js` tool with the following parameters:
- script name: index.html
- data: A JSON string with an `action` field and action-specific fields below.

The skill secret (service account JSON) is passed automatically as the second parameter.

## Process

### 1. Fetch unsynced events

Call `calendar-storage` skill with `run_js`:
- data: `{ "action": "unsynced" }`

If result is empty array, tell user: "All events are already synced."

### 2. Push each event

Call this skill with `run_js` for each unsynced event:
- data: JSON string with these fields:
  - `action`: "push"
  - `calendarId`: "primary" (or user-specified calendar ID)
  - `event`: object with `id`, `name`, `date` (YYYY-MM-DD), `startTime` (HH:MM), and optionally `endTime`, `location`, `description`, `googleEventId`

### 3. Mark as synced

After each successful push, call `calendar-storage` skill:
- data: `{ "action": "mark_synced", "id": "<local id>", "googleEventId": "<returned Google id>" }`

### 4. Report results

Tell user how many events synced and if any failed.

## Error handling

- `401` error → service account JSON invalid, ask user to re-enter secret
- `403` error → calendar not shared with service account email address
- `409` error → event already exists in Google Calendar, mark as synced anyway
- Network error → retry once, then report failure

## Calendar ID

Default is `"primary"` (user's main Google Calendar). User can specify a different calendar ID if they have multiple calendars.
