---
name: play-console-answers
description: "Draft the Google Play Console production-access / closed-testing questionnaire answers (tester recruitment, engagement, feedback, audience, value, changes made, readiness). Use when the user is applying for Play production access, filling the closed-test questions, or asks for app-store review answers under a character limit."
---

# /play-console-answers

Produce Play Console production-access answers grounded in the app's real features, each
under **300 characters**, written in plain first-person prose.

## Steps

1. **Read the app, not the user's memory.** Find the mobile project (`mobile/`, `android/`,
   `app/`, or ask) and list feature/screen files to get the real feature names:
   `find <app>/lib -name "*screen*" -o -name "*page*"` (Flutter) or the equivalent
   routes/activities. Answers must name features that exist.
2. **Ask only for what the code can't tell you** — one short batch, not a form:
   tester count, test length + opt-in dates, how testers were recruited, feedback channels
   used, bugs actually fixed, Android vitals crash-free rate. If the user won't supply them,
   continue with clearly-labelled placeholder values (step 5).
3. **Write the answers** to `<app>/PLAY_CLOSED_TEST_ANSWERS.md`, one `##` heading per
   question with the char count in the heading, e.g. `## 2. Engagement received (285 chars)`.
4. **Count characters yourself** for every answer and keep each ≤300. Prefer a comma or
   semicolon over an em dash — the Play text fields and the user's own style both prefer it.
5. **Never invent facts silently.** Anything not verified goes in a `> **Before submitting:**`
   note at the top of the file listing exactly which numbers and claims to check, and say the
   same in the reply. These answers are factual claims to Google.

## The questions (cover whichever are asked)

- How did you recruit testers? — real channels; say plainly if friends/family or a paid provider.
- Describe tester engagement — which features were used; where usage differed from real users
  (unused paid tier, small data volumes, short window, setup-heavy).
- Summary of feedback + how it was collected — channels, what testers liked, complaints, requests.
- Intended audience — who it's for, and who it is *not* for.
- How the app provides value — the manual thing it replaces, then the core loop.
- Changes made from the test — fixes shipped, then what's queued.
- Why it's ready for production — flow completed end to end, crash/correctness bugs fixed,
  remaining items are additions not blockers.

## Output

The file, then the answers inline in the reply (the user pastes from the terminal), then one
line naming any claim that still needs verifying. No commentary beyond that.
