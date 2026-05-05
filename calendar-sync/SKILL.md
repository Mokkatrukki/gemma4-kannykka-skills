---
name: calendar-sync
description: Syncs local calendar events to Google Calendar using a service account. Use this skill when the user wants to push saved events to Google Calendar, sync their schedule, or asks why an event isn't showing in Google Calendar. Triggers on: "sync to Google Calendar", "push events to Google", "synkkaa kalenteri", "vie tapahtumat Googleen".
metadata:
  require-secret: true
  require-secret-description: |
    This skill needs your Google service account JSON.

    How to get it:
    1. Go to Google Cloud Console → IAM & Admin → Service Accounts
    2. Open your service account → Keys → Add Key → JSON
    3. Open the downloaded .json file and copy its entire contents
    4. Paste the full JSON here

    The service account must have access to your Google Calendar:
    - Open Google Calendar → Settings → your calendar → Share with specific people
    - Add the service account email (looks like name@project.iam.gserviceaccount.com)
    - Give it "Make changes to events" permission
---

# Calendar Sync

Pushes unsynced local events to Google Calendar. Uses service account credentials — no OAuth token expiry.

## When to use

- User asks to sync or push events to Google Calendar
- User notices an event is missing from Google Calendar
- After batch of new events were captured

## Process

### 1. Fetch unsynced events

Call `calendar-storage`:
```json
{ "action": "unsynced" }
```
If empty array → tell user: "All events are already synced."

### 2. Push each event

Call this skill for each event:
```json
{
  "action": "push",
  "calendarId": "primary",
  "event": {
    "id": "evt_...",
    "name": "Hammaslääkäri",
    "date": "2026-05-10",
    "startTime": "14:00",
    "endTime": "15:00",
    "location": "Keskusklinikka",
    "description": "..."
  }
}
```

### 3. Mark as synced

After each successful push, call `calendar-storage`:
```json
{ "action": "mark_synced", "id": "evt_...", "googleEventId": "<returned id>" }
```

### 4. Report results

Tell user how many events synced and if any failed.

## Error handling

- `401` → service account key invalid, ask user to re-paste secret
- `403` → calendar not shared with service account email
- `409` → already exists, mark as synced anyway
- Network error → retry once, then report

## Calendar ID

Default: `"primary"`. For a shared calendar, user provides the full calendar ID (looks like an email address).
