---
id: 9691e6d0-c5dd-44f7-ad4e-281903a106e3
binder: binder-mvvle32a-1
---
Social post · 2026-10-27-failure-handoff

## Status
REJECTED BY FOUNDER. Approval card 66E643B9-E30C-4D33-9C7D-2CC5E52D3731 was explicitly rejected in the conversation after the 27 October submission. No reason was recorded. Not published from this card. Retired: do not revive, silently revise or resubmit this story. Counts in creative rotation, not publication or engagement cohorts. This disposition supersedes the readiness and approval language below. Reconciled 28 October 2026.

## sources
Capability `runs` from the skill's capabilities.md: every run in its own isolated, single-use environment, logs live, exactly the reviewed plan applied. Plus these reads of the read-only mirror (~/Library/Application Support/Pennant/knowledge/devplus-repos), done 27 October 2026:
- devplus-executor services/executor_service.go:823-839: the deployment log callback prefixes each line with time-of-day (time.Now().Format("15:04:05")), appends it to the deployment and broadcasts it on the WebSocket notifier.
- devplus-backend services/deployment_service.go:893: AppendLogs pushes lines onto the deployment document in MongoDB.
- devplus-frontend app/projects/[projectId]/deployments/[deploymentId]/logs/page.tsx:214-220 seeds the stored lines; :271-287 opens the live stream and appends log frames.
- devplus-backend handlers/deployment_handler.go:896: on redeploy the handler sets existingDeployment.Logs = []string{}. So the post says review the current output before redeploying, and does not claim permanent history.
Deliberately not claimed: no customer, no number, no benchmark, no diagnosis by the product, no permanent per-run archive.

## concepts
A point-of-view / selected. Hook: the failure is at the bottom of the log, the clue may be a few lines above. Cards: the sequence a failed run leaves, the same run as timestamped output, and what to do before a redeploy clears it.
B how-it-works / held. Follow the output back to the first failed step.
C risk-story / held. Redeploying clears the output before anyone reads it.

## plan
capability runs · format point-of-view · theme failed-run-handoff · palette p-night · no news hook · carousel: three cards so the reader sees the sequence, the recorded lines, and the redeploy boundary, which one card cannot carry.
Rotation checked against the 18 post plans on disk (Sep 23 to Oct 18). runs was last published 4 October, 23 days back, and is not in the last three (architect 18 Oct rejected, ttl 14 Oct, cost 12 Oct rejected, agents-question 7 Oct rejected). Format point-of-view is not in the last two. Theme failed-run-handoff is new.

## draft-1
A failed deployment needs more than the last error line.
DevPlus streams the deployment's logs as it runs and stores timestamped lines on its record. Open the logs to follow the sequence leading up to a failure, rather than handing someone a pasted error without the context.
The final error tells you where it stopped. The lines above tell you why.
See it: https://devplus.io/demo

## draft-2 · humanizer pass
The failure is at the bottom of the log. The clue may be a few lines above it.
DevPlus streams deployment logs as the work runs, with timestamped lines stored on the deployment record. Open that output to follow the sequence leading up to a failure before handing it to the next person.
Review the current output before you redeploy. Redeploying clears those logs, so this is a place to investigate the current attempt, not a permanent history of every run.
Book a walkthrough: https://devplus.io/demo

## draft-3 · busy-founder and sceptical-engineer pass
The engineer pass cut the slogan line and added the boundary sentence, because the redeploy reset makes any stronger claim wrong. The founder pass replaced "See it" with the verified link text.

## post
The failure is at the bottom of the log. The clue may be a few lines above it.

DevPlus streams deployment logs as the work runs, with timestamped lines stored on the deployment record. Open that output to follow the sequence leading up to a failure before handing it to the next person.

Review the current output before you redeploy. Redeploying clears those logs, so this is a place to investigate the current attempt, not a permanent history of every run. The output tells you what the run reported; it still needs someone to interpret it.

Book a walkthrough: https://devplus.io/demo

## carousel
palette p-night, compose lede on card 1, cards diagram / record / steps. Headlines: "A failed run leaves {a sequence}, not one error line." / "The output is timestamped, {so the order survives}." / "Redeploying clears the output. {Read it first.}" Card 2 rows are illustrative and stamped "Example data". Kicker "DevPlus · How it works" on all three.

## checks
- ai_writing_check: no findings. No em or en dashes.
- novelty.py against the 18 creative folders only (analytics and measurement folders excluded): ok, no findings. The first run flagged three light canvases in a row, so the palette moved from p-mono to p-night and the dark frame passed.
- Render: render.mjs ok, three PNGs at 1080x1350, each opened and checked for overflow, cut text and phone-size legibility. p-electric was tried and dropped: frame_check measured it as light canvas.
- Claims: every DevPlus statement traces to capabilities.md or the mirror reads above. Card 2's timestamps are labelled example data.
- Not verified: no live failed deployment was run for this post. The statements describe the shipped code, and the card says what the record holds, not what a particular run did.
