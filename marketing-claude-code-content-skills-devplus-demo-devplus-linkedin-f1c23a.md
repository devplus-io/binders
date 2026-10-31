---
id: f1aa37e7-62c8-4fb6-9676-833371f1c23a
binder: binder-mvvle32a-1
---
Marketing | Claude Code content skills (devplus-demo, devplus-linkedin)

Source: Claude Code session, 2026-10-05 (~/.claude/skills/devplus-demo, ~/.claude/skills/devplus-linkedin on Helder's Mac)

Updated 2026-10-05 against the skills on Helder's Mac (not in a repo).

The DevPlus LinkedIn and YouTube demo pipeline now also runs from Claude Code, as two user-level skills on Helder's Mac. They are rebuilt versions of the Cove/Pennant skills `devplus-linkedin-carousel` and `devplus-product-demo` (cove-skills repo), which keep running in Pennant unchanged.

**/devplus-demo** (product demo video)
- Recorded only on the showroom mock site (http://127.0.0.1:3100, synthetic data), narrated in Helder's cloned voice (mlx-audio Qwen3-TTS on the Mac), edited with Remotion.
- Two cuts from one recording: YouTube 16:9 (`demo.mp4`) and a LinkedIn 4:5 feed cut with burned-in captions (`demo-portrait.mp4`), plus `captions.srt`, a thumbnail cropped to the product detail, and YouTube chapters built from the scene timings.
- Opens on the payoff (cold open); the tour frames what each scene talks about with `focus()` so the UI is readable; review.py checks length, loudness, hook, pace, soundtrack vs script and camera legibility.
- Uploads to YouTube as Unlisted only after Helder approves the exact files in the chat.
- Studio: `~/Documents/DevPlus Studio/studio-cc` (Pennant keeps `studio/`); videos in `~/Documents/DevPlus Studio/videos/`.

**/devplus-linkedin** (company-page post)
- One capability per post, rotated; facts only from devplus.io, the DevPlus binder or the code at origin/main.
- Default visual: a 4–7 slide carousel posted as a PDF document (the swipeable LinkedIn format), from a new design kit with five palettes and a legibility audit (nothing under 22 px, overflow, overlap, contrast, word count). Single posters and demo video cuts are also supported.
- lint_post.py checks house style (no em dashes, banned phrases, hook length, one devplus.io link, 0–3 hashtags, no compliance claims, unsourced numbers); novelty.py + frame_check.py stop repeats in wording and look.
- Posts to linkedin.com/company/devplus-io only after Helder approves the exact text and media in the chat.

**Publishing safety:** approval is recorded as sha256 fingerprints of the approved files (approve.py); the YouTube and LinkedIn publishers refuse anything missing or changed since. Both use a dedicated Chrome profile (`~/Documents/DevPlus Studio/browser-profile`) that Helder signs into himself.

**Not public yet (don't post about without Helder's go-ahead):** GCP connectors/discovery, drift detection, brownfield adoption, GitHub repository analysis. They are in the product but not on devplus.io.
