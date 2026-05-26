# flyingdraw-skills

Claude Code skills for [FlyingDraw](https://flyingdraw.com) — push wireframes from any AI coding session to a live Excalidraw canvas in your browser.

Works with **Claude Code**, **Cursor**, **Windsurf**, **Gemini CLI**, **Codex CLI**, and **GitHub Copilot**.

---

## What is FlyingDraw?

FlyingDraw is a collaborative Excalidraw whiteboard that lets your AI coding assistant generate and push wireframes directly to your browser — in real time, no page reload. You sketch with words, the AI draws.

---

## Prerequisites

- Access to a hosted FlyingDraw instance and your workspace URL (ask your team admin)
- Your workspace URL looks like: `https://flyingdraw.com/b450fda4-9a25-4414-abcd-237b16dfa1df`

---

## Setup by tool

### Claude Code

**1. Create `skills/flyingdraw.md` in your project:**

```markdown
# FlyingDraw (Remote Skill)

**Workspace URL:** https://flyingdraw.com/YOUR-UUID-HERE

Before doing anything, fetch the latest skill instructions from GitHub:
- WebFetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md

After fetching, follow the instructions using the Workspace URL above as FLYINGDRAW_URL.
Do not proceed without fetching.
```

**2. Register it in `CLAUDE.md`:**

```markdown
## Skills
- **FlyingDraw** — Push wireframes to the live canvas. Invoke with "flyingdraw …",
  "wireframe …", "sketch …", etc. See `skills/flyingdraw.md`.
```

---

### Gemini CLI

**1. Create `skills/flyingdraw.md` in your project** (same stub as Claude Code above).

**2. Register it in `GEMINI.md`:**

```markdown
## Skills
- **FlyingDraw** — Push wireframes to the live canvas. Invoke with "flyingdraw …",
  "wireframe …", "sketch …", etc. See `skills/flyingdraw.md`.
```

---

### Codex CLI

**1. Create `skills/flyingdraw.md` in your project** (same stub as Claude Code above).

**2. Register it in `AGENTS.md`:**

```markdown
## Skills
- **FlyingDraw** — Push wireframes to the live canvas. Invoke with "flyingdraw …",
  "wireframe …", "sketch …", etc. See `skills/flyingdraw.md`.
```

---

### Cursor

**1. Create `.cursor/rules/flyingdraw.mdc` in your project:**

```markdown
---
description: Push wireframes to FlyingDraw canvas. Triggers on "flyingdraw", "wireframe", "sketch", "mock".
alwaysApply: false
---

# FlyingDraw

**Workspace URL:** https://flyingdraw.com/YOUR-UUID-HERE

When triggered, fetch the latest skill instructions from GitHub:
- Fetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md

Then follow the instructions using the Workspace URL above as FLYINGDRAW_URL.
Do not proceed without fetching.
```

---

### Windsurf

**1. Add to `.windsurfrules`** in your project root (create if it doesn't exist):

```markdown
## FlyingDraw

**Workspace URL:** https://flyingdraw.com/YOUR-UUID-HERE

When the user says "flyingdraw", "wireframe", "sketch", or "mock":
1. Fetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md
2. Follow the instructions using the Workspace URL above as FLYINGDRAW_URL.
Do not proceed without fetching.
```

---

### GitHub Copilot

**1. Add to `.github/copilot-instructions.md`** in your project (create if it doesn't exist):

```markdown
## FlyingDraw

**Workspace URL:** https://flyingdraw.com/YOUR-UUID-HERE

When the user says "flyingdraw", "wireframe", "sketch", or "mock":
1. Fetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md
2. Follow the instructions using the Workspace URL above as FLYINGDRAW_URL.
Do not proceed without fetching.
```

---

## Usage

Once installed, trigger the skill in any session:

```
flyingdraw a login screen
wireframe the onboarding flow
sketch a dashboard with a sidebar
mock the checkout page
```

The AI will:
1. Check FlyingDraw is reachable
2. Ask which project and board to save it under
3. Generate the Excalidraw wireframe JSON
4. Push it to your canvas via the API
5. Confirm with the board name and URL

---

## How it works

The stub in your project tells the AI your workspace URL and where to fetch the latest skill instructions. The AI fetches `flyingdraw.md` from this repo, then uses your workspace URL for all API calls.

Your workspace URL is the only project-specific config — the skill logic always comes from the latest version here.

---

## Skill reference

The full skill definition is in [`flyingdraw.md`](./flyingdraw.md). It covers:

- Board and project structure
- Step-by-step draw workflow
- Excalidraw JSON element format
- Design principles and colour palette
- All supported triggers
