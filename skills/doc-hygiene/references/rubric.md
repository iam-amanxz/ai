# Scoring rubric — 100 points across seven dimensions

Score each dimension, show the arithmetic, and name the specific docs that cost
points. A score with no citations is unfalsifiable and therefore useless.

Rounding: whole points. When genuinely split between two bands, take the lower
one and say why in the reason line — an optimistic score gets the remediation
deprioritised.

| # | Dimension | Points |
|---|---|---|
| 1 | Discoverability & index | 15 |
| 2 | Housing & structure | 15 |
| 3 | Lifecycle state | 20 |
| 4 | **Truthfulness — claims vs code** | **25** |
| 5 | Reference integrity | 10 |
| 6 | Ownership | 5 |
| 7 | Enforcement | 10 |

---

## 1. Discoverability & index (15)

Can a newcomer find the right doc in one hop?

| Condition | Points |
|---|---|
| One index, generated from the docs themselves, covering every doc | 15 |
| One index, hand-maintained and accurate | 10 |
| One index, hand-maintained and demonstrably drifted (entries missing or dead) | 5 |
| Multiple competing indexes, or none | 0 |

Probes:

```sh
find . -iname 'README.md' -o -iname 'INDEX.md' -o -iname 'SUMMARY.md' | grep -v node_modules
```

Then verify: does the index list files that no longer exist, and does the tree
contain files the index never mentions? Count both. A hand-maintained index is
only worth its 10 points if both counts are zero.

Deduct 5 if the index is a wall of links with no status or one-line summary —
a link list tells you a doc exists, not whether to trust it.

## 2. Housing & structure (15)

Is every doc where its type says it belongs?

| Condition | Points |
|---|---|
| Typed grouping (by folder or an enforced frontmatter type), every doc conforming | 15 |
| Typed grouping, a handful of stragglers | 10 |
| Single flat docs dir, mixed types, no grouping | 6 |
| Docs scattered across unrelated locations, or docs about this repo living outside it | 0–3 |

Probes:

- Docs loose at the repo root that aren't README/CONTRIBUTING/LICENSE/CHANGELOG.
- Docs about *this* component living in a parent, sibling, or another repo. Grep
  a monorepo root and sibling repos for this component's name in doc filenames
  and H1s. **This is the highest-value structural finding and the easiest to
  miss** — misplaced docs are invisible to the people who need them and are
  usually the reason a doc went stale in the first place.
- Two docs whose H1s describe the same subject, with no `supersedes` link
  between them. Duplicates without a supersession chain are worse than either
  doc alone, because the reader can't tell which one lost.
- Naming: is there a convention, and do outliers exist? A redundant prefix
  (`<servicename>-*.md` inside that service's own repo) is noise.

## 3. Lifecycle state (20)

Can you tell, without reading the body, whether each doc is a proposal, in
flight, done, or abandoned?

| Component | Points |
|---|---|
| Machine-readable state on every doc that has a lifecycle (frontmatter `status` or equivalent) | 8 |
| A defined, closed status vocabulary — not free text | 4 |
| Dates: created + last-substantively-updated | 4 |
| Supersession recorded both ways (`supersedes` / `superseded_by`) where docs replaced each other | 4 |

Scale each partially: if 30 of 40 docs carry a status, that's 6/8.

Probes:

```sh
# frontmatter coverage
for f in $(find <docsdir> -name '*.md'); do head -1 "$f" | grep -q '^---' || echo "no frontmatter: $f"; done
# the vocabulary actually in use
grep -rhoE '^status: *.*' <docsdir> | sort | uniq -c | sort -rn
```

A vocabulary with 9 spellings of "done" (`IMPLEMENTED`, `DONE`, `shipped`,
`Implemented ✅`…) scores 0 on the vocabulary component. Free-text status is
prose wearing a field's clothing.

Do not award the state points for a `**Status:**` line in the body *and* a
frontmatter field — two homes for one fact is a drift generator. Score the
frontmatter, and log the duplication as a finding.

## 4. Truthfulness — claims vs code (25)

**The differentiator.** Everything else is tidiness; this is whether the docs
lie. Weight it accordingly and never skip it because it's slow.

Verify, per doc, and count the failures:

| Claim type | How to verify |
|---|---|
| "Implemented / shipped / done" | Find the artifact. A migration file, a table, a symbol, an endpoint, a flag. Absent → the claim is false. |
| "Not implemented / planned / TODO" | Same search inverted. Present → the doc is behind the code, which is the more dangerous direction: someone will rebuild it. |
| Named files, symbols, endpoints, tables, env vars | `git grep` each. Missing → drift. |
| Architecture/transport claims ("via X", "over Y") | Grep for the mechanism. A doc describing a retired transport is the worst category — it reads as authoritative and is wholly wrong. |
| Counts ("four events", "12 endpoints") | Count them in the code. |
| Cross-repo contracts (specs handed to another team) | Verify against the consumer/producer that exists today. A wrong spec propagates into someone else's sprint. |

Scoring:

| Condition | Points |
|---|---|
| No false claims found in the verified sample | 25 |
| Cosmetic drift only (renamed file paths, stale counts) | 18 |
| One doc materially wrong about behaviour or state | 12 |
| Several docs wrong, or one *spec-for-another-team* wrong | 5 |
| Docs describe an architecture that no longer exists | 0 |

Risk ranking for sampling — verify in this order:

1. Specs another team implements against. Wrong here costs someone else's time.
2. Runbooks. Wrong here costs an incident.
3. Anything claiming a status.
4. Docs whose subject area has recent, heavy commit activity
   (`git log --since='3 months ago' --name-only` → which docs cover the hottest
   paths).
5. Docs untouched longest.

Cheap signal for where to look: compare a doc's last commit date against the
last commit date of the code it describes.

```sh
git log -1 --format=%ad --date=short -- <doc>
git log -1 --format=%ad --date=short -- <the src dir it documents>
```

Months of gap is not proof of staleness, but it is the right place to spend
verification effort.

## 5. Reference integrity (10)

| Condition | Points |
|---|---|
| Every relative link and cited path resolves | 10 |
| A few broken links, all inside docs already flagged stale | 6 |
| Broken links in current docs | 3 |
| Links pointing at files that never existed in history | 0 |

That last band deserves its own check: a reference to a doc that was never
committed means someone wrote the pointer and never landed the target, or the
target lives in another repo. Both are findings.

```sh
git log --oneline -1 -- <the-referenced-path>   # empty output = never existed here
```

Portable link check (works with no deps):

```sh
python3 - <<'PY'
import pathlib, re
root = pathlib.Path('.')
for md in root.rglob('*.md'):
    if any(p in md.parts for p in ('node_modules', '.git', 'dist', 'build')): continue
    for m in re.finditer(r'\]\((?!https?:|mailto:|#)([^)\s]+)\)', md.read_text(errors='ignore')):
        t = m.group(1).split('#')[0]
        if t and not (md.parent / t).exists(): print(f'{md}: {t}')
PY
```

Also check inbound: code comments, protos, CI config and other repos that cite
docs by path. Those break silently on a rename and nobody notices.

## 6. Ownership (5)

| Condition | Points |
|---|---|
| Every doc names a current owner; gated docs (decisions, approved plans) also name an approver | 5 |
| Owner on most docs | 3 |
| Owner via git history only | 1 |
| Nothing | 0 |

`git log` is not ownership. It tells you who typed it, not who answers for it
now. One person, never a team — a team-owned doc is an unowned doc.

## 7. Enforcement (10)

Will this repo drift back to its current score in six weeks?

| Component | Points |
|---|---|
| An automated check (CI or hook) that fails on doc-convention violations | 6 |
| A written, discoverable convention (`CONVENTIONS.md` or equivalent) | 2 |
| A convention referenced from where agents and newcomers actually look (`CLAUDE.md`, `CONTRIBUTING.md`) | 2 |

A convention nobody enforces is a preference. Score the check, not the
intention. If the repo has a lint/CI pipeline, note whether adding the check is
one line there — that makes it a cheap remediation step.

---

## Reporting the score

```
Documentation health: 47/100 (D)

| Dimension              | Score | Why |
|------------------------|-------|-----|
| Discoverability        |  5/15 | README index lists 6 docs; 31 exist |
| Housing                |  3/15 | 28 docs about this service live in the monorepo root |
| Lifecycle state        |  6/20 | 12/31 have a status; 9 spellings of "done"; no dates |
| Truthfulness           |  5/25 | 3 docs describe the retired Redis transport; 6 statuses contradict the code |
| Reference integrity    |  3/10 | 14 broken links; 2 point at files never committed |
| Ownership              |  1/5  | git history only |
| Enforcement            |  0/10 | none |
```

Say how many docs you deep-verified versus counted. "Verified 20 of 31; the
remaining 11 are counted for structure only" is honest. Silence implies full
coverage you didn't do.
