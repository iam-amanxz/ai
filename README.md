# ai

Agent skills I use daily to drive coding agents.

A skill is a markdown file that teaches an agent one job properly — when to reach for
it, what to produce, and what it is not allowed to do. These are the ten I keep
installed. Most exist because an agent's *default* behaviour on that task was wrong in
a specific, repeatable way, and the skill is the correction.

## The skills

| Skill | What it does |
|---|---|
| [`roundtable`](skills/roundtable/SKILL.md) | Convenes an expert panel on a plan. Picks the squad for *this* plan, runs **each seat as its own subagent** so no critic sees another's take, puts the unresolved clashes to me, and a separate synthesis agent writes the final plan. |
| [`ddd-review`](skills/ddd-review/SKILL.md) | Audits a service's bounded contexts and layering — are they the right partition, and does the code respect them? Emits a violation ledger, a keep/rename/split/merge verdict per context, a file-move list, and a phased remediation plan. |
| [`doc-hygiene`](skills/doc-hygiene/SKILL.md) | Scores a repo's docs out of 100 — discoverability, lifecycle state, link integrity, and whether claims are *true against the actual code* — then remediates. Hard stop for approval between audit and fix. |
| [`ux-discovery`](skills/ux-discovery/SKILL.md) | Interviews me about a product, then writes a Product & UX Discovery Specification for handoff to a design AI. Documents users, jobs, workflows, screens, states, and rules. |
| [`open-questions`](skills/open-questions/SKILL.md) | Turns a tangle of unresolved decisions into one question at a time, each with a recommendation, waiting for an explicit choice before the next. |
| [`grill-me`](skills/grill-me/SKILL.md) | Interrogates a plan until every branch of the decision tree is resolved. Explores the codebase rather than asking what it could look up. |
| [`playstore-listing`](skills/playstore-listing/SKILL.md) | Turns a repo's existing device captures and icon into a full Play Console listing — every text field, every graphic, and a script that rebuilds both. Generates no new artwork. |
| [`play-console-answers`](skills/play-console-answers/SKILL.md) | Drafts the Play production-access questionnaire, grounded in the app's real features, each answer under 300 characters. |
| [`plain`](skills/plain/SKILL.md) | Re-explains whatever just happened in plain English. |
| [`caveman`](skills/caveman/SKILL.md) | Compresses output ~75% while keeping every technical detail exact. Six intensity levels. |

## What they have in common

**Constraints are prohibitions, not preferences.** `ux-discovery` is forbidden from
emitting a single colour, font, or spacing value — "not even as a suggestion" — because
an agent asked politely to stay out of visual design will drift into it by the third
screen. `doc-hygiene` cannot touch a file until I approve the audit. `playstore-listing`
may not invent artwork.

**The panel is many agents, not one agent doing voices.** `roundtable` runs every seat
as an isolated subagent. One model role-playing five experts in a single pass produces
consensus; five agents that have never seen each other's output produce the
disagreement that is the entire point.

**Asking is a first-class output.** `open-questions` and `grill-me` exist because the
expensive failure is an agent guessing confidently on an underspecified task. A
recommendation attached to each question makes answering cheap without letting the
agent proceed alone.

**They are load-bearing, not demos.** Two apps on Google Play
([TuitionHub](https://play.google.com/store/apps/details?id=com.shooque.tuitionhub),
[MooChild](https://play.google.com/store/apps/details?id=com.shooque.moochild)) had
their listings and production-access answers produced by the two Play skills here.

## Install

Skills live in `~/.claude/skills/`. Symlink the ones you want:

```bash
git clone git@github.com:iam-amanxz/ai.git
ln -s "$PWD/ai/skills/roundtable" ~/.claude/skills/roundtable
```

Then invoke by name — `/roundtable`, `/doc-hygiene` — or just describe the task and let
the agent match on the skill's `description`.

## Related

[**campus-advisor**](https://github.com/iam-amanxz/campus-advisor) — a multi-agent RAG
system built with this way of working. Its `WORKFLOW.md` documents the plan, the
harness, and the phase prompts that produced it.
