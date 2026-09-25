# State

Central, keyed by `nameWithOwner`. **Never keyed by path** — reviews are routinely run
from git worktrees, and path-keyed state fragments across them and vanishes when one is
removed.

Single machine. No sync. Accepted limitation.

```
~/.claude/reviews/
  index.md                      cross-repo rollup
  repo-path-map.json            nameWithOwner -> local path
  contributor-repos.json        admin+contributor set, with contributor counts
  <owner>-<repo>/
    settings.json               per-repo settings (attribution)
    review-focus.md             accumulated review knowledge
    prs/
      1105.md                   ledger + decisions
      1105-body.md              persisted draft body
```

## `prs/<n>.md`

The durable artifact. The posted body is on GitHub and the analysis is re-derivable from
the diff — **what was raised, what was deliberately dropped, and why, exists nowhere
else.**

```markdown
---
repo: sillsdev/FieldWorks
number: 1105
title: "LT-22728: Scope local libraries to one build"
author: johnml1135
created: 2026-08-21
status: posted            # analyzed | in-progress | drafted | posted | declined
verdict: request changes
head_sha_at_review: 260593ec3
reviewed_at: 2026-08-26T18:29:28Z
---

## Findings

### 1. Nested restore drops the version override — required
`Build/PackageRestore.targets:100-119` Execs a fresh restore with only
/p:Configuration and /p:Platform.
**Decision:** require a verification run with palaso or lcm. Framed as "the existing
evidence structurally cannot exercise this", not as an asserted bug.

### 2. Every-build cleanup — required
**Decision:** push back on the design. Cleanup is deliberate, so it can be blunt and
should be visible. Folded in that CleanNuGet was the predecessor, removed in #678.

### 3. Installed-path walk cost — dropped
**Decision:** skipped. "Not slow enough to matter."
```

`declined` is terminal and means the user has decided not to engage further — either they
read the PR and posted nothing, or they reviewed it, closed their threads out and expect no
further round. The queue skips those in **every** tier, tier 1 included (see `queue.md`),
because a `posted` record otherwise promotes to tier 1a on every push the author makes. The
record is kept, not pruned: its findings and dispositions are the reason not to analyse the
PR again.

Findings keep their number, a disposition (`required` / `question` / `dropped`), and a
one-line reason. Pending findings are listed with no decision — that is what makes a
review resumable.

## `prs/<n>-body.md`

The draft review body. **Prose rewrites are first class**: the user may edit this file
directly, and their words are never silently replaced.

When a later decision requires regeneration, compose the new body, **diff it against the
persisted file, and show what would change to user-written prose before writing.**

On resume, if the head SHA has moved since the body was drafted, say so before touching
it. Persisted prose describing an older diff is the one way this design produces a
confidently wrong review.

## `settings.json`

```json
{ "attribution": true }
```

Attribution defaults on. Decide once per repo rather than once per review — partial
disclosure across a repo's history reads worse than either consistent choice.

## `review-focus.md`

Only what convention discovery cannot give: which findings recur, which rules keep being
broken. Not a restatement of `AGENTS.md`.

```markdown
# Review focus — sillsdev/FieldWorks

- Comment-hygiene violations appear in nearly every PR; check added comments against
  the length cap and the banned content categories before anything else.
- PR bodies routinely overstate what was verified. Check evidence claims against the
  tree; several have been false.
- Tests that construct WinForms controls. Recurring, and usually a symptom of logic
  welded to the UI rather than a style slip.
- `-CommentHygiene` skipped or unverified.
```

**Propose additions when a finding repeats. Never write silently.**

## `index.md`

One line per repo with open state, newest activity first. Read by `/review` from
outside any repo, so parent-directory mode never scans the filesystem.

## Pruning

**When a PR closes or merges, delete its ledger and body.** No exceptions, including PRs
closed by the user's own rejection — those are unlikely to be overridden, and if the work
returns it returns in a new shape.

Prune on queue build, so closed PRs do not accumulate.
