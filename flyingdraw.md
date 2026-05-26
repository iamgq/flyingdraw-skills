# FlyingDraw Skill

## Trigger

When the user says any of:
- "flyingdraw [requirement]"
- "draw [requirement] in flyingdraw"
- "wireframe [requirement]"
- "sketch [requirement]"
- "show me [requirement] in flyingdraw"
- "mock [requirement]"
- "give me a mock [requirement]"
- "create a mock [requirement]"
- "show me a mock [requirement]"

---

## Purpose

Push a low-fidelity wireframe diagram to the user's FlyingDraw canvas.
FlyingDraw is an Excalidraw whiteboard — browser live-updates instantly via SSE (no reload needed).

This skill works with any AI coding assistant that can read files and run shell commands — Claude Code, Cursor, Windsurf, Copilot Workspace, etc.

Each user has an isolated workspace identified by a UUID in the URL path.

`FLYINGDRAW_URL` is the **full workspace URL** — it already includes the UUID path:
- e.g. `http://localhost:3456/b450fda4-9a25-4414-abcd-237b16dfa1df`
- e.g. `https://flyingdraw.com/b450fda4-9a25-4414-abcd-237b16dfa1df`

It is set directly in the project's `skills/flyingdraw.md` stub (see the Installing section). All API calls use `$FLYINGDRAW_URL` as the base — no separate workspace ID variable needed.

---

## Authentication

FlyingDraw uses a two-layer authentication model to keep workspaces private and secure:

### Layer 1 — Google Sign-In (browser)
To use FlyingDraw you must first sign in with a Google account. This is a one-time step in the browser and confirms you are a legitimate user. Without signing in, you cannot access any workspace.

### Layer 2 — CLI Token (AI tools)
AI coding tools (Claude Code, Cursor, Windsurf, etc.) cannot do a browser-based Google sign-in. Instead, once you are signed in, you generate a **CLI token** in the browser and paste it directly into your AI chat session. The tool then uses that token to read and write diagrams on your behalf.

Key properties of CLI tokens:
- **Scoped** — each token is tied to one specific workspace
- **Rolling expiry** — the token stays valid as long as it is used; it expires after **30 days of inactivity**
- **Revocable** — you can revoke any token at any time from the FlyingDraw UI
- **Never stored in files** — the token lives only in the chat conversation; it is never written to disk

### How to get a token
1. Open your FlyingDraw workspace in the browser
2. Click your **avatar** (top-right corner)
3. Click **Get CLI Token**
4. Click **Copy** — this copies the raw token value
5. Paste the token directly into your AI chat when prompted

> **First-time use with an AI tool:** When you invoke the skill for the first time (or after a token expires), the tool will ask you to paste a token. Simply follow the steps above, paste the token into the chat, and the tool will continue automatically. The token is never written to any file.

---

## Board Structure

Boards in FlyingDraw are organised into **projects**. Every board belongs to exactly one project.

```
Project
└── Board (a saved diagram)
└── Board
└── Board

Project
└── Board
```

- A **project** is a named group (e.g. `"Marketing"`, `"Mobile App"`, `"Design System"`). Created automatically when first used.
- A **board** is a saved Excalidraw diagram within a project (e.g. `"Onboarding Flow"`, `"Dashboard v2"`).
- Boards without an explicit project land in **`Uncategorised`**.
- The canonical way to reference a board is `Project / Board name` (e.g. `Mobile App / Onboarding Flow`).

When pushing a diagram you must always supply both `_boardProject` and `_boardName`. The API rejects neither — but omitting them means the diagram only updates the live canvas and is not saved to any board.

---

## Steps

### Step 1 — Resolve the workspace URL, token, and check the server is running

Read `FLYINGDRAW_URL` from the stub file that invoked this skill. This is the bare workspace URL — it does **not** contain a token.

**Set up the token for this session:**
- Scan the current conversation for a URL that matches the workspace base and contains `?token=` — the user may have already pasted one.
- If found, extract the token value → set `TOKEN_PARAM=?token=VALUE`.
- If not found, set `TOKEN_PARAM=""` for now; you will request one below if needed.

```bash
echo "FLYINGDRAW_URL=$FLYINGDRAW_URL"
curl -s "${FLYINGDRAW_URL}/api/projects${TOKEN_PARAM}" > /dev/null 2>&1 && echo "running" || echo "not running"
```

- If `FLYINGDRAW_URL` is empty: tell the user:
  > "Your `skills/flyingdraw.md` stub is missing the Workspace URL. Open FlyingDraw in your browser, copy the full URL from the address bar (e.g. `http://localhost:3456/b450fda4-…`), and paste it into the stub as shown in the Installing section."
  > Stop here.
- If **not running**: tell the user:
  > "FlyingDraw isn't reachable at `$FLYINGDRAW_URL`. Check that the server is running and the workspace URL in your stub is correct."
  > Stop here.

**Token request flow** — trigger this whenever any API call returns `401`:
1. Tell the user:
   > "FlyingDraw needs an access token. In your browser, open the workspace, click your avatar (top-right), click **Get CLI Token**, copy the token, and paste it here."
2. When the user pastes a value, extract the token:
   - If it looks like a raw token (hex string, no spaces) → use it directly.
   - If it's a URL containing `?token=VALUE` → extract `VALUE`.
3. Set `TOKEN_PARAM=?token=TOKEN_VALUE` and use it for all remaining API calls.
4. **Never write the token to any file** — it lives only in this conversation.
5. Retry the request that triggered the 401.

### Step 2 — Ask which project and board name to use

Fetch the current project structure:
```bash
curl -s "$FLYINGDRAW_URL/api/projects${TOKEN_PARAM}"
```

This returns a JSON array like:
```json
[
  { "project": "ZR",            "boards": ["Ask ZR Context CTA", "Chat UI"] },
  { "project": "Coach",         "boards": ["Dashboard", "Calendar"] },
  { "project": "Uncategorised", "boards": ["Playground"] }
]
```

Present the structure to the user and ask:

> "Which project should this board go under?
>
> **ZR** — Ask ZR Context CTA, Chat UI
> **Coach** — Dashboard, Calendar
> **Uncategorised** — Playground
>
> Pick an existing project or type a new name. What should the board be called?"

Wait for both answers before continuing. Store them as `$BOARD_PROJECT` and `$BOARD_NAME`.

- If the user picks an existing board name within a project, you are **updating** that board — preserve its intent.
- If the user gives a new name, you are **creating** a new board in that project.
- If the user skips the project question, default to `Uncategorised`.

### Step 3 — Read the current diagram (optional but recommended)
```bash
curl -s "$FLYINGDRAW_URL/api/diagram${TOKEN_PARAM}"
```
- If it has existing elements you should preserve, keep them or ask the user
- If the user said "new diagram" or "fresh", ignore existing content

### Step 4 — Generate the Excalidraw JSON
Design the wireframe based on the user's requirement. Follow the element format reference below.

**Design principles for good wireframes:**
- Use a canvas width of **800px** — elements should fit within x: 0–800
- Stack sections **top to bottom** with clear vertical separation
- Use **roughness: 1** for a sketchy hand-drawn feel
- Use **roughness: 0** for clean precise shapes
- Section backgrounds: wide rectangles (width > 400) behind content elements
- Use **dashed strokeStyle** for placeholder/image areas
- Keep text concise — wireframes show structure, not final copy
- Use **fontFamily: 1** (Virgil hand-drawn) for labels and annotations
- Design **mobile-first** — always note mobile layout in annotations

**Colour palette to use:**
- Primary purple: `#7c3aed`
- Dark navy: `#1a1a2e`
- Light purple bg: `#faf5ff`
- Light purple border: `#c4b5fd`
- Muted purple: `#ede9fe`
- Grey bg: `#f3f4f6`
- Grey border: `#e5e7eb`
- White: `#ffffff`
- Body text: `#4b5563`
- Muted text: `#9ca3af`

### Step 5 — Push via API
Send the diagram via the API — **never write files directly**:

```bash
curl -s -X PUT "${FLYINGDRAW_URL}/api/diagram${TOKEN_PARAM}" \
  -H "Content-Type: application/json" \
  -d '<JSON with _boardName and _boardProject fields>'
```

If the response is `{"error":"token_expired"}` or `{"error":"unauthorized"}`:
- Follow the **Token request flow** from Step 1 — ask the user to paste a fresh token and retry.

Always include both `_boardName` and `_boardProject` in the top-level JSON:

```json
{
  "_boardName": "Architecture",
  "_boardProject": "FlyingDraw",
  "type": "excalidraw",
  "version": 2,
  "source": "flyingdraw",
  "elements": [ ...elements... ],
  "appState": { "gridSize": null, "viewBackgroundColor": "#f8f9fa" },
  "files": {}
}
```

- `_boardName`: the board name (ask the user if not stated — default to a short descriptive name)
- `_boardProject`: the project from Step 2 (always set this — use `Uncategorised` if the user skipped)
- Without both fields: updates canvas only, board panel won't reflect the save

### Step 6 — Confirm
Tell the user:
> "Wireframe pushed to **[project] / [board]** — check $FLYINGDRAW_URL ✓"
> Briefly describe what sections/elements you drew (1–2 lines).

---

## Excalidraw JSON Format

### Top-level structure
```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "flyingdraw",
  "elements": [ ...elements... ],
  "appState": { "gridSize": null, "viewBackgroundColor": "#f8f9fa" },
  "files": {}
}
```

### Required fields for EVERY element
| Field | Type | Notes |
|-------|------|-------|
| `id` | string | Unique, descriptive (e.g. `"hero-bg"`, `"nav-cta"`) |
| `type` | string | `rectangle` `ellipse` `text` `arrow` `line` `diamond` |
| `x` | number | Canvas X position |
| `y` | number | Canvas Y position |
| `width` | number | Element width |
| `height` | number | Element height |
| `angle` | number | Always `0` unless rotated |
| `strokeColor` | string | Hex colour |
| `backgroundColor` | string | Hex colour or `"transparent"` |
| `fillStyle` | string | `"solid"` `"hachure"` `"cross-hatch"` |
| `strokeWidth` | number | `1`, `2`, or `4` |
| `strokeStyle` | string | `"solid"` `"dashed"` `"dotted"` |
| `roughness` | number | `0` = clean, `1` = sketchy, `2` = very rough |
| `opacity` | number | `0`–`100` |
| `isDeleted` | boolean | Always `false` |
| `groupIds` | array | `[]` if ungrouped |
| `frameId` | null | Always `null` |
| `seed` | number | Any integer (controls rough.js randomness) |
| `version` | number | `1` |
| `versionNonce` | number | Any integer |
| `index` | string | Fractional z-order string — must be 3+ chars, e.g. `"a01"` `"a02"` `"a0a"` |
| `updated` | number | Unix ms timestamp e.g. `1700000000000` |
| `link` | null | Always `null` |
| `locked` | boolean | `false` |
| `boundElements` | null | `null` or `[]` |

### Text elements — additional required fields
```json
{
  "type": "text",
  "text": "Your label here",
  "fontSize": 16,
  "fontFamily": 1,
  "textAlign": "left",
  "verticalAlign": "top",
  "containerId": null,
  "originalText": "Your label here",
  "autoResize": true,
  "lineHeight": 1.25
}
```
- `fontFamily`: `1` = Virgil (hand-drawn), `2` = Helvetica, `3` = Cascadia (mono)
- `fontSize`: `12` (caption) / `13`–`14` (body) / `16`–`18` (subhead) / `22`–`28` (headline) / `32`–`40` (hero)

### Arrow elements — additional required fields
```json
{
  "type": "arrow",
  "points": [[0, 0], [120, 0]],
  "lastCommittedPoint": null,
  "startBinding": null,
  "endBinding": null,
  "startArrowhead": null,
  "endArrowhead": "arrow"
}
```

### Z-order (index field)
Elements render bottom-to-top by index. Use ascending 3-character base-36 strings:
`"a01"` → `"a02"` → `"a03"` → ... → `"a09"` → `"a0a"` → `"a0b"` → ... background rects first, text on top.

**Critical:** Index values must be exactly 3+ characters in base-36 format (letter + two alphanumeric chars using `0-9a-z`). Never use 2-character values like `"a1"` or `"c7"` — these cause "invalid order key" errors that break drawing on the board. For up to 36 elements stay in the `a0?` range (`a01`–`a0z`); for more continue `a10`, `a11`, ... `a1z`, `a20`, etc.

---

## Minimal working example

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "flyingdraw",
  "elements": [
    {
      "id": "section-bg", "type": "rectangle",
      "x": 0, "y": 0, "width": 800, "height": 80,
      "angle": 0, "strokeColor": "#1a1a2e", "backgroundColor": "#1a1a2e",
      "fillStyle": "solid", "strokeWidth": 1, "strokeStyle": "solid",
      "roughness": 0, "opacity": 100, "isDeleted": false,
      "groupIds": [], "frameId": null, "seed": 1, "version": 1,
      "versionNonce": 1, "index": "a01", "updated": 1700000000000,
      "link": null, "locked": false, "boundElements": null
    },
    {
      "id": "section-label", "type": "text",
      "x": 40, "y": 28, "width": 200, "height": 24,
      "angle": 0, "strokeColor": "#ffffff", "backgroundColor": "transparent",
      "fillStyle": "solid", "strokeWidth": 1, "strokeStyle": "solid",
      "roughness": 0, "opacity": 100, "isDeleted": false,
      "groupIds": [], "frameId": null, "seed": 2, "version": 1,
      "versionNonce": 2, "index": "a02", "updated": 1700000000001,
      "link": null, "locked": false, "boundElements": null,
      "text": "Nav Bar", "fontSize": 18, "fontFamily": 1,
      "textAlign": "left", "verticalAlign": "top",
      "containerId": null, "originalText": "Nav Bar",
      "autoResize": true, "lineHeight": 1.25
    }
  ],
  "appState": { "gridSize": null, "viewBackgroundColor": "#f8f9fa" },
  "files": {}
}
```

---

## Rules

- **Always write the complete JSON** — never partial updates (Excalidraw replaces the whole file)
- **Never invent data** — if you don't know the user's content, use placeholder text like `"[ Hero Image ]"` or `"Section Title"`
- **Mobile-first** — canvas is 800px wide (desktop view); always add an annotation note describing mobile layout changes
- **Keep it low-fi** — this is a wireframe, not a finished design. Rough shapes and placeholder text are correct.
- **Preserve user's existing drawings** unless they explicitly ask for a new/fresh diagram
- If the user annotates or edits the diagram, read the file back before making changes

---

## Installing this skill in another project (Remote Skill)

### Prerequisites

All you need is a **Google account**. FlyingDraw is hosted at [flyingdraw.com](https://flyingdraw.com) — no self-hosting required.

**Getting your workspace URL:**

- **New user:** Open [flyingdraw.com](https://flyingdraw.com) and sign in with Google. A private workspace is created for you automatically. Your workspace URL appears in the address bar and looks like `https://www.flyingdraw.com/b450fda4-9a25-4414-abcd-237b16dfa1df`. Once signed in you can create additional workspaces from the workspace panel.
- **Joining a team workspace:** Someone with access (e.g. a team member) shares the workspace URL with you. Open it and sign in with Google to join.

Workspaces support real-time collaboration — multiple people can view and edit the same canvas simultaneously.

### Setup (one time per project)

1. **Sign in and copy your workspace URL** — open [flyingdraw.com](https://flyingdraw.com), sign in with Google, and copy the full URL from the address bar. It will look like `https://www.flyingdraw.com/b450fda4-9a25-4414-abcd-237b16dfa1df`.

2. **Create `skills/flyingdraw.md`** in your project — download the ready-made stub:

   ```bash
   curl -o skills/flyingdraw.md https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/stub.md
   ```

   Then open the file and replace `https://www.flyingdraw.com/YOUR-UUID` with your actual workspace URL from step 1.

   Or create it manually:

```markdown
# FlyingDraw (Remote Skill)

**Workspace URL:** https://www.flyingdraw.com/YOUR-UUID
(Replace YOUR-UUID with your workspace UUID from the FlyingDraw browser tab.)

Before doing anything, fetch the latest skill instructions — try local first, fall back to GitHub:
1. WebFetch http://localhost:3456/skills/flyingdraw.md
2. WebFetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md

After fetching, follow the instructions using the Workspace URL above as `FLYINGDRAW_URL`.
Do not proceed without fetching.
```

3. **Reference it in your project's AI instructions file** (`CLAUDE.md`, `.cursorrules`, `AGENTS.md`, etc.):

```markdown
## Skills
- **FlyingDraw** — Push wireframes to the live canvas. Invoke with "flyingdraw …",
  "wireframe …", "sketch …", etc. See `skills/flyingdraw.md`.
```

4. **Connect your AI tool with a token** — your AI tool needs a token to access your workspace. To get one:
   1. Open [flyingdraw.com](https://flyingdraw.com) in your browser
   2. Click your **avatar** (top-right) → **Get CLI Token** → **Copy**
   3. The first time you invoke FlyingDraw from your AI tool (e.g. Claude Code, Cursor), it will prompt you for a token — simply paste it into the chat and the tool will continue automatically

   > Tokens are never stored in any file. They live only in the chat session. If a token expires (after 30 days of inactivity), repeat the steps above to get a fresh one.

### How it works

When an AI assistant sees a flyingdraw trigger it:
1. Reads the stub → sees the workspace URL and fetch instruction
2. Fetches the canonical skill from localhost (fast) or GitHub (fallback)
3. Checks the current conversation for a previously pasted token
4. If no token is found and the API returns 401, asks you to paste one from FlyingDraw (avatar → Get CLI Token)
5. Uses `FLYINGDRAW_URL` + `?token=TOKEN` for all API calls — token stays in chat, never in files

The workspace URL is the only project-specific setting. Works with Claude Code, Cursor, Windsurf, Copilot Workspace, and any other AI assistant that can read files and run shell commands.

> **Requires:** The `flyingdraw-skills` GitHub repo must be public so `raw.githubusercontent.com` URLs resolve without authentication.
