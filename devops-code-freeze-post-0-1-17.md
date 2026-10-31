---
id: 4c4c7e2a-14c6-4276-bf89-1f2ef0c12e76
binder: binder-mvvle32a-1
---
DevOps | code freeze (post-0.1.17)

Source: Helder's "To Do:" note and card "Finish 0.1.7 release" (AEE99E06); umbrella CLAUDE.md

Updated 2026-10-26 against origin/main (umbrella CLAUDE.md, devplus-platform CHANGELOG.md).

# Code freeze after release 0.1.17

**In force since 2026-10-24, until Helder lifts it.** Approved by Helder with the 0.1.17 release plan (card "Finish 0.1.7 release", read as 0.1.17).

## What may merge to `main` in any DevPlus repo
- Bug fixes.
- Performance work.
- Security and dependency fixes (audit bumps, advisories).
- Docs and runbook updates.

## What waits
- New features. They stay on their branch until the freeze lifts. Example: project images (branch `feat/project-images` in devplus-shared, devplus-backend and devplus-frontend, tabled 2026-10-22).

## How it is enforced
- By convention, not by a GitHub ruleset. The devplus-github-settings ruleset has drifted from its code (README "Live drift"), so an apply to add a freeze could undo live settings.
- The umbrella CLAUDE.md carries a "Code freeze" line at the top, so every Claude Code session in the workspace follows it: a feature request gets "frozen, ask first" before any PR.

## Fixes merged under the freeze so far
- devplus-agent-service #60, devplus-frontend #131 (audit), devplus-core-infra #102 (CI runners exempt from the /api/v1 rate limit), devplus-backend #86 (support cursor 400), devplus-platform #83 (CHANGELOG). See "DevOps | release 0.1.17 status".

## Lifting it
Helder says so. Then remove the line from the umbrella CLAUDE.md and mark this entry lifted with the date.
