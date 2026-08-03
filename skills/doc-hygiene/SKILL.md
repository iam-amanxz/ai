---
name: doc-hygiene
description: "Audit a repo's documentation, score it out of 100, and fix it. Scores discoverability, housing, lifecycle state (built/approved/deferred), truthfulness of claims against the actual code, link integrity, ownership and enforcement — then reports, waits for approval, and remediates. Use when the user asks to audit/clean up/organise docs or plans, says they can't tell what's shipped vs planned, mentions stale or scattered docs, or invokes /doc-hygiene."
---

# /doc-hygiene

Three phases, in order, with a hard stop between 2 and 3.

```
/doc-hygiene              # audit → score → report → wait
/doc-hygiene --fix        # skip straight to remediation of an approved plan (same session only)
/doc-hygiene <path>       # scope to a subtree
```

**Phase 1 — Audit (read-only).** Change nothing. Not one file.
**Phase 2 — Report.** Score out of 100, findings, ordered remediation plan.
**Phase 3 — Remediate.** Only after the user explicitly agrees.

The single most common failure of this skill is remediating during the audit
because a fix looked trivial. Don't. The user asked for a score first, and an
unrequested edit invalidates the baseline they're about to approve against.

## Phase 1 — Audit

Read `references/rubric.md` for the seven scored dimensions, their point
allocations, and the exact probes for each.

Order of work:

1. **Find every doc.** All `*.md`/`*.mdx`/`*.rst`/`*.adoc` outside
   `node_modules`, `.git`, `dist`, `build`, `vendor`, `target`, `.venv`,
   lockfile dirs. Include the repo root and any sibling/parent directories the
   user's working set covers — docs about *this* service frequently live in a
   monorepo root or another repo entirely, and that misplacement is a finding.
2. **Classify each doc** by what it *is*, from its content, not its path:
   decision record, plan/design, guide/reference, spec-for-another-team,
   runbook, incident, changelog, generated/tool-owned, README.
3. **Detect the existing conventions** before proposing any. Frontmatter keys
   already in use, status vocabularies already in use, folder layout, any
   index file, any lint/CI hook that touches docs. You are adapting to this
   repo, not installing a house style.
4. **Verify claims against code.** This is the highest-weighted dimension and
   the reason the skill is worth running. See `references/rubric.md` §4.
5. **Score**, then write the report.

Exclude from scoring (note their existence, don't grade them): generated API
docs, vendored docs, tool-owned directories (`openspec/`, `.changeset/`,
`CHANGELOG.md`), and license/contributing boilerplate.

### Non-negotiables during audit

- **No writes.** No `Write`, `Edit`, `sed -i`, `mv`, `rm`, `git add`.
- **Sample, don't skim everything.** Above ~40 docs, deep-verify the highest-risk
  20 (see rubric §4 for the risk ranking) and say in the report exactly how many
  you verified versus counted. Never imply full coverage you didn't do.
- **Every finding cites a path**, and a line number where one applies. A finding
  you can't point at is a hunch — leave it out or label it as a hunch.

## Phase 2 — Report

Print, in this order:

1. **Score** — `NN/100 (grade)`, then the seven dimensions as a table with
   `points earned / possible` and a one-line reason each. Grades: A ≥90,
   B 75–89, C 60–74, D 40–59, F <40.
2. **Inventory** — doc count by type and location; how many you deep-verified.
3. **Findings** — worst first. Each: what, where, why it costs something.
   Separate **factually wrong** (a doc claims something the code contradicts)
   from **structurally messy** (no index, no frontmatter). Wrong beats messy
   every time — a doc nobody can find wastes minutes; a doc that lies wastes an
   afternoon and sometimes ships a bug.
4. **Remediation plan** — numbered steps, each with: effort (S/M/L), point gain,
   blast radius (files touched, whether other repos are affected), and
   reversibility. Order by point-gain-per-effort, except that anything
   destructive or cross-repo goes last regardless.
5. **What I will not touch** — generated files, tool-owned dirs, other repos
   unless the user says so.

Then stop and ask for approval. Offer the obvious partial: "steps 1–4 only" is
usually the right first bite. If the user approves a subset, do exactly that
subset.

## Phase 3 — Remediate

Read `references/remediation.md` for the playbook, the frontmatter schema,
index-generator patterns, and the safety rules.

Rules that override any convenience:

- **Back up first.** Copy the doc tree to the scratchpad before the first
  mutation. One `cp -r`, and it turns every later step into a survivable
  mistake.
- **Never delete or overwrite a doc you haven't confirmed is recoverable.**
  `git log --oneline -1 -- <path>` must show a commit, or the content must be
  reproduced elsewhere. Untracked-and-unread means read it first.
- **Fix references as part of the move that breaks them**, not as a later pass.
  After any rename/move, re-grep the whole working set — including code
  comments, protos, CI config and sibling repos — for the old path.
- **Statuses describe the code, not the prose.** Never copy a status out of a
  doc's own header. Derive it from evidence and cite the evidence in the note.
- **One mechanical script per bulk change**, run once, output reviewed. Don't
  hand-edit 40 files, and don't write a script whose regex you haven't tested
  on one file first. Loose patterns eat body text — anchor them.
- **Verify at the end**: every relative link resolves, any checker you added
  passes, and the project's own typecheck/lint still passes if you touched code
  or config.

Report honestly at the end: what changed, what you left, what you found *while*
fixing that the audit missed. Mid-remediation discoveries are common and belong
in the final report, not buried.

## Scope discipline

Docs work has an unusual pull toward scope creep, because reading a doc reveals
real problems in the thing it documents. When you find a code bug, a missing
alert, an unwired handler: **write it down as a finding, don't fix it.** The
user approved a docs remediation. A code change smuggled in alongside is not
what they agreed to, however small and however right you are.

The one exception: a doc's *own* tooling (an index generator, a CI check) that
you added in Phase 3 and that doesn't pass. Finish what you started.
