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

Each user has an isolated workspace identified by a UUID in the URL path.

`FLYINGDRAW_URL` is the **full workspace URL** — it already includes the UUID path:
- e.g. `https://flyingdraw.com/b450fda4-9a25-4414-abcd-237b16dfa1df`

It is set directly in the project's `skills/flyingdraw.md` stub (see the Installing section). All API calls use `$FLYINGDRAW_URL` as the base — no separate workspace ID variable needed.

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

### Step 1 — Resolve the workspace URL and check the server is running

Read `FLYINGDRAW_URL` from the stub file that invoked this skill (or the environment).

```bash
echo "FLYINGDRAW_URL=$FLYINGDRAW_URL"
curl -s "$FLYINGDRAW_URL/api/projects" > /dev/null 2>&1 && echo "running" || echo "not running"
```

- If `FLYINGDRAW_URL` is empty: tell the user:
  > "Your `skills/flyingdraw.md` stub is missing the Workspace URL. Open FlyingDraw in your browser, copy the full URL from the address bar (e.g. `https://flyingdraw.com/b450fda4-…`), and paste it into the stub as shown in the Installing section."
  > Stop here.
- If **not running**: tell the user:
  > "FlyingDraw isn't reachable at $FLYINGDRAW_URL. Check that the server is running and the workspace URL in your stub is correct."
  > Stop here.

### Step 2 — Ask which project and board name to use

Fetch the current project structure:
```bash
curl -s "$FLYINGDRAW_URL/api/projects"
```

This returns a JSON array like:
```json
[
  { "project": "Marketing",     "boards": ["Landing Page", "Pricing Page"] },
  { "project": "Mobile App",    "boards": ["Onboarding", "Dashboard"] },
  { "project": "Uncategorised", "boards": ["Playground"] }
]
```

Present the structure to the user and ask:

> "Which project should this board go under?
>
> **Marketing** — Landing Page, Pricing Page
> **Mobile App** — Onboarding, Dashboard
> **Uncategorised** — Playground
>
> Pick an existing project or type a new name. What should the board be called?"

Wait for both answers before continuing. Store them as `$BOARD_PROJECT` and `$BOARD_NAME`.

- If the user picks an existing board name within a project, you are **updating** that board — preserve its intent.
- If the user gives a new name, you are **creating** a new board in that project.
- If the user skips the project question, default to `Uncategorised`.

### Step 3 — Read the current diagram (optional but recommended)
```bash
curl -s "$FLYINGDRAW_URL/api/diagram"
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
curl -s -X PUT "$FLYINGDRAW_URL/api/diagram" \
  -H "Content-Type: application/json" \
  -d '<JSON with _boardName and _boardProject fields>'
```

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
| `index` | string | Fractional z-order string: `"a1"` < `"a2"` < `"b1"` |
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
Elements render bottom-to-top by index. Use ascending strings:
`"a01"` → `"a02"` → `"a03"` ... background rects first, text on top.

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
      "versionNonce": 1, "index": "a1", "updated": 1700000000000,
      "link": null, "locked": false, "boundElements": null
    },
    {
      "id": "section-label", "type": "text",
      "x": 40, "y": 28, "width": 200, "height": 24,
      "angle": 0, "strokeColor": "#ffffff", "backgroundColor": "transparent",
      "fillStyle": "solid", "strokeWidth": 1, "strokeStyle": "solid",
      "roughness": 0, "opacity": 100, "isDeleted": false,
      "groupIds": [], "frameId": null, "seed": 2, "version": 1,
      "versionNonce": 2, "index": "a2", "updated": 1700000000001,
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

### Setup (one time per project)

1. **Get your workspace URL** — open FlyingDraw in your browser. The full URL in the address bar (e.g. `https://flyingdraw.com/b450fda4-9a25-4414-abcd-237b16dfa1df`) is your workspace URL.

2. **Create `skills/flyingdraw.md`** in your project with this content, replacing the placeholder URL:

```markdown
# FlyingDraw (Remote Skill)

**Workspace URL:** https://flyingdraw.com/YOUR-UUID-HERE
(Replace with your URL from the FlyingDraw browser tab — this is your private workspace)

Before doing anything, fetch the latest skill instructions from GitHub:
- WebFetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md

After fetching, follow the instructions using the Workspace URL above as `FLYINGDRAW_URL`.
Do not proceed without fetching.
```

3. **Reference it in `CLAUDE.md`**:

```markdown
## Skills
- **FlyingDraw** — Push wireframes to the live canvas. Invoke with "flyingdraw …",
  "wireframe …", "sketch …", etc. See `skills/flyingdraw.md`.
```

### How it works

When an AI assistant sees a flyingdraw trigger it:
1. Reads the stub → sees the workspace URL and fetch instruction
2. Fetches the canonical skill from this GitHub repo
3. Uses the workspace URL from the stub as `FLYINGDRAW_URL` for all API calls

The workspace URL is the only project-specific setting. The skill logic always comes from the latest version in this repo.
