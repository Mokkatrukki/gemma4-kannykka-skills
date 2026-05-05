---
name: calendar-storage
description: Stores and manages calendar events in device local storage. Use when the user wants to add, update, delete, list, or search calendar events.
---

# Calendar Storage

## Instructions

Call the `run_js` tool with the following parameters:
- script name: index.html
- data: A JSON string with an `action` field and action-specific fields below.

Local calendar database using device localStorage. Supports full CRUD and fuzzy search for duplicate detection.

## Data schema

Each event:
```json
{
  "id": "evt_1746873600000",
  "name": "Hammaslääkäri",
  "date": "2026-05-10",
  "startTime": "14:00",
  "endTime": "15:00",
  "location": "Keskusklinikka",
  "description": "Muista ottaa vakuutuskortti",
  "synced": false,
  "createdAt": 1746873600000,
  "updatedAt": 1746873600000
}
```

`synced: false` means not yet pushed to Google Calendar via calendar-sync.

## Actions

### `add`
Add new event.
```json
{ "action": "add", "name": "...", "date": "YYYY-MM-DD", "startTime": "HH:MM", "endTime": "HH:MM", "location": "...", "description": "..." }
```
Returns: `{ "result": "Added: evt_1746873600000" }`

### `update`
Update existing event by id. Only provided fields are changed.
```json
{ "action": "update", "id": "evt_...", "startTime": "15:00" }
```
Returns: `{ "result": "Updated: evt_..." }`

### `delete`
Delete event by id.
```json
{ "action": "delete", "id": "evt_..." }
```
Returns: `{ "result": "Deleted: evt_..." }`

### `list`
List upcoming events. Optional date range filter.
```json
{ "action": "list", "from": "2026-05-01", "to": "2026-05-31" }
```
Returns: `{ "result": "<JSON array sorted by date+time>" }`

### `search`
Fuzzy search by name and/or date. Used for duplicate detection before adding.
```json
{ "action": "search", "query": "hammaslääkäri", "date": "2026-05-10" }
```
Returns: `{ "result": "<JSON array of matches>" }` — empty array if none found.

### `unsynced`
Returns all events where `synced: false`. Used by calendar-sync.
```json
{ "action": "unsynced" }
```
Returns: `{ "result": "<JSON array>" }`

### `mark_synced`
Mark event as synced after successful Google Calendar push.
```json
{ "action": "mark_synced", "id": "evt_...", "googleEventId": "abc123" }
```
Returns: `{ "result": "Synced: evt_..." }`
