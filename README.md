# flyingdraw-skills

Claude Code skills for [FlyingDraw](https://flyingdraw.com) — push wireframes from any AI coding session to a live Excalidraw canvas in your browser.

---

## What is FlyingDraw?

FlyingDraw is a local Excalidraw whiteboard that lets Claude Code generate and push wireframes directly to your browser — in real time, no page reload. You sketch with words, Claude draws.

---

## Installing the FlyingDraw skill

### Prerequisites

- [FlyingDraw](https://flyingdraw.com) running locally (`npm run dev` at `localhost:3456`)
- Claude Code in your project

### Setup (one time per project)

**1. Get your workspace URL**

Open FlyingDraw in your browser. Copy the full URL from the address bar — it looks like:

```
http://localhost:3456/b450fda4-9a25-4414-abcd-237b16dfa1df
```

That UUID is your private workspace. Keep it — you'll need it in the next step.

**2. Create `skills/flyingdraw.md` in your project**

```markdown
# FlyingDraw (Remote Skill)

**Workspace URL:** http://localhost:3456/YOUR-UUID-HERE
(Replace with your URL from the FlyingDraw browser tab)

Before doing anything, fetch the latest skill instructions — try local first, fall back to GitHub:
1. WebFetch http://localhost:3456/skills/flyingdraw.md
2. WebFetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md

After fetching, follow the instructions using the Workspace URL above as FLYINGDRAW_URL.
Do not proceed without fetching.
```

**3. Register it in your `CLAUDE.md`**

```markdown
## Skills
- **FlyingDraw** — Push wireframes to the live canvas. Invoke with "flyingdraw …",
  "wireframe …", "sketch …", etc. See `skills/flyingdraw.md`.
```

---

## Usage

Once installed, trigger the skill in any Claude Code session:

```
flyingdraw a login screen
wireframe the onboarding flow
sketch a dashboard with a sidebar
mock the checkout page
```

Claude will:
1. Check FlyingDraw is running
2. Ask which project and board to save it under
3. Generate the Excalidraw wireframe JSON
4. Push it to your canvas via the API
5. Confirm with the board name and URL

---

## How it works

The stub in your project tells Claude where to find your workspace and where to fetch the latest skill instructions. Claude fetches `flyingdraw.md` from your running FlyingDraw server (fast) or from this repo (fallback), then uses your workspace URL for all API calls.

Your workspace URL is the only project-specific config — the skill logic always comes from the latest version here.

---

## Skill reference

The full skill definition is in [`flyingdraw.md`](./flyingdraw.md). It covers:

- Board and project structure
- Step-by-step draw workflow
- Excalidraw JSON element format
- Design principles and colour palette
- All supported triggers
