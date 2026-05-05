---
name: google-gallery-skill
description: Creates and edits skills for the Google AI Edge Gallery platform (github.com/google-ai-edge/gallery). Use this skill whenever the user wants to create a new Gallery skill, scaffold a skill directory, generate the correct file structure, edit an existing Gallery skill, or asks about the Gallery skill format. Triggers on phrases like "create gallery skill", "make a gemma skill", "new skill for gallery", "scaffold gallery skill", "edit gallery skill", "google-gallery-skill".
---

# Google AI Edge Gallery Skill Creator

Creates and edits on-device Gemma skills in the exact format required by the Google AI Edge Gallery platform.

## Skill Types

| Type | Files needed | Use when |
|------|-------------|----------|
| **text-only** | `SKILL.md` only | Pure LLM reasoning, no external logic needed |
| **js** | `SKILL.md` + `scripts/index.html` | Deterministic logic, localStorage, API calls |
| **js+webview** | `SKILL.md` + `scripts/index.html` + `assets/webview.html` | Interactive UI — charts, dashboards, forms |
| **native** | `SKILL.md` only | Device system intents (email, SMS) via `run_intent` |

## Workflow

### Step 1: Gather requirements

Ask the user (if not already known):
1. **Skill name** — kebab-case (e.g., `calendar-storage`, `mood-tracker`)
2. **One-sentence description** — what it does
3. **Type** — text-only / js / js+webview / native
4. **Inputs** — what JSON fields the skill receives (text, base64 image, action string, ...)
5. **Outputs** — text result, base64 image, webview HTML, or iframe URL

For JS skills also clarify:
- What actions/operations are needed (add / update / delete / list / search)?
- Does it need persistence? Use localStorage if yes.

### Step 2: Create directory structure

```
<skill-name>/
├── SKILL.md                    ← always required
├── scripts/
│   └── index.html              ← js and js+webview skills
└── assets/
    └── webview.html            ← js+webview skills only
```

Create in the current working directory unless the user specifies otherwise.

### Step 3: Write SKILL.md

**YAML frontmatter rules:**
- `name`: kebab-case, must match directory name exactly
- `description`: what it does + specific phrases/contexts that should trigger it
- `metadata.homepage`: optional URL to repo or docs
- `metadata.require-secret`: `true` only when API key needed (omit otherwise)
- `metadata.require-secret-description`: shown to user explaining how to get credentials

**Body:** Markdown instructions for the LLM — when to call the skill, exact JSON input fields, how to interpret outputs, error handling.

**SKILL.md template** (replace `~~~` with triple backticks when writing the actual file):
```
~~~ (opening frontmatter)
name: <skill-name>
description: "<what it does. Trigger when: user says X, Y, Z.>"
~~~ (closing frontmatter)

# <Display Name>

<One paragraph: purpose and capabilities.>

## When to use

<Bullet list of trigger conditions — specific phrases and contexts.>

## Calling this skill

Pass JSON with these fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | yes | One of: `add`, `update`, `delete`, `list`, `search` |
| ... | ... | ... | ... |

## Return values

- `{ "result": "..." }` — success, text output
- `{ "result": "...", "image": { "base64": "data:image/png;base64,..." } }` — success with image
- `{ "webview": { "html": "webview.html", "aspectRatio": 1.5 } }` — interactive UI
- `{ "webview": { "iframe": true, "url": "https://...", "aspectRatio": 1.5 } }` — embedded URL
- `{ "error": "..." }` — failure with reason

## Actions

### `add`
Required fields: ...
Returns: `{ "result": "Added: <id>" }`

### `list`
Optional fields: ...
Returns: `{ "result": "<JSON array of items>" }`
```

### Step 4: Write scripts/index.html (JS skills)

The function name and signature are fixed by the platform — do not change them:

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

    // localStorage helpers — prefix key with skill name to avoid collisions
    const STORAGE_KEY = '<skill-name>_data';
    const load = () => JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
    const save = (items) => localStorage.setItem(STORAGE_KEY, JSON.stringify(items));

    if (action === 'add') {
      const items = load();
      const item = {
        id: `item_${Date.now()}`,
        createdAt: Date.now(),
        updatedAt: Date.now(),
        // spread skill-specific fields from input
      };
      items.push(item);
      save(items);
      return JSON.stringify({ result: `Added: ${item.id}` });
    }

    if (action === 'update') {
      const { id, ...updates } = input;
      const items = load().map(i =>
        i.id === id ? { ...i, ...updates, updatedAt: Date.now() } : i
      );
      save(items);
      return JSON.stringify({ result: `Updated: ${id}` });
    }

    if (action === 'delete') {
      const items = load().filter(i => i.id !== input.id);
      save(items);
      return JSON.stringify({ result: `Deleted: ${input.id}` });
    }

    if (action === 'list') {
      const items = load();
      return JSON.stringify({ result: JSON.stringify(items) });
    }

    if (action === 'search') {
      const { query } = input;
      const lower = (query || '').toLowerCase();
      const matches = load().filter(i =>
        Object.values(i).some(v =>
          String(v).toLowerCase().includes(lower)
        )
      );
      return JSON.stringify({ result: JSON.stringify(matches) });
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

**Non-negotiable rules for index.html:**
- `window['ai_edge_gallery_get_result']` — exact key, bracket notation required
- Always `async`
- Always wrap everything in try/catch
- Every code path returns stringified JSON
- localStorage key prefixed with skill name

### Step 5: Write assets/webview.html (js+webview only)

Used for interactive UI. Access shared data via localStorage (same key as index.html uses). Receives messages via `window.addEventListener('message', ...)` if the main script posts updates.

### Step 6: Validate before finishing

- [ ] `name` in SKILL.md frontmatter matches directory name
- [ ] `description` field present and non-empty
- [ ] JS skills: `window['ai_edge_gallery_get_result']` present in index.html
- [ ] JS skills: function is `async`
- [ ] JS skills: try/catch wraps all logic
- [ ] JS skills: all code paths return `JSON.stringify({...})`
- [ ] localStorage key prefixed with skill name (no collision risk)
- [ ] No hardcoded secrets or API keys in files

## Editing Existing Skills

When user asks to modify an existing skill:
1. Read current SKILL.md and scripts/index.html
2. Understand existing structure and actions
3. Apply targeted edits
4. Re-validate checklist after changes

## RAG Pattern (search before write)

When a skill needs to avoid duplicates (e.g., calendar events), instruct the LLM in SKILL.md to:
1. Call skill with `action: "search"` using name/date as query
2. If match found → call with `action: "update"` + found item's `id`
3. If no match → call with `action: "add"`

## Deploying / Installing Skills

### From URL (Android, recommended)
Host on GitHub Pages. User enters folder URL in Gallery app (e.g. `https://user.github.io/repo/skill-name`). Non-google-ai-edge hosts show a disclaimer dialog — user must tap **Agree** to proceed.

### Local import (iOS and Android)
**Skills must be imported as a folder, NOT a zip file.**

Steps:
1. Copy the skill folder (e.g. `calendar-storage/`) to the device
2. iOS: use AirDrop, iCloud Drive, or USB file transfer → Files app
3. In Gallery app: Add skill → Import local skill → select the folder
4. Do NOT zip the folder — the app expects a directory, not an archive

### Secret injection
When `require-secret: true`, the app shows a dialog before the first JS call. The user pastes the secret (API key, JSON, token) there. The skill receives it as the **second parameter**: `async (data, secret) => {}`

## Examples

See `references/examples.md` for complete worked examples of each skill type.
