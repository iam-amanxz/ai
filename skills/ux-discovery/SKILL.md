---
name: ux-discovery
description: "Interview the user about a product, then produce a Product & UX Discovery Specification for handoff to a UI/UX design AI (Claude Design, v0, etc.). Documents users, jobs, real-world workflows, screens, data, states, business rules, and priorities — and deliberately specifies NO visual design. Use when the user says /ux-discovery, 'ux discovery', 'discovery spec', 'design brief', 'spec for a design AI', 'what should the designer know', or is about to hand a product to a design tool."
---

# /ux-discovery

Produce a **Product & UX Discovery Specification**: everything a design agent needs to design the product correctly, and nothing about how it should look.

## The one hard rule

**You are not designing the UI.** Never output colors, typography, font sizes, spacing, design tokens, radii, shadows, component styling, visual themes, breakpoints, pixel layouts, element positions, component libraries, animations, or decorative elements. Not even "as a suggestion". Not even in the handoff section.

You *do* specify behavior: what must be reachable in one step, what the system must remember, what must be clear next, what must be prevented. Behavioral requirement, never its visual implementation.

Conflict between documenting the product and describing the interface → document the product.

## Flow

1. **Ground yourself first.** If there's a codebase, repo, PRD, or existing app in scope, read it before asking anything. Don't spend the user's questions on facts sitting in the files. If `graphify-out/` exists, query it.
2. **Interview** (below). Groups of 2–4 related questions, never a wall of 50. Adapt to answers. Vague answer or an unexplored area that matters → follow up before moving on.
3. **Draft the spec**, save to `./ux-discovery-spec.md` (or where the user says).
4. **Show the Open Questions section** and let the user resolve or accept the assumptions.

For the interview mechanics — one decision at a time, each with a recommendation — the `open-questions` skill already does this; use it rather than reinventing the loop.

Never assume a requirement the user hasn't given. You may *propose* one as a question, but anything unconfirmed lands in §13 as an assumption, labeled.

## Interview: what to actually dig for

Ask in this order, but skip what the code or docs already answer.

**Product** — what it does, the problem it solves, what people do today instead, what "working" looks like.

**Users** — roles, and for each: what they're trying to get done, what's painful now, how often they use it, where they physically are when they use it, on what device, and whether they're focused at a desk or one-handed mid-task. Demographics are useless here; behavior isn't.

**Workflows** — for each major one: trigger, goal, what they do *before* opening the app, what they need to know, what they enter, what they decide, what they do, what can go wrong, desired end state, frequency, importance. Think **goal → actions → decisions → outcome**, not feature → screen.

**Jobs** — extract the discrete jobs, classify each by frequency (daily/weekly/monthly/occasional/rare) and importance (critical/high/medium/low), plus user type, trigger, outcome, dependencies, failure modes. Rank them. Not every feature deserves equal prominence.

**Structure** — what the product must contain: major areas, primary views, supporting views, settings, detail views, create/edit flows, search and filtering, notifications, auth/onboarding. Don't invent a screen because products conventionally have one. If a thing fits inside a workflow the user is already in, say so — unnecessary fragmentation is the failure mode.

**Friction** — hunt the workflows for: repeated data entry, navigation that exists only to satisfy the app's structure, redundant confirmations, things the system could remember, actions that should happen together, forced context switches, dead ends where the next step is unclear, decisions the system could reasonably make itself, and points where the user needs information *before* they can decide. Don't make users learn the app's structure to do their task.

**Data** — entities, relationships, required vs optional fields, user-generated vs system-generated, what gets searched/filtered/sorted, what needs history or audit, what can be edited/deleted/archived.

**States** — only the ones that really apply: first use, empty, loading, success, failure, validation error, no results, partial data, missing info, offline, permission-restricted, deleted/archived, very large data sets, conflicting data. Don't manufacture edge cases.

**Rules** — who can do what, who can't, what needs confirmation or approval, what happens after an action, dependencies, limits, required fields, and which actions are irreversible. Confirmed rules and assumed rules go in separate lists.

## Deliverable

```
1.  Product Overview
2.  Users & Roles
3.  Core User Jobs             — prioritized
4.  User Workflows             — real-world, step-level
5.  Feature Requirements
6.  Screen/View Inventory      — purpose, users, why it exists, key info,
                                 key actions, workflows in, workflows out, priority
7.  Navigation & Workflow Relationships   — functional, not visual patterns
8.  Data & Information Requirements
9.  States & Edge Cases
10. Business Rules & Constraints
11. UX Requirements            — behavioral only
12. Priority Matrix            — critical/high/medium/low, with reasoning
13. Open Questions & Assumptions  — confirmed / assumed / undecided, separated
14. Design-Agent Handoff
```

**§11 UX Requirements** — derive a *small* set of principles from this product, not a generic list. Only what's actually load-bearing here. Shape: "Frequent task X must complete without leaving context Y", "the system must not re-ask for Z", "the user must be able to see A before deciding B".

**§14 Design-Agent Handoff** — self-contained brief a design AI can work from cold: who the users are, what they need to accomplish, the top workflows, the top screens, what information matters, which actions matter, which flows must be especially efficient, key states and constraints, the UX principles. Same hard rule applies — no visuals.

## Quality bar

The spec fails if it reads like it could describe any SaaS product. Keep interviewing until you understand how this thing gets used **in real life** — the specific person, the specific moment, the specific mess they're in when they open it. The goal isn't to document every feature; it's to give the design AI enough grasp of the people, problems, workflows, priorities, information, actions, and constraints that it doesn't have to guess.
