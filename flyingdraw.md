---
name: flyingdraw
description: Push wireframes and diagrams to a live FlyingDraw/Excalidraw canvas. Use when the user says "flyingdraw", "wireframe", "sketch", "draw [something]", "mock [something]", or "show me [something] in flyingdraw".
---

# FlyingDraw (Remote Skill)

**Personal workspace URL:** https://www.flyingdraw.com/YOUR-UUID
(Your own private workspace — replace YOUR-UUID with the UUID from your FlyingDraw browser tab.)

**Team workspace URL:** https://www.flyingdraw.com/YOUR-TEAM-UUID
(Optional — only if your repo shares a team workspace. Replace YOUR-TEAM-UUID with its URL; the skill will then ask each engineer whether to use the team or their personal workspace. Leave the placeholder as-is if you have no team workspace — add `skills/flyingdraw.local.md` to your `.gitignore` so personal overrides aren't committed.)

Before doing anything, fetch the latest skill instructions from GitHub:
- WebFetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/skill.md

After fetching, follow the instructions using the URLs above as `FLYINGDRAW_URL`.
Do not proceed without fetching.

> Tokens are never stored here. When the skill needs auth, it will ask you to paste a token from FlyingDraw directly into the chat (avatar → Get CLI Token → Copy). Tokens live only in conversation context and are never written to any file.
