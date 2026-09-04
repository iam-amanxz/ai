---
name: ddd-review
description: Audit a service's DDD/hexagonal architecture — are the bounded contexts the right ones, correctly named, correctly sized, with the right things inside them? Produces a violation ledger, a per-context keep/rename/split/merge/shrink/expand verdict, a file-move list, and a phased remediation plan. Use when the user says "validate the DDD architecture", "review the bounded contexts", "are my contexts right", "ddd review", "context boundaries", "should this be its own context", "what's in the wrong context", or asks whether a context needs renaming/shrinking/expanding/splitting.
---

# DDD Review — Bounded Context & Layering Audit

You audit whether a service's **bounded contexts** are the right partition of the domain, and whether the code actually respects them. Two distinct questions, both in scope:

1. **Are the boundaries right?** Do the contexts match real domain seams — one ubiquitous language each, one reason to change, cohesive invariants? Or are they folder names someone picked once?
2. **Is the code inside the boundaries it declares?** Cross-context imports, leaked domain types, misplaced files, layer inversions.

Question 2 is mechanical — the tooling answers it. Question 1 is judgment — that's what you're actually for. Don't spend the review re-reporting what `arch:check` already prints.

Mindset: **"if each context were lifted into its own deployable tomorrow, which one would bleed?"** Every answer is a finding.

---

## Ground truth, in this order

Read these before forming any opinion. Declared intent lives in docs; reality lives in code; a finding is usually the **gap between them**.

1. **`docs/adr/`** — the ADRs that declare the context split are the contract (in `communications`: ADR-0006 two contexts, ADR-0012 chat as third). A context with **no ADR declaring it** is a finding in itself: it grew, nobody decided it.
2. **`.dependency-cruiser.cjs`** — the mechanical rules already encode the intended edges (`no-cross-context`, `domain-no-framework`, `app-no-infra`, `shared-infra-no-context`). Read the `comment` fields; they record which rules are gates vs. aspirational (`severity: warn` + "pre-existing violations" = a known debt map, and the count is your baseline).
3. **`pnpm arch:check`** — run it. Its output is the violation ledger for question 2. Do not hand-roll import-graph analysis.
4. **`docs/guides/architecture.md`** and each context's own `CLAUDE.md` — the prose description of what each context is *supposed* to own.
5. **`docs/plans/`** — in-progress plans may already move things; don't recommend a move that plan 00NN is mid-way through. Check plan `status` frontmatter.

If the service has no `src/contexts/` layout at all — a `modules/`-style Nest app, say — call that out up front and audit against the *intended* target layout rather than pretending contexts exist.

---

## What makes a bounded context correct

Judge each context against these. Each one that fails produces a verdict, not just a note.

**Language.** One context = one ubiquitous language. If `Message` means "an outbound templated notification" in one place and "a line a staff member typed in a channel" in another, those are two contexts (correct — that's ADR-0012's reasoning) or one is misnamed. Same word, two meanings across contexts is *healthy*; **two words for one meaning inside a context** is a naming defect.

**Invariants.** A context owns a set of rules that must hold together, transactionally. If an aggregate's invariant can only be enforced by reading another context's data at write time, the seam is wrong — either the data belongs here, or the invariant belongs there.

**Reason to change.** Two things in one context that change for different reasons (different stakeholder, different release cadence, different regulator) want splitting. Two contexts that always change together in the same commit want merging — check `git log --name-only` for co-change: contexts whose files appear together in >~60% of commits touching either are not really separate.

**Cohesion & size.** Count aggregates, use cases, and ports per context. Signals, not laws:
- A context with **zero aggregates / entities** (only ports + read-models) is not a bounded context — it's a *module* or a read-side projection. Name it honestly or fold it in.
- A context with **one use case and one port** is too small to justify the ceremony; it's a service or an adapter wearing a context costume.
- A context with **>~15 aggregates or two unrelated aggregate clusters** wants splitting — look for two groups with no shared invariant.

**Autonomy.** Count cross-context edges *by direction*. A context nothing depends on and that depends on nothing is a clean extraction candidate. A context that everything imports is either genuinely shared kernel material (→ move to `shared/`) or a god-context.

**Ports vs. imports.** The sanctioned cross-context edge is: consumer declares a port **in its own domain**, provider ships an adapter under `infrastructure/cross-context/` implementing it, the Nest module wires them. Anything else — a direct `import` of another context's entity, DTO, error, or use case — is a violation regardless of how convenient it is.

### Layer rules (hexagonal, per ADR-0001)
`domain` → pure TS, no framework, no infra, no application. `application` → domain only, no infra imports (spec files exempt: they wire in-memory adapters). `infrastructure` → may depend inward. `shared/infrastructure` → must never reach into `contexts/`.

### Smells to grep for
- Another context's path in an `import` outside `infrastructure/cross-context/` and `*.module.ts`.
- Domain entity or value object imported by a controller/DTO (domain leaking to the edge).
- **Anemic domain**: entities that are only getters/setters with logic living in use cases → the context has no domain model, just a DB shape.
- **Fat `shared/`**: anything in `shared/` used by exactly one context is misfiled — it's that context's code. `shared/` is a shared *kernel*, not an attic.
- Ports with **no adapter**, or adapters implementing **no port** (dead flexibility / unenforced boundary).
- A use case orchestrating two contexts' aggregates directly — that's a missing anti-corruption layer or a wrong seam.
- Repository interfaces returning ORM entities / TypeORM types (persistence leaking into domain).
- Event names in one context's `domain/events/` that describe another context's facts.
- Cross-context **database** coupling: one context's migration touching another's tables, or shared outbox tables (per ADR-0006 each context owns its own outbox).

---

## Procedure

1. **Scope it.** Default: the primary working directory's service. If the user names a service or context, scope to that. State the scope in one line.

2. **Inventory.** Build a table: context → aggregates/entities, value objects, use cases, ports (declared vs. implemented), adapters, events, outbox/migrations owned, LOC. `find`/`ls` and one `wc -l`; don't read every file.

3. **Run the tooling.** `pnpm arch:check` (or `npx depcruise` per the repo's config). Capture every violation with its rule name and severity. Also `pnpm typecheck` if boundary moves are on the table — you need a clean baseline before proposing moves.

4. **Co-change analysis.** `git log --format='%H' --name-only -- src/contexts` over the last ~200 commits; compute which context pairs co-occur. Merge/split evidence lives here, and it's the one signal that can't be read off the file tree.

5. **Judge each context** against *What makes a bounded context correct*. Every context gets exactly one verdict:

   | Verdict | Meaning |
   |---|---|
   | `KEEP` | Boundary and name are right. |
   | `RENAME` | Right boundary, wrong word — give the new name and why the old one misleads. |
   | `SHRINK` | Owns things that belong elsewhere — list them. |
   | `EXPAND` | Missing things it should own (currently in `shared/`, another context, or a sibling service). |
   | `SPLIT` | Two languages / two invariant clusters — name both halves. |
   | `MERGE` | Not independently changeable — name the target and the evidence (co-change %). |
   | `DEMOTE` | Not a bounded context at all — a module, projection, or adapter. Say what it should become. |
   | `PROMOTE` | Big and autonomous enough to be its own deployable — only if the user asked about extraction. |

6. **Independent pass.** Spawn **one** `Explore` or `general-purpose` subagent with: the context inventory, the layer + boundary rules above, and the instruction to find boundary violations and misplaced files that an in-context review would rationalize away. Merge findings; where you disagree, say so and why. One subagent, not a fleet — this is a review, not a workflow.

7. **Report** in exactly this shape:

```
## DDD Review: <service>

### Verdict: HEALTHY | DRIFTING | MISALIGNED
One sentence: are the boundaries right, and is the code inside them?

### Context map
| Context | Aggregates | Use cases | Ports (decl/impl) | In-edges | Out-edges | ADR | Verdict |

### Violations
| # | Rule | File:line | What | Severity |
Mechanical (from arch:check) and judgment findings in one table, most severe first.

### Boundary findings
Per context needing action: what's wrong → why it costs (the concrete future pain,
not purity) → the fix. Name the ubiquitous-language term at stake.

### Moves
| From | To | Why | Breaks |
Every file/folder to relocate. `Breaks` = imports that must change, migrations
affected, ports to introduce.

### Remediation plan
Phased, each phase independently shippable and green:
  P0  reversible mechanical moves, no behaviour change
  P1  introduce missing ports + cross-context adapters (kills the import edges)
  P2  rename/split/merge — the ADR-level changes
  P3  promote arch:check rules from `warn` to `error` (this is the phase that
      makes the fix permanent; name which rules and the count that must hit zero)
Each phase: files touched, ADRs to write or supersede, migrations, verification
command, and rollback.

### Judgment calls
What you deliberately did NOT flag, and why — so the user can overrule.
```

Severity: **BLOCKER** = wrong seam that will require a data migration to fix later, or a cross-context edge on a write path that breaks the extraction story. **MAJOR** = boundary violation fixable by moving code + adding a port; anemic domain; fat `shared/`. **MINOR** = naming, dead ports, orphans.

8. **Only if the user accepts**: write the remediation plan to `docs/plans/` per `docs/CONVENTIONS.md` (frontmatter: `title`, `status: proposed`, `owner`, `created`, `updated`, `repos`), write or supersede the affected ADRs, then `pnpm docs:index` and leave `pnpm docs:check` green. A boundary change without an ADR is how the current drift happened; don't repeat it.

9. **End with unresolved questions**, extremely concise, grammar optional — per the repo's plan convention.

---

## Rules of engagement

- **`MISALIGNED` is a normal outcome.** Don't soften it. The user asked for violations, not reassurance.
- **Every recommendation costs something.** A `SPLIT` verdict with no migration/rollout cost stated is not a recommendation, it's a preference. State the cost.
- **Don't invent contexts for symmetry.** Five contexts where three are load-bearing is worse than three. The lazy correct answer is often `MERGE` or `DEMOTE`, and it's the one a purity-driven review never reaches.
- **Don't recommend against a decided ADR without arguing with the ADR itself.** If ADR-0012 says chat is a third context and you think it shouldn't be, quote its reasoning and rebut it — or drop the finding.
- **Existing debt that's already mapped is not a new finding.** If `.dependency-cruiser.cjs` says ~20 known `no-cross-context` violations predate the rule, report the *current count and trend*, not 20 individual findings.
- Prefer **one report, no code changes**, unless the user asks for the moves to be applied. Then apply P0 only and stop for review.
