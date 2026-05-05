# Google AI Edge Gallery Skill Examples

## Text-only skill

**Use case:** Pure LLM task, no external state needed.

```markdown
---
name: recipe-suggester
description: Suggests recipes based on available ingredients. Use when user lists ingredients and wants meal ideas.
---

# Recipe Suggester

Given a list of ingredients, suggest 3 recipes the user can make.

## When to use

- User says "what can I cook with X, Y, Z"
- User asks for recipe ideas from what they have

## Instructions

List 3 recipes with:
- Name
- Missing ingredients (if any)
- Estimated prep time
- Brief steps
```

---

## JS skill (localStorage CRUD)

**Use case:** Note keeper with add/list/delete.

**SKILL.md:**
```markdown
---
name: note-keeper
description: Saves, lists, and deletes personal notes. Use when user wants to save a note, see their notes, or delete one.
---

# Note Keeper

Stores personal notes in device localStorage.

## Calling this skill

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | yes | `add`, `list`, `delete` |
| `text` | string | add only | Note content |
| `id` | string | delete only | Note ID to remove |

## Actions

### `add`
Input: `{ "action": "add", "text": "Buy milk" }`
Returns: `{ "result": "Added: item_1746873600000" }`

### `list`
Input: `{ "action": "list" }`
Returns: `{ "result": "[{\"id\":\"item_...\",\"text\":\"Buy milk\",...}]" }`

### `delete`
Input: `{ "action": "delete", "id": "item_1746873600000" }`
Returns: `{ "result": "Deleted: item_1746873600000" }`
```

**scripts/index.html:**
```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"></head>
<body>
<script>
window['ai_edge_gallery_get_result'] = async (data) => {
  try {
    const input = JSON.parse(data);
    const { action } = input;

    const STORAGE_KEY = 'note_keeper_data';
    const load = () => JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
    const save = (items) => localStorage.setItem(STORAGE_KEY, JSON.stringify(items));

    if (action === 'add') {
      const items = load();
      const note = {
        id: `item_${Date.now()}`,
        text: input.text,
        createdAt: Date.now()
      };
      items.push(note);
      save(items);
      return JSON.stringify({ result: `Added: ${note.id}` });
    }

    if (action === 'list') {
      return JSON.stringify({ result: JSON.stringify(load()) });
    }

    if (action === 'delete') {
      save(load().filter(i => i.id !== input.id));
      return JSON.stringify({ result: `Deleted: ${input.id}` });
    }

    return JSON.stringify({ error: `Unknown action: ${action}` });

  } catch (err) {
    return JSON.stringify({ error: err.message });
  }
};
</script>
</body>
</html>
```

---

## JS skill with RAG (search before write)

**Use case:** Calendar storage that avoids duplicate events.

**SKILL.md excerpt (instructions to LLM):**
```markdown
## Duplicate check (RAG)

Before adding a new event, always search first:
1. Call with `{ "action": "search", "query": "<event name>" }`
2. If results contain an event on the same date → use `action: "update"` with its `id`
3. If no match → use `action: "add"`
```

**search action in index.html:**
```javascript
if (action === 'search') {
  const { query, date } = input;
  const items = load();
  const results = items.filter(i => {
    const nameMatch = i.name?.toLowerCase().includes((query || '').toLowerCase());
    const dateMatch = !date || i.date === date;
    return nameMatch && dateMatch;
  });
  return JSON.stringify({ result: JSON.stringify(results) });
}
```

---

## JS+webview skill (interactive dashboard)

**Use case:** Habit tracker with visual dashboard.

**SKILL.md:**
```markdown
---
name: habit-tracker
description: Tracks daily habits and shows a visual streak dashboard. Use when user wants to log a habit, check their streaks, or see habit history.
---

## Actions

### `log`
Records completion of a habit for today.
Input: `{ "action": "log", "habit": "exercise" }`

### `dashboard`
Opens interactive visual dashboard.
Input: `{ "action": "dashboard" }`
Returns: `{ "webview": { "html": "webview.html", "aspectRatio": 1.3 } }`
```

**scripts/index.html action:**
```javascript
if (action === 'dashboard') {
  // Data already in localStorage — webview reads it directly
  return JSON.stringify({ webview: { html: 'webview.html', aspectRatio: 1.3 } });
}
```

**assets/webview.html (simplified):**
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    body { font-family: sans-serif; padding: 16px; }
    .habit { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #eee; }
  </style>
</head>
<body>
<div id="app"></div>
<script>
  const STORAGE_KEY = 'habit_tracker_data';
  const data = JSON.parse(localStorage.getItem(STORAGE_KEY) || '{}');
  
  const app = document.getElementById('app');
  app.innerHTML = Object.entries(data).map(([habit, logs]) =>
    `<div class="habit"><span>${habit}</span><span>${logs.length} days</span></div>`
  ).join('') || '<p>No habits tracked yet.</p>';
</script>
</body>
</html>
```

---

## Native skill (system intent)

**Use case:** Send email via device mail app.

**SKILL.md:**
```markdown
---
name: send-email
description: Sends an email using the device's mail app. Use when user wants to send an email.
---

# Send Email

Uses `run_intent` to open the device mail app pre-filled with recipient, subject, and body.

## Parameters for run_intent

| Intent | Parameter | Description |
|--------|-----------|-------------|
| `send_email` | `extra_email` | Recipient address |
| `send_email` | `extra_subject` | Subject line |
| `send_email` | `extra_text` | Body text |

Call: `run_intent("send_email", { extra_email: "...", extra_subject: "...", extra_text: "..." })`
```
