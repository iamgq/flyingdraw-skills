---
name: flyingdraw
description: Push wireframes and diagrams to a live FlyingDraw/Excalidraw canvas. Use when the user says "flyingdraw", "wireframe", "sketch", "draw [something]", "mock [something]", or "show me [something] in flyingdraw".
---

# FlyingDraw (Remote Skill)

**Workspace URL:** https://www.flyingdraw.com/YOUR-UUID
(Replace YOUR-UUID with your workspace UUID from the FlyingDraw browser tab.)

Before doing anything, fetch the latest skill instructions from GitHub:
- WebFetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/skill.md

After fetching, follow the instructions using the Workspace URL above as `FLYINGDRAW_URL`.
Do not proceed without fetching.

> Tokens are never stored here. When the skill needs auth, it will ask you to paste a token from FlyingDraw directly into the chat (avatar → Get CLI Token → Copy). Tokens live only in conversation context and are never written to any file.
