# FlyingDraw (Remote Skill)

**Workspace URL:** https://YOUR-FLYINGDRAW-URL/YOUR-UUID
(Replace with your workspace URL from the FlyingDraw browser tab — this is your private workspace)

Before doing anything, fetch the latest skill instructions — try local first, fall back to GitHub:
1. WebFetch http://localhost:3456/skills/flyingdraw.md
2. WebFetch https://raw.githubusercontent.com/iamgq/flyingdraw-skills/main/flyingdraw.md

After fetching, follow the instructions using the Workspace URL above as `FLYINGDRAW_URL`.
Do not proceed without fetching.

> Tokens are never stored here. When the skill needs auth, it will ask you to paste a token from FlyingDraw directly into the chat. Tokens live only in conversation context.
