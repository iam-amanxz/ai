---
name: roundtable
description: Critique a plan through a critical round-table of expert personas, then arrive at the best version of the plan. Picks the right specialist squad for THIS plan (not a fixed panel), runs each seat as its own subagent, lets them clash, puts unresolved decisions to the user, and a dedicated synthesis agent writes the final plan. Use when the user wants a plan stress-tested by experts, a design review panel, "roundtable", "expert critique", or "poke holes in this plan".
---

# /roundtable

Convene a critical expert round-table on a plan, surface its real defects through disagreement, and produce the best version of the plan. **Not applause — critique.** Every seat's job is to find what breaks, what's missing, or what's over-built.

**Each seat runs as its own subagent.** The chair (you) selects the squad and synthesises, but every expert critique is produced by a *separate* agent that has never seen the other seats' takes — that independence is the whole point: it kills the groupthink you'd get from one model role-playing five voices in one pass.

## Step 0 — get the plan

Use the plan the user names or pastes. If none: the most recently approved/active plan in context, memory, or the repo. If still ambiguous, ask which plan — one line, then proceed.

Read the plan **in full** (and the files it touches) before seating anyone. A critique of a plan you haven't traced is theatre.

## Step 1 — pick the squad (this is the point — do NOT seat everyone)

There is a candidate roster (below). **Pick only the 3–5 seats that catch the defects THIS plan is most likely to ship with**, and **cut the rest with a one-line reason each**. A backend schema remodel does not need a UX seat; a portal feature does not need a DBA arguing migration order. Wrong seats dilute the signal.

Selection heuristic: for each seat, ask *"what class of defect does this seat catch, and can this plan plausibly contain it?"* No → cut it, say why.

Then add any **project-specific seat** the repo's own rules demand (e.g. a governance/compliance seat where a spec or regulator governs outcomes; a multi-tenancy seat where the platform must stay tenant-agnostic). Honour the repo's CLAUDE.md — its hard rules are non-negotiable constraints the panel enforces, not opinions it debates.

### Candidate roster (archetypes — adapt titles to the domain)

- **Domain SME** — is the model *correct*? Catches invented rules, wrong entities, missing states. Seat for anything domain-shaped.
- **Data modeller / DBA** — schema shape, atomicity, invariants, migration safety, reversibility. Seat when a schema or data model changes.
- **System architect** — boundaries, coupling, sync-vs-event, dependency direction. Seat when a plan crosses services/modules or adds a boundary.
- **Senior developer** — buildability and cost: does the diff wire up, what's the real blast radius, what did the plan hand-wave. Seat almost always.
- **Security / privacy engineer** — trust boundaries, PII, access control, audit. Seat when data is sensitive or access rules change.
- **Frontend / UX + i18n** — the user surface, accessibility, RTL/localisation. Seat only when a UI ships.
- **Integration engineer** — external systems, gateways, contracts, versioning. Seat when the plan crosses a wire.
- **Compliance / governance** — regulatory figures, spec fidelity, supersession/ADR discipline, "does this change an *outcome* someone else owns". Seat where a governing canon or regulator exists.
- **Multi-tenancy / config purist** — "is this hardcoded for one tenant? who edits it later — an admin via config, or a dev via redeploy?" Seat on any platform meant for more than one customer.
- **The lazy skeptic (YAGNI)** — "does this need to exist at all? what's already built we're re-inventing?" The chair holds this seat by default; name it explicitly only if the plan reeks of over-building.

## Step 2 — run the round table

### 2a. Opening critiques — one agent per seat, in parallel

Spawn **one subagent per selected seat**, all in a **single message with multiple Agent tool calls** so they run concurrently. Use the `general-purpose` agent type (or a project-defined critique/review agent if one exists) — the seat must be able to *read and reason over* the plan and the files it touches, so a read-only search agent is the wrong tool.

**Each seat does a thorough, deep analysis — not a skim.** The mandate is an investigation, not a first-impression react: the agent reads the plan *in full*, **traces every file, table, contract and call-path the plan touches** in its lane, and checks the plan's claims against what the code actually does before it objects. A surface-level "looks risky" is worthless; a defect named at a specific `file:line` / field / step, with the failure it causes traced through, is the whole deliverable. Tell the agent explicitly to dig until it either finds the real defects or can defend that the lane is sound.

Each seat's prompt must be **self-contained** — the agent does not share the chair's context:

- **The plan** — paste it in full, or give the exact file path(s) and tell the agent to read it in full first.
- **The files it touches** — the paths the agent should trace before judging. Instruct it to go deep: open them, follow the call-paths, verify the plan's assumptions against the real code — a critique of an un-traced plan is theatre.
- **The persona + mandate** — "You are the {seat}. Do a deep, thorough analysis of THIS plan in your domain and find what it ships broken, missing, or over-built. Adversarial, not agreeable. Trace the real code before you judge. Critique the plan, not the author."
- **The class of defect to hunt** — the one-line "what this seat catches" from the roster.
- **Repo hard rules** — any CLAUDE.md constraints in the seat's lane are non-negotiable, enforced not debated.
- **A structured return contract**, so the chair and the synthesis agent can compare them cleanly. Ask each agent to return its sharpest **1–5 objections** (depth over a fixed count), each as: `location` (file:line / field / step named), `defect` (what breaks), `why` (the failure it causes, traced), `evidence` (what in the code/plan confirms it), `fix` (the smallest change that resolves it), `severity` (blocker / major / minor). Plus a one-line verdict: does the plan's core hold, or not. **No praise padding.** A seat with nothing real to say returns "no material objection" — but only after a genuine deep pass, not as a shortcut.

### 2b. Cross-fire — chair-run, re-engage seats only where tension is real

You (chair) collect the agents' returns and stage the genuine clashes: seat A demands X, seat B says X breaks Y — who wins and why. This is where the best version is forged; don't smooth over a real tension, pick a side with a reason.

Where a clash is genuinely unresolved and the resolution changes the plan, **re-engage the two clashing seats for one focused rebuttal round** — continue each agent with `SendMessage` (its context is intact), handing it the opposing seat's objection and asking it to concede or defend in ≤3 lines. Don't run a rebuttal round for tensions you can resolve yourself — that's filler.

### 2c. Open-questions gate — resolve with the user BEFORE synthesis

Collect every decision the panel could **not** settle without the user — a clash cross-fire couldn't resolve, a missing domain fact, a fork where two seats each have a defensible answer. **Do not guess these, and do not let the synthesis agent guess them.**

Run the **`open-questions` skill** to put them to the user — one question at a time, each paired with a clear recommendation, waiting for an explicit choice. The user's answers become binding inputs to synthesis. If the user defers or skips one, carry it as an explicit residual open item (the synthesis agent must not silently resolve it).

**Synthesis does not start until this gate is answered** (or the user explicitly says "proceed with your recommendations").

### 2d. Synthesis agent — a dedicated seat writes the final plan

Spawn **one final `general-purpose` subagent — the synthesis/editor seat** — to produce the revised plan. It is *not* one of the critic seats; its only job is to reconcile. Its self-contained prompt carries:

- the **original plan** (full text or path);
- **every seat's structured critique** (from 2a) and the outcome of each cross-fire clash (from 2b) — which objections won, which were rejected and why;
- the **user's answers** from the open-questions gate (2c), as binding constraints;
- the **repo hard rules** (CLAUDE.md) it must honour;
- a mandate: *fold the surviving critiques into the best, buildable version of the plan — concrete, not a diff of opinions; every change traceable to the objection or user answer that forced it; do not re-open settled questions or invent facts.*

Ask it to return **(1)** the revised plan and **(2)** a "what changed and why" list (one line per change, citing the seat or user answer that forced it).

The chair's remaining job is routing and quality control: if the synthesis agent papered over a surviving objection, ignored a user answer, or invented a fact, hand it back for one revision. Seat and synthesis outputs are internal working material — surface the *substance*, never dump a raw agent transcript.

## Step 3 — present

- **The best version of the plan** — the synthesis agent's revised plan, concrete and buildable.
- **What changed and why** — one line per change, each citing the seat or the user answer that forced it.
- **Residual open items** — only questions the user deferred/skipped at the 2c gate. Extremely concise; sacrifice grammar for concision. (If the gate was fully answered, this is empty — the open questions were resolved *before* the plan, not left dangling after it.)

## Rules

- Critique the plan, not the author. The goal is a better plan, not a longer meeting.
- A seat that only agrees gets cut — dissent is the whole value.
- **The squad you cut in Step 1 is agents you never spawn.** Each seat is a real subagent with a real cost — 3–5 well-chosen seats, spawned together in one message, never "seat everyone to be safe".
- The chair does **not** get its own critique agent — you hold squad-selection, cross-fire, the open-questions gate, and quality-control of the synthesis. The critic seats and the final synthesis are the agents; you route and adjudicate.
- **Order is fixed: critiques → cross-fire → open-questions gate (user answers) → synthesis agent → present.** The final plan is never produced before the user has answered the open questions.
- Don't invent domain facts to win an argument — this binds the critic seats AND the synthesis agent. Missing fact or clashing instruction → it goes to the user through the open-questions gate, never papered over.
- Length scales with the plan. A small plan gets a tight three-seat exchange, not a symposium.
