# Remediation playbook

Only after explicit approval. Work the approved steps in the order below —
it is dependency-ordered, and skipping ahead creates rework.

## 0. Back up

```sh
cp -r <docs-tree> "$SCRATCHPAD/docs-backup"
```

One command, and every later step becomes survivable. Do it even when the repo
is clean — a bad regex across 40 files is not something `git checkout` helps
with if the files were untracked.

## 1. Establish the convention before moving anything

Write the conventions doc first, so every later step has a target. **Adapt to
what the repo already does** — if it uses `adr/` with MADR-style headers, keep
MADR; if statuses are `Accepted`/`Superseded`, keep those words. You are
codifying and tightening, not replacing.

Frontmatter schema — the useful core, in this order:

```yaml
---
title: One line. May differ from the H1.
status: <from the per-type vocabulary>
owner: <one person>
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

Add per type, only where they earn their place:

| Field | Where | Why |
|---|---|---|
| `approver`, `approved_on` | decisions, plans | gated artifacts need a signer |
| `repos` / `components` | plans spanning more than one | tells you where the work lands |
| `migrations` / `reserved-ids` | plans claiming numbered resources | collision check across sibling plans |
| `supersedes`, `superseded_by` | anything replaced | the chain a reader needs |
| `related` | real coupling only | not "vaguely adjacent" |
| `note` | anywhere | one line: what's left, what's deferred, why parked. **The field people are most grateful for later.** Write it as a handover. |

Three rules that prevent most drift:

1. **One home per fact.** If `status` is in frontmatter, delete the
   `**Status:**` line from the body. Two homes guarantee divergence.
2. **Type from location** (folder) or from an enforced field — never both.
   Duplicating type in frontmatter *and* folder means a moved file lies.
3. **Closed vocabularies.** Per type, because "shipped" is meaningless for a
   runbook and "accepted" is meaningless for a plan. Suggested:

| Type | Vocabulary |
|---|---|
| plan | `draft` → `approved` → `in-progress` → `shipped`, plus `parked`, `superseded` |
| decision / ADR | `proposed` → `accepted`, plus `superseded`, `deprecated` |
| guide, spec, runbook | `current`, `draft`, `stale` |
| incident | `open` → `mitigated` → `closed` |

`stale` is not an admission of failure, it's a load-bearing status: it tells a
reader "don't trust this" in one word, which is far more useful than a
confident doc that's wrong. Use it freely and immediately.

## 2. Set statuses from evidence, never from the prose

For each doc, find the artifact and derive the status. Then put the evidence in
`note`: `"migration 178…-Foo.ts + 15 files under src/x"`, or
`"plan doc committed abc1234; no such symbol in src"`.

This is where the audit's verification work pays off. Two directions matter:

- A doc claiming *done* with nothing in the code → `approved`, not `shipped`.
- A doc claiming *not done* that's actually built → `shipped`. Missing this one
  gets the work done twice.

Commit messages lie about scope. `feat(x): enable Y` that touches only a plan
doc is a plan doc, not a feature. Check the diffstat:

```sh
git show --stat <sha> | head -20
```

## 3. Add frontmatter mechanically

One script, a per-file table of values you derived in step 2, run once.

Hard-won specifics:

- **Anchor your regexes to the header block.** A pattern like
  `^\*\*(Status|Repos?)` will match `**Repository stays…**` and
  `**Status vocabulary change…**` deep in the body. Scope the strip to
  everything before the first `##`, and diff the removed lines afterwards:

  ```sh
  diff <(grep -v '^$' backup/x.md) <(grep -v '^$' x.md) | grep '^<'
  ```

  Read that output. Every time.
- **Don't globally replace quotes.** `.replace("'", '"')` turns `tenant's` into
  `tenant"s` across every note. Quote only values that need it (those containing
  `: `).
- Soft-wrapped metadata lines have continuation lines. Consume them or you'll
  leave orphan fragments mid-document.
- Preserve information the old header carried that your schema doesn't — fold it
  into `note` rather than dropping it.

## 4. Move and rename

`git mv` for tracked files so history follows; plain `mv` for untracked. Then,
in the same step, fix every reference:

```sh
# outbound: links inside the moved docs, whose depth changed
# inbound: everything else in the working set that cited the old path
git grep -n 'old/path' -- . ':!node_modules'
grep -rn 'old/path' <sibling-repos> <monorepo-root> 2>/dev/null
```

Search code comments, protos, CI YAML, and other repos — not just markdown.
Those citations break silently.

Depth changes are the fiddly part: a doc moved one level deeper needs every
`../` prefix extended. Compute the new prefix from the file's depth rather than
hand-editing:

```python
depth = len(md.relative_to(DOCS).parts) - 1
up = "../" * (depth + 1)
```

Then re-run the link checker from rubric §5 and fix what it finds. Distinguish
three cases in the output:

- Depth wrong → fix the prefix.
- Target renamed → repoint.
- Target never existed → it's a finding, not a fix. Either the pointer was
  aspirational or the target is in another repo. Say which, in the doc.

## 5. Generate the index

The index must be generated. A hand-maintained index is a second copy of every
status, and it will drift within weeks.

Write it in the repo's own language and wire it into the existing script runner
(`package.json`, `Makefile`, `pyproject`) so it's discoverable. Two entry points:

```
<runner> docs:index     # rewrite the index
<runner> docs:check     # verify only; non-zero exit on stale index or violation
```

The generator should:

- Read frontmatter (a flat `key: value` parser is plenty — resist adding a YAML
  dependency for five keys).
- Group by type, then status, newest first within each group.
- Emit a `<!-- GENERATED by … -->` header line so nobody hand-edits it.
- Carry `note` into the table. A table of titles and dates isn't worth
  generating; the note is the payload.
- **Validate while it walks**: unknown status for the type, missing required
  field, file in an unknown folder, second file named `README.md`. Print
  violations into the index itself *and* fail `--check`. A violations section
  visible in the index is what keeps it honest.
- Aggregate any collision-prone claims (reserved migration timestamps, ADR
  numbers, port numbers) into one table. This turns scattered prose warnings
  into a single lookup.

`--check` in CI next to the existing lint/test steps is the whole enforcement
dimension. Without it the score decays.

## 6. Point the agents and newcomers at it

Add a few lines to `CLAUDE.md` (or `AGENTS.md`) and `CONTRIBUTING.md`: where
docs live, the required frontmatter, the two commands, and the one rule that
matters most — *status describes the code, not the document*. If a memory or
rules file records an older convention, update it; a stale instruction file
actively fights the new convention.

## 7. Rewriting a stale doc

When a doc describes an architecture that no longer exists, marking it `stale`
is triage, not the fix. To actually rewrite:

1. **Read the code, not the old doc.** Consumers, handlers, wire encoders, error
   paths, config, the retry/failure machinery. Derive the contract from source.
2. **Diff the catalog.** List what the doc claims exists against what does. In
   practice you'll find invented items, missing real ones, and renamed ones —
   all three.
3. **Name what changed for the reader**, loudly. A contract change buried in a
   table gets missed. If a failure mode moved from "retries forever" to
   "dead-letters in 13 minutes", that deserves its own section with the number
   in it, because it changes what callers must guarantee.
4. **Keep the honest negatives.** Handlers that exist but nothing dispatches;
   fields written through without validation; asymmetries between two events
   that ought to match. Documenting a wart beats silently smoothing it — the
   next reader needs to know it's a wart, not discover it in production.
5. **Include the one likely porting mistake.** If the old shape wrapped payloads
   and the new one doesn't, say so where someone porting will look, and put it
   in the acceptance tests.
6. **Reset the frontmatter**: `status: current`, `updated` today, and a `note`
   saying it was rewritten against the code and on what date.

Do not rewrite a spec another team implements against without flagging the
contract deltas separately in your final report. They have to act on those.

## Verification before reporting done

- Link checker clean, or every remaining break explained.
- `docs:check` (or equivalent) exits 0.
- Project typecheck/lint still passes if you touched code, config or scripts.
- Every status you set traces to evidence in its `note`.
- The backup is still on disk until the user confirms they're happy.

## Final report

- Score before → after, per dimension.
- What changed, by category, with counts.
- **What you found while fixing that the audit missed.** Normal, and the most
  valuable part of the report.
- What you deliberately left, and why.
- Anything needing a decision from someone else: cross-repo contract deltas,
  code bugs found in passing, docs whose owner you had to guess.
