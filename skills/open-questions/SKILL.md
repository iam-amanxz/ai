---
name: open-questions
description: Resolve the open decisions in a task by asking the user one question at a time, each paired with a clear recommendation, and waiting for their explicit choice before moving on. Use this whenever a request has several unresolved decisions, ambiguities, or branching choices that need the user's input before or while you proceed — underspecified requests, planning something, making a sequence of design or technical calls, or anytime the user says things like "ask me what you need," "walk me through the decisions," "what do you need from me," or "help me figure this out." Reach for it even when the user doesn't literally say "ask me questions" — if you'd otherwise have to guess at several things that materially change the outcome, use this instead of guessing or dumping every question at once.
---

# Open Questions

A protocol for turning a tangle of unresolved decisions into a calm, one-at-a-time conversation. The user answers each question in turn, helped by a concrete recommendation, and ends up with a fully specified task without having to think of everything themselves upfront.

The core idea: **one question per message, each with a recommendation, and you wait for the user's explicit choice before asking the next.** A recommendation is a suggestion to make the choice easy — it is never a license to proceed on your own.

## When this applies

Use this whenever proceeding well would require knowing the answer to more than one open question. The trigger isn't the user asking for questions — it's the presence of genuine decision points where (a) different reasonable answers would meaningfully change what you produce, and (b) the answer isn't already given or clearly inferable from context.

If only one thing is unresolved, just ask it plainly — no need for the full protocol. If nothing is genuinely unresolved, don't manufacture questions; proceed with the task.

## The workflow

### 1. Find the real questions

Privately scan the task for genuine decision points before asking anything. A question earns a spot only if both are true:

- **It matters.** Different reasonable answers would change the output in a way the user would care about.
- **You can't responsibly default it.** The answer isn't stated, isn't inferable from context, and isn't a trivial detail you could sensibly assume and mention.

Resist padding the list with trivia. Five sharp questions respect the user's time; fifteen including obvious ones exhaust it. If you can answer it yourself sensibly, do so and note the assumption rather than asking.

### 2. Order them deliberately

Sequence matters because earlier answers reshape later questions. Order so that:

- **Blocking and foundational questions come first** — the ones that everything else depends on.
- **Questions that constrain later ones come before those later ones**, so you never have to re-ask something differently depending on an answer you haven't gotten yet.

### 3. Ask exactly one at a time

Each message contains a single question and then stops. Never batch, never sneak a "and also..." onto the end. Each question turn should include:

- **The question**, stated plainly.
- **Why it matters** — one short line, only if it isn't obvious. Skip it when the stakes are self-evident.
- **The choices** — 2–4 concrete options when the decision is discrete; an open prompt when it's genuinely open-ended.
- **Your recommendation**, clearly labeled, with a one-line rationale. Pick the option you'd choose and say why in a sentence.

Then wait. Do **not** act on the recommendation. The user chose to make each call explicitly, so hold for their answer even when your recommendation feels obvious. The recommendation removes the burden of generating an answer from scratch; it does not remove the decision.

### 4. Use tappable options when you have them

If an interactive choice tool (e.g. `ask_user_input`) is available, use it for discrete-choice questions — tapping is faster than typing, especially on mobile. Note which option you recommend in the short message that accompanies the choices, since the buttons themselves can't carry a rationale. Fall back to plain text when the question is open-ended or no such tool exists.

### 5. Keep the user oriented

Give a light sense of scale so the user knows how long this will take — e.g. "Question 2 of about 5." Keep the count approximate; new questions may surface and others may evaporate as answers come in. Don't make a ceremony of it.

### 6. Adapt the queue as answers arrive

Each answer can change what's left. If an answer makes a later question moot, drop it. If it raises a new one, slot it in. Never ask something an earlier answer already settled — that signals you weren't listening.

### 7. Recap and proceed

When the open questions are resolved, give a short recap of the decisions made, then carry out the task. The recap lets the user catch a misread before you build on it.

## Honoring an override

If the user says something like "just use your best judgment" or "go with your recommendations," switch modes: proceed on your recommended answers without waiting, briefly noting the calls you made so they can correct any. The default is to wait for explicit choices, but the user can lift that at any time.

## What this looks like

**Good — one question, with a recommendation, then a stop:**

> **Question 1 of ~4.** Should the export be a single combined file or one file per section?
> This affects how downstream tools ingest it.
> - Single combined file
> - One file per section
>
> *My recommendation: single combined file — easier to share and most tools handle it fine. But one-per-section is better if you'll edit sections independently.*

**Bad — batching and proceeding without waiting:**

> Here are my questions: (1) combined or per-section? (2) what format? (3) include metadata? I'll assume combined Markdown with metadata and get started.

The second version dumps everything at once and acts before the user has chosen — exactly what this skill exists to prevent.
