---
name: pr-docket
description: Maintain a ranked cross-repository pull-request review docket and drive one review at a time to a posted verdict. Use when the user runs /pr-docket, asks which PRs need their attention, wants to pick up or resume reviewing someone else's pull request, or names a specific PR to review. Handles docket ranking, deep analysis, finding-by-finding decisions, inline posting, and the author-response loop. For reviewing local uncommitted changes, use the `code-review` skill instead.
argument-hint: "[next | list | <PR reference or fuzzy description>]"
user-invocable: true
---

# PR docket

The matters awaiting your judgment: a personal, cross-repository docket of pull requests,
ranked by who is waiting on you, worked one at a time, resumable across days.

Ranks what needs attention, drives one review to a posted verdict, and keeps enough
state that a review can be abandoned mid-stream and picked up later.

This is about other people's pull requests. For reviewing local changes, use `code-review`.

Design rationale lives in `DESIGN.md`. Read it when a decision here seems arbitrary —
most were made from evidence and the reasoning is recorded there.

## Invocation

| Input | Behaviour |
|---|---|
| *(empty)* | Resume an in-progress review. If none, show the ranked docket. |
| `next` | Take the top of the docket and start it. |
| `list` | Print the docket as addressable identifiers. |
| anything else | Resolve to one PR and start it. |

Never auto-advance. After a review is posted, stop and wait.

**Resolution** (`/pr-docket FieldWorks PR about toothpaste`): match title, branch and author
first; fall back to body only if that yields nothing. Disambiguate the repo before
touching PRs. **Always confirm the resolved PR before starting**, even on a single strong
match. On no clear match, list candidates in `list` form.

There is no author filter. Tier membership is defined by the rules below, not by whose
PRs they are.

## Hard rules

These are not guidance. Violating them produces the failures this skill exists to avoid.

1. **Verify before presenting.** Any finding proposed as a blocker or a major finding
   must have its central claim checked against source first. Questions and nits are
   exempt. Analysis output is a hypothesis until read against the code.
2. **Never post without explicit approval.** Show the full body, wait for a decision.
3. **Verify the post landed.** Re-read the review after posting and confirm state.
4. **Never name a withdrawn claim in the posted body.** The author never saw the draft.
5. **Confirm the PR is still open in the same step that posts.** Read `state` immediately
   before the call, not at the start of the session. A long analysis is exactly when a PR
   merges underneath it; two comments have already landed on an hour-old merge that way.

Rule 1 is the important one. See `DESIGN.md` for three real cases where a headline
blocker was materially wrong and only reading the code caught it.

**Rule 1 covers counts, and this is where it is broken most often.** A number in a posted
comment -- "52 of 60 files", "it touches two of their files", "25 assertions" -- is a
claim like any other, and one the author can check in seconds. Recompute every count at
the head being reviewed, immediately before writing it. Never carry one forward from an
earlier round, from analysis output, or from the PR body.

Three went out wrong in one week, all the same shape: a BOM ratio quoted as 52 of 60 when
the directory held about 92 files, a file count given as two when it was four, and an
assertion count of 72 that had since become 62. Authors corrected two of the three. None
changed a conclusion, which is exactly why the temptation is to skip the check -- and why
being wrong about them is expensive out of proportion to the stake: a reviewer who
miscounts what is in front of them invites the author to discount the findings that took
real work.

If a count is not worth recomputing, it is not worth putting in the comment. Prefer the
claim without the number.

## Queue

Build with `gh`, never by scanning the filesystem. See `references/queue.md` for the
exact queries, caches, and ranking.

Five tiers, `0` to `4`, drafts excluded unless `--include-drafts`. Tier 1 has two halves,
`1a` and `1b`; "tier 1" on its own means both together.

| Tier | Contents |
|---|---|
| **0** | **Your own PR** is approved, green and mergeable -- nothing left but the button |
| **1a** | You requested changes, the author has since responded (commits **or** replies) |
| **1b** | **Your own PR** has feedback you have not answered |
| **2** | Requested reviewer, no *human* has reviewed |
| **3** | Requested reviewer, another human reviewed, you have not engaged; or any PR that @-mentions you |
| **4** | Open, no human reviews, in admin+contributor repos |

**Review bots are not reviewers.** CodeRabbit, Copilot and their kind comment on
everything in the repos that run them; counting them would empty tier 2 and tell you a
human had looked when none had. Detection and the extension point for bots that run under
ordinary user accounts are in `references/queue.md`. The single exception is tier 1b: a
bot that blocks *your own* PR is still a loop waiting on you.

**Tier 0 is above tier 1, because it is the cheapest loop on the board.** An approved,
green, mergeable PR of your own needs one click, and until it lands the work is done and
delivering nothing -- the branch rots, the base moves under it, and a reviewer who spent
real effort has nothing to show for it. Everything below tier 0 costs you reading and
judgment; this costs a merge.

Tier 1 comes next because **closing loops beats opening them.** Both halves are a loop
waiting on you: one is a review you already invested in, the other is your own work
blocked on your reply.

**Tier 1a and 1b interleave, ordered by how long the response has been waiting.** Age is
the honest measure of who has waited longest; splitting them into sub-tiers would let a
two-day-old comment on your PR outrank a two-week-old response on someone else's. Each
docket line says which side it is, so you can still pick deliberately.

**A PR that @-mentions you is promoted to tier 3**, unless a tier already claims it —
somebody typed your name, which is a person asking. No effect where you were already asked
to review (tiers 1-3); a tier 4 sweep item moves up one; and a PR no tier surfaces at all —
someone else reviewing who wants your opinion — enters at tier 3, which is the case the
rule exists for. The docket line says `(mentioned)`, because the search cannot tell a
request for help from a thank-you.

<!-- PERSONAL: the label names below are the sillsdev convention. If your repos use a
different priority-label scheme, substitute it here and in references/queue.md; the
two-unlabelled-bands logic still applies. -->
**Within every tier, the priority label sorts first.** Some repos label a PR `🟥High`,
`🟨Medium` or `🟩Low`. That is the author's own judgment of what matters, and it outranks
age. Five bands, in order:

| Band | What lands here |
|---|---|
| 1 | `🟥High` |
| 2 | **unlabelled, in a repo with no priority labels** |
| 3 | `🟨Medium` |
| 4 | **unlabelled, in a repo that has the labels** |
| 5 | `🟩Low`, then `🟪Idea` |

The two unlabelled bands are the point. A repo that never labels anything says nothing by
omission, so its PRs sit between High and Medium rather than sinking to the bottom. A repo
that does label says something real by leaving one blank, so those sit between Medium and
Low. Do not collapse the two into one "untagged".

Within a band the tier's own rule breaks the tie: age for tiers 1a-3, repo priority then
age for tier 4. `references/queue.md` lists which repos carry the labels and how to read
them.

**"Refresh the docket" means every tier, and tier 4 is the one that gets skipped.**
It is the only tier that needs a per-repo sweep rather than one search, so it is the
expensive one and the tempting one to leave out — and it is also the largest, and the
only route for a PR that names no reviewer. Skipping it does not look like an error: the
docket still prints, just without the work. A refresh that has not run
`--review none` across the contributor repos is not a refresh; say so rather than
presenting a partial docket as the docket.

This has already cost once: a FieldWorks PR requesting no reviewers was invisible to a
refresh that ran tiers 1a, 1b, 2, 3 and mentions, and the sweep would have surfaced nine
tier 4 items in that repo alone.

**Tier 1b hands off.** Reviewing someone else's code and answering feedback on your own
are different jobs. When the user takes a 1b item, invoke the `respond-to-pr-review`
skill and let it drive. `pr-docket` owns the docket — detecting, ranking, surfacing —
not the author-side workflow.

**Tier 0 hands back.** There is no review to run, so do not start one. Say the PR is
ready, say what approved it, and offer to merge. Merging is the user's call every time:
it is outward-facing and hard to reverse, and a green PR is not always one they want in
the base branch today. Never merge unasked, and never squash-vs-merge by your own
preference -- ask which, or follow a repo convention you can point at.

## Reviewing one PR

### 1. Open and orient

Open the PR page in the default browser first (`open` on macOS, `xdg-open` on Linux):

```powershell
Start-Process "https://github.com/<owner>/<repo>/pull/<n>"
```

Do not open linked issue trackers — the PR carries the link where it matters, and
trackers differ across repos.

**Read the existing review threads before analysing, on any PR where someone else has
already reviewed.** On tier 3 and mention-promoted items you are joining a conversation,
not opening one. Findings another reviewer has already raised — and the author's responses
to them — change what is worth saying, and restating them as new analysis wastes the
author's time and misrepresents whose work it was. Bot reviewers count: their findings
reach the PR through whoever quoted them. Note what is already covered before spending
anything on fresh analysis.

```bash
gh api repos/<owner>/<repo>/pulls/<n>/comments --paginate   # inline threads, with in_reply_to_id
gh api repos/<owner>/<repo>/issues/<n>/comments --paginate  # general comments
```

Where the user was **mentioned** rather than asked to review, the job is usually a reply
in the thread that named them, not a review. Say so, and hold the bar high: one comment
that is new and certain beats six that re-tread the thread.

Then give a **brief orientation, about fifteen lines**: size, complexity, what is
genuinely good about the change, and how many findings there are. Keep the full analysis
on disk; offer it rather than pasting it.

Naming what is good is not politeness. It is what makes a ten-finding review read as
calibration rather than a pile-on.

### 2. Findings, one at a time

For each finding, present:

- the claim, with file:line evidence
- where it would anchor
- **a recommended disposition**, with reasoning

Then stop and wait. Do not batch. Do not present the next finding until this one is
ruled on.

Record each disposition as it is made: `required`, `question`, or `dropped`, each with a
one-line reason. The dropped ones matter most — they are the only part of this record
that exists nowhere else, and they stop the same argument recurring next time.

### 3. Draft, approve, post

Compose the body from the decisions. If a persisted body already exists and has been
edited, **diff the new one against it and show what would change to the user's prose
before writing.** Never silently replace their words.

Show the complete body. Get explicit approval. Then post, then verify.

See `references/posting.md` for inline anchoring and the API call.

### 4. Done

A review is finished when the post is verified as landed. Stop there for that PR until
the author responds.

At the end of a working session, surface any follow-ups that belong to the user rather
than the author — then drop them. This skill is not a to-do list.

## Verdicts

| Verdict | GitHub event |
|---|---|
| approve | `APPROVE` |
| nit pick | `APPROVE` with comments — unblocks the author, which is the point of a nit |
| request changes | `REQUEST_CHANGES` |
| request redesign | `REQUEST_CHANGES` |
| reject | `COMMENT`, then close |

`request redesign` and `request changes` are identical to GitHub but distinct in the
ledger: "did I ask for fixes, or for a reconsideration?" is worth knowing later.

## Analysis

**Triage the whole queue cheaply** — draft status, age, size, files, subsystems,
existing reviews. Enough to rank and to exclude drafts before spending anything.

**Deep-analyse on demand**, pipelined:

- Analyse the first PR the user picks up immediately.
- Keep **two ahead** analysed in the background.
- Abandon a prefetched analysis if the user reorders past it.
- **Re-validate against head SHA** before reviewing — a day may have passed.

**Fan out to parallel sub-agents above 15 files or 1,500 changed lines.** Otherwise one
agent. A doc-heavy diff will occasionally trip this and over-invest; that is accepted.

Prefer running a tool to reading it. Executing a repo's own checker against a fixture
settles questions that reading it cannot.

## Author responses (tier 1a)

When a PR returns:

- **Open the PR page in the browser first**, the same as for a fresh review. The author's
  replies and new commits are best read in the form the author sees them.
- **Map each prior finding against the delta.** Show the finding, the author's response,
  and **the diff that proves it**.
- **Re-analyse automatically when the delta exceeds 100 changed lines.** Below that, ask
  before doing anything automatic, including drafting a second round.
- The mapping is a claim. **Hard rule 1 still applies** — "addressed" on a blocker is
  verified against code, not inferred from a commit message.
- A reply that pushes back instead of changing code is still a response. Surface the
  argument; do not go looking for a diff that is not there.

## Repo conventions

Discover at review time: `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, repo skills.

Accumulate in `~/.claude/reviews/<owner>-<repo>/review-focus.md` only what discovery
cannot give — which findings recur, which rules keep getting broken. **Propose an
addition when a finding repeats**; do not write to it silently.

Keep the user's cross-repo preferences in Claude memory, not here. "Comments run 3-4
sentences" is a preference; "net48 means C# 7.3" is a repo fact.

## State

Central, keyed by `nameWithOwner`. Never path-derived — reviews are routinely run from
git worktrees.

```
~/.claude/reviews/
  index.md                      cross-repo rollup
  repo-path-map.json            nameWithOwner -> local path
  contributor-repos.json        admin+contributor set
  <owner>-<repo>/
    settings.json               per-repo settings (attribution)
    review-focus.md
    prs/<n>.md                  ledger + decisions
    prs/<n>-body.md             persisted draft body
```

Record the **head SHA at review time** — it drives tier 1a promotion and staleness
detection.

**Prune the whole record when a PR closes or merges.** No exceptions.

Formats in `references/state.md`.
