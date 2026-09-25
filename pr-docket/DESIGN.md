# pr-docket — design spec

Derived from the FieldWorks review session of 2026-08-24/26 (12 PRs) and a grilling
pass on 2026-08-26. Every decision below was made explicitly; rationale is recorded
where it came from evidence rather than preference.

Status: **built.** This file is the rationale record; `SKILL.md` and `references/` are
the operative text and win where they differ.

---

## 1. What it is

A personal (not repo-scoped) skill that maintains a ranked queue of pull requests
needing the user's attention across every repository, drives one review at a time to a
posted verdict, and tracks enough state that a review can be abandoned mid-stream and
resumed days later.

Lives at `~\.claude\skills\pr-docket\SKILL.md`, alongside the other personal skills.

Named `pr-docket` rather than `review`: a review skill (now the built-in `code-review`)
already exists for reviewing local changes, and a docket — matters awaiting judgment, ranked, worked one at a time,
adjourned and resumed — describes this one exactly.

## 2. Source of truth

**GitHub, not the filesystem.** The queue is built from `gh` queries. The local
checkout is consulted only when code must be read for analysis.

<!-- PERSONAL: counts and paths below describe the original author's machine. -->
Evidence: `C:\Repositories` holds 80 git repos at depth 1 and 6 more at depth 2
(including `fwroot\fw`, the repo the source session ran in). Scanning 86 repos per
launch is slow, misses PRs in uncloned repos, and inverts the relationship — a review
request exists on GitHub whether or not anything is cloned.

A `nameWithOwner -> local path` map is built once and cached, refreshed on a miss.
It must handle both depths.

## 3. The queue

Five tiers, `0` to `4`; tier 1 has two halves, `1a` and `1b`, and "tier 1" alone means both.
Drafts excluded by default (`--include-drafts` overrides).

| Tier | Contents | Order within tier |
|---|---|---|
| **0** | The user's own PR, approved and mergeable — `reviewDecision APPROVED` + `mergeStateStatus CLEAN` | Oldest approval first |
| **1a** | PRs where the user requested changes and the author has since responded — new commits **or** replies | Interleaved with 1b, oldest response first |
| **1b** | The user's own PR with feedback they have not answered | Interleaved with 1a, oldest response first |
| **2** | User is a requested reviewer, nobody has reviewed | Oldest first |
| **3** | User is a requested reviewer, others have reviewed, user has not engaged | Oldest first |
| **4** | Open, no reviews from anyone, in repos where the user is admin **and** contributor | Repo priority, then age |

Tier 0 sits above tier 1 because **shipping finished work beats reading unfinished work.**
It was added 2026-09-22 after an approved PR of the user's own vanished from the docket
entirely: nothing was waiting on a reply, so tier 1b did not claim it, and it is the
user's PR, so no review tier did either. The cost of the omission is asymmetric -- a
missed review is a delay, a missed merge is work that never lands.

Tier 1 exists because **finishing reviews is higher value than starting new ones.**
Closing a loop beats opening one.

Tier 2 is logically a subset of "requested and not engaged"; tier 3 is the remainder.

**Repo priority applies to tier 4 only.** A direct review request is a person waiting,
regardless of repo; letting a `chorus` request sit because `FieldWorks` has older ones
inverts what makes tiers 2 and 3 urgent. Tier 4 is discretionary attention, which is
exactly where "which repo matters" and "where am I likely the only reviewer" belong.

<!-- PERSONAL: the original author's tier 4 repo ranking. Replace items 1-5 with your
own; the operative copy is in references/queue.md under "Tier 4 repo order". -->
Tier 4 repo order:

1. `sillsdev/FieldWorks`
2. `sillsdev/liblcm`
3. `sillsdev/interlinearizer-extension`
4. `sillsdev/flexbridge`
5. `sillsdev/chorus`
6. everything else, **fewest contributors first** — fewer contributors means nobody
   else is likely to review it.

<!-- PERSONAL: the repo counts in this paragraph are the original author's. -->
**Admin + contributor** is computed as the intersection of `user/repos` where
`permissions.admin == true` (485 repos) with the repos returned by
`gh search prs --author=@me` (10 repos, one call). Authored-PRs is a *proxy* for
contributor and will miss a repo the user only pushed directly to. Cached; refreshed
on demand.

Tier 4 is also the sweep — no separate verb, and **no author filtering.** Tier
membership is defined by the rule above, not by whose PRs they are. Author-scoping
would turn the queue from "what needs attention" into "whose backlog am I working
through", which is how the source session happened to run but is not how the queue
should work day to day.

## 4. Invocation

```
/pr-docket                             resume in-progress review; else show the ranked queue
/pr-docket next                        take the top of the queue
/pr-docket list                        the queue as addressable identifiers
                                    (sillsdev/FieldWorks#1105, ...)
/pr-docket <anything>                  resolve to one PR and start it
```

`/pr-docket next` is never automatic — after finishing a review the skill stops and waits.

**Fuzzy resolution** (`/pr-docket FieldWorks PR about toothpaste`): match against title,
branch, and author first; fall back to body only if that yields nothing. Disambiguate
the repo before touching PRs (`FieldWorks` matches both `sillsdev/FieldWorks` and
`sillsdev/fieldworks-analytics-reporting`). **Always confirm before starting**, even on
a single high-confidence match. On no clear match, list candidates in the same
addressable form `/pr-docket list` emits.

## 5. State

**Central, keyed by `nameWithOwner`.** Not repo-local, not path-derived.

```
~/.claude/reviews/
  index.md                              cross-repo rollup
  repo-path-map.json                    nameWithOwner -> local path (cache)
  contributor-repos.json                admin+contributor set (cache)
  sillsdev-FieldWorks/
    review-focus.md                     accumulated repo-specific review knowledge
    prs/
      1105.md                           ledger + decisions
      1105-body.md                      persisted draft review body
```

Path-based keying is rejected on evidence: the source session ran entirely from
`fw\.claude\worktrees\johnl-pr-review`. Repo-local state would have lived in a worktree
and vanished with it.

**One machine.** No git sync. Accepted limitation.

### Per-PR record

- Number, title, author, repo, created date, draft status
- **Head SHA at review time** — the trigger for tier 1a promotion and the staleness check
- Review status: `analyzed` / `in-progress` / `drafted` / `posted`
- Verdict once posted
- Every finding, each with: anchor (file:line or none), disposition
  (`required` / `question` / `dropped`), and a one-line reason

The decision record is the point. The posted body is on GitHub; the analysis is
re-derivable from the diff. **What is raised and what is deliberately dropped, and
why, exists nowhere else** — and it is what stops the same argument recurring next time
the author touches the same file.

### Draft body

Persisted as a file. **Prose rewrites are first class** — the user may edit it directly.

When a later decision requires regeneration, the skill composes the new body, **diffs it
against the persisted one, and shows what would change to user-written prose before
writing.** Nothing the user wrote is ever silently replaced.

On resume, if the head SHA has moved since the body was drafted, say so before touching
it — persisted prose describing an older diff is the one way this design produces a
confidently wrong review.

### Pruning

**Prune the whole record when a PR closes or merges.** No exceptions, including PRs
closed by the user's own rejection: rejections are unlikely to be overridden, and if
work returns it returns in a new shape.

## 6. Analysis

**Two phases.**

**Triage** covers the whole queue and is cheap: draft status, age, size, files touched,
subsystems, existing reviews. Enough to rank, to exclude drafts, and to show the shape
of the queue before spending anything.

Evidence for the split: the source session deep-analysed all twelve PRs up front and
discarded four — two were drafts never reviewed, two were fully analysed before the
skip-drafts rule existed. A single boolean would have filtered them.

**Deep analysis** is on demand and pipelined:

- Analyse the first PR the user picks up immediately.
- Keep **2 PRs ahead** analysed in the background.
- **Abandon** a prefetched analysis if the user reorders past it.
- **Re-validate against head SHA** before review — a day may have passed.

**Fan out to parallel sub-agents when a PR exceeds 15 files or 1,500 changed lines.**
Otherwise single-agent.

Known false positive: a doc-heavy diff (the source session's #978 — 15 files, ~2k lines,
of which 1,875 were bulk prompt-file deletion with no product risk) will over-invest.
Accepted rather than complicating the threshold.

Fan-out earns its cost. On the 88-file theming PR, four independent investigations found
what a single pass missed: a `Location`-vs-`CodeBase` distinction, ten scanner evasion
holes proven by *running* the scanner, and a 26-of-29 locale gap.

## 7. Reviewing a PR

**Open the PR in the browser, brief orientation, then findings one at a time.**

Every review begins by opening the PR page in the default browser. Sometimes that is
the best orientation available — the rendered diff, the conversation, the CI status and
the author's own framing, all in the form the author sees them.

Linked issue trackers are deliberately **not** opened: the PR carries the link where it
matters, and bug trackers differ across repos.

Orientation is ~15 lines: size, complexity, **what is genuinely good about the change**,
and the finding count. The full analysis stays on disk and is available on request.

Evidence: the source session presented complete analyses up front and the user
reported losing the thread on the last PR. Every decision was made from per-finding
evidence, not from the up-front document. But "what is good" must survive into the
orientation — it shaped how several reviews opened and is what makes a ten-finding
review read as calibration rather than a pile-on.

Each finding is presented with its evidence, its anchor, and **a recommended
disposition**. The user rules. The skill does not batch.

### Hard rules

1. **Verify before presenting.** Any finding proposed as a blocker or a major finding
   must have its central claim verified against source first. Questions and nits are
   exempt.

   This is the highest-value rule in the spec. In the source session three findings
   were materially wrong and each was caught only by reading the code: a claim that a
   copyright string fed RAMP archive metadata (the caller never reads it), a claim that
   a menu handler was dead (it works via an else-branch), and a claim that an unreviewed
   dark theme shipped to every user (the surface is fail-closed and opt-in). All three
   were headline blockers.

2. **Never post without explicit approval.**
3. **Verify the post landed afterwards.**
4. **Never name a withdrawn claim in the posted body.** The author never saw the draft;
   naming what was withdrawn is noise at best and confusing at worst.

### Guidance

- Prefer running a tool to reading it. Executing a repo's own scanner against a fixture
  settled a question that reading it could not.
- Frame an unverified mechanism as "this needs demonstrating, and here is why the
  existing evidence cannot settle it" rather than as an assertion of fact.
- Credit what is genuinely good, specifically, before the findings.

## 8. Verdicts

Five, with the redesign/changes split retained:

| Verdict | GitHub action |
|---|---|
| approve | APPROVE |
| nit pick | **APPROVE with comments** — unblocks the author, which is the intent of a nit |
| request changes | REQUEST_CHANGES |
| request redesign | REQUEST_CHANGES |
| reject | COMMENT, then close |

`request redesign` and `request changes` are indistinguishable to GitHub but not in the
ledger, where "did I ask for fixes or for a reconsideration?" is worth knowing later.

`approve` is included because a queue that can only request changes or reject is not a
review queue — its absence in the source session was a property of that sample, not of
reviewing.

## 9. Posting

**One review call: inline comments plus a summary body.**

- **Anchor every finding that can be anchored**, to the nearest changed line the finding
  is *about*. GitHub only permits comments on files in the diff — and many of the source
  session's most valuable findings concerned untouched files (a nested restore in an
  unmodified `.targets`, a crash mechanism in a file the PR did not open). Those anchor
  to the changed line that reaches them, with the chain explained in the comment.
- **The summary body is always present, brief, and free-form.** It carries only what the
  inline comments do not — scope objections, verification demands, cross-cutting design
  push-back — plus anything with no anchor at all.
- **The attribution trailer goes on the summary only**, never on inline comments.

**Attribution is a per-repo setting, default on.** The model name comes from the live
session, never hardcoded. The source session disclosed on three of ten reviews because
the request came mid-way; partial disclosure reads worse than either consistent choice.

## 10. Completion and the response loop

A review is **done when the post is verified as landed**. The session ends there for
that PR until the author responds.

When an author responds — commits **or** replies — the PR is promoted to tier 1a. On
pickup:

- **Map each prior finding against the delta**, showing: the finding, the author's
  response, and **the diff that proves it**.
- **Re-analyse automatically when the delta exceeds 100 changed lines.** Below that, ask
  before doing anything automatic, including drafting a second round.
- The mapping is a claim, not a verdict — **hard rule 1 still applies**, so "addressed"
  on a blocker is verified against code, not inferred from a commit message.
- A reply that pushes back rather than changing code is still a response: surface the
  argument, do not go looking for a diff that is not there.

## 11. Repo conventions

**Hybrid.**

- **Discover** at review time: `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, repo skills.
  Stays current automatically.
- **Accumulate** in `review-focus.md` per repo — only what discovery cannot give:
  which findings recur, and which rules keep getting broken.

**The skill proposes additions when it notices a repeat finding.**

For FieldWorks the initial file would record: comment-hygiene violations appeared in
nearly every PR; PR bodies routinely overstate what was verified; tests that construct
WinForms controls; `-CommentHygiene` skipped or unverified.

Note the split in kind. "Comments run 3-4 sentences" and "the word *gate* is banned in
new prose" are the user's cross-repo preferences and belong in Claude memory.
"net48 means C# 7.3" and "registration-free COM" are facts about FieldWorks and belong
in `review-focus.md`.

## 12. Follow-ups

Follow-ups arising from a review that belong to the **user** rather than the author —
"message the author about the PR you closed", "retire these two dead targets" — are
**surfaced at the end of the session and then dropped.** They are output, not state.
The skill does not become a to-do list.

---

## Open items

None. Every branch was decided in the grilling pass.

## Not yet decided (implementation, not design)

- Exact file formats for the ledger and caches.
- Whether triage output is a table, and how much of it.
- Cache invalidation policy for the repo-path and contributor-repo maps.
