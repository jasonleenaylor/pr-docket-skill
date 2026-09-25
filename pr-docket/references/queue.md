# Queue construction

<!-- PERSONAL: `C:\Repositories` is the original author's clone root. Substitute yours
here and under `repo-path-map.json` below. -->
The queue is built from GitHub, never by scanning the filesystem. `C:\Repositories` holds
~80 repos at depth 1 and more at depth 2; walking them is slow, misses uncloned repos,
and inverts the relationship — a review request exists on GitHub whether or not anything
is cloned locally.

The local checkout is needed only when code must be read.

## Caches

Both are cheap to rebuild. Refresh on a miss, on demand, or when older than a few days.

### `repo-path-map.json` — `nameWithOwner` to local path

Repos live at more than one depth. Build by walking `C:\Repositories` one and two levels
deep, taking directories containing `.git`, and reading each one's origin remote:

```powershell
git -C <dir> remote get-url origin
```

Normalise to `owner/repo`. A worktree resolves to its main checkout, which is correct —
state is keyed by repo identity, not by working directory.

### `contributor-repos.json` — admin AND contributor

```bash
# admin (large — hundreds)
gh api "user/repos?affiliation=owner,collaborator,organization_member&per_page=100" \
  --paginate --jq '.[] | select(.permissions.admin==true) | .full_name'

# contributor proxy (small — one call)
gh search prs --author=@me --limit 100 --json repository \
  --jq '.[].repository.nameWithOwner'
```

Intersect. Authored-PRs is a **proxy** for contributor and will miss a repo the user has
only pushed to directly. Accepted: it is one call instead of hundreds.

## What counts as a review

Two kinds of `reviews` entry are not somebody reviewing the PR, and both inflate the
tier if counted.

**A bot review is not a review.** Review bots comment on nearly every PR in the repos that
run them, so counting them as reviewers would push every such PR out of tier 2 and make
the docket claim a human has looked when none has.

Treat a review as a bot's when **either** holds:

- `.user.type == "Bot"` — GitHub Apps (Copilot, dependabot, most review bots)
- the login ends in `[bot]` — `coderabbitai[bot]`, `github-actions[bot]`

**Nor is the author replying to one.** GitHub creates a `COMMENTED` review whenever
anyone replies in a review thread, so an author answering a bot appears in `reviews`
under their own human login. Filtering only bots leaves those behind and the PR still
reads as tier 3 — not hypothetical, it is exactly what
`sillsdev/interlinearizer-extension#283` looked like. Exclude the PR's own author too.

```bash
export PR_AUTHOR=$(gh pr view <n> --repo <owner>/<repo> --json author --jq .author.login)
gh api repos/<owner>/<repo>/pulls/<n>/reviews \
  --jq '[.[] | select(.user.type != "Bot"
                      and (.user.login | endswith("[bot]") | not)
                      and .user.login != env.PR_AUTHOR)] | length'
```

`gh --jq` is gojq, which takes no `--arg`, so the author login travels through `env`.
That form is verified working and needs no standalone `jq`.

Some bots run under an ordinary user account and pass both tests. Add those here as they
are identified — this list is the extension point, and it is deliberately empty until a
real one turns up:

| login | repos | notes |
|---|---|---|
| *(none yet)* | | |

**The one exception is tier 1b**, and it is not an exception to "bots are not reviewers" —
it is a different question. There the test is whether a loop is waiting on you, and a bot
that blocks your own PR is a loop whatever wrote it. See tier 1b below.

## Tier queries

Drafts are excluded unless `--include-drafts`. Every query filters `isDraft == false`.

### Tiers 2 and 3 — review requested

```bash
gh search prs --review-requested=@me --state=open \
  --json repository,number,title,author,createdAt,isDraft --limit 100
```

Split by whether anyone has reviewed:

```bash
gh api repos/<owner>/<repo>/pulls/<n>/reviews --jq 'length'
```

Count **human** reviews only (see Bot reviewers above):

- zero human reviews -> **tier 2**, however many bots have commented and however many
  times the author has replied to them
- some human reviews, none authored by the user -> **tier 3**
- a review authored by the user -> not tiers 2-3; it is tier 1a or already handled

**Skip a PR whose local record says `status: declined`, in every tier including tier 1.**
That status means the user has decided not to engage further — either they read it and
chose not to comment, or they engaged, closed their threads out and are done. Both are
decisions, not omissions. Nothing on GitHub records either one: a review request stays
open, and a `posted` record keeps promoting to tier 1a on every push the author makes. So
without this check the docket re-offers the PR forever and the analysis is spent again
each pass. Say so once when it is skipped rather than
hiding it:

```
(skipped: interlinearizer-extension#283, closed without comment 2026-09-04)
```

Declining is not withdrawing as a reviewer. The user is still a requested reviewer and the
author may still be waiting; whether to say so on the PR is the user's call, never an
automatic one.

Say so on the docket line when a bot has reviewed and no human has, so tier 2 does not
read as "nobody has looked at this at all":

```
tier 2   sillsdev/interlinearizer-extension#283   Add Paratext 9 interlinear test projects   (bot-reviewed only)
```

### Tier 1a — the user reviewed, the author responded

Candidates come from local state: every PR with status `posted`. For each, compare
against the recorded head SHA and review timestamp.

```bash
gh pr view <n> --repo <owner>/<repo> \
  --json headRefOid,state,isDraft,updatedAt,commits
```

A PR is tier 1a when **either** holds:

- `headRefOid` differs from the recorded head SHA — new commits
- a comment or review by anyone other than the user postdates the recorded review
  timestamp — a reply

Check all three comment sources; a push-back often arrives as an inline reply:

```bash
gh api repos/<owner>/<repo>/issues/<n>/comments --paginate
gh api repos/<owner>/<repo>/pulls/<n>/comments --paginate
gh api repos/<owner>/<repo>/pulls/<n>/reviews --paginate
```

`updatedAt` matching the recorded review timestamp to the second is a reliable signal
that **nothing** has happened.

### Tier 0 — your own PR, ready to merge

Same search as tier 1b; the split is one extra field per PR:

```bash
gh pr view <n> --repo <owner>/<repo> \
  --json mergeable,mergeStateStatus,reviewDecision,isDraft
```

A PR is tier 0 when **all** hold:

- it is yours and not a draft
- `reviewDecision == "APPROVED"`
- `mergeable == "MERGEABLE"` — no conflicts
- `mergeStateStatus == "CLEAN"`

`CLEAN` is the whole test for "green". It is the one merge state that is unambiguous:
required checks passed, required approvals in, base not behind. `BLOCKED` is the state to
resist reading — it covers a failing check, a missing approval and an unsatisfied
protection rule alike, so a PR that is merely unapproved looks identical to one whose
build is red.

**A bot approval does not make tier 0**, and this needs no special-casing: `reviewDecision`
counts only reviews from users who can approve, so a Copilot or CodeRabbit pass never sets
it to `APPROVED`. What does need care is the reverse — a human `APPROVED` plus a bot
still requesting changes leaves the state `BLOCKED`, not `CLEAN`, so the PR correctly
stays out of tier 0 until that thread is resolved.

**Tier 0 wins over tier 1b when a PR qualifies for both.** An approved, green PR with an
unanswered comment on it is still one click from done; answering the comment can happen
in the same breath, and the docket line says both.

Order within the tier by **how long it has been ready** — the newest approving review's
`submitted_at`, oldest first. Not PR age: a PR opened in March and approved this morning
has been mergeable for an hour.

```bash
gh api repos/<owner>/<repo>/pulls/<n>/reviews \
  --jq '[.[] | select(.state == "APPROVED")] | max_by(.submitted_at) | .submitted_at'
```

**The failing-checks demotion cannot apply here** — `CLEAN` already means the checks
passed. A PR that goes red after landing in tier 0 simply stops being `CLEAN` and falls
back to tier 1b or off the docket.

### Tier 1b — your own PR, feedback unanswered

```bash
gh search prs --author=@me --state=open \
  --json repository,number,title,createdAt,isDraft --limit 100
```

For each, a PR qualifies when **the newest human activity on it is not yours** — a
review, an inline review comment, or an issue comment postdating your last commit or
comment. Check all three sources; a reviewer's push-back often arrives as an inline
reply rather than a formal review.

Bots count as human activity for this purpose when they request changes — a Copilot
review that blocks the PR is still a loop waiting on you.

Interleave tiers 1a and 1b by **how long the response has been waiting**, not by which
kind they are. Label each line so the user can see which side they are on.

**Hand off tier 1b** to the `respond-to-pr-review` skill. `pr-docket` detects, ranks and
surfaces; it does not run the author-side workflow.

### Mentions — somebody asked for the user by name

```bash
gh search prs --mentions=@me --state=open --json repository,number --limit 100
```

**A mention promotes a PR to tier 3 — never past a tier it already holds.** Concretely:

- already tier 1a, 2 or 3 -> **no change.** A review request already outranks a mention,
  and the user asked for it to have no effect where they have been asked to review.
- tier 4 -> **tier 3.** One tier up, which is what a person asking is worth over a sweep.
- **in no tier at all -> tier 3.** This is the case worth having the rule for. A PR where
  somebody else is reviewing and wants a second opinion has human reviews on it and no
  request to the user, so tiers 2-4 all miss it and the docket never shows it. Tier 3 is
  where it belongs on the definitions already in use: others reviewed, the user has not
  engaged.

Not the user's own PRs — tier 1b covers those. Drafts stay excluded. **Repo scope does not
apply**: a mention is a person asking whatever repo it came from, so do not filter this to
the contributor set the way tier 4 is filtered.

Sort by age within tier 3 like anything else; a mention buys the tier, not a jump inside it.

Mark the line, because the reason for the placement is not visible from the tier alone:

```
tier 3   paranext/paranext-core#2763   perf(dev): cut dev startup   (mentioned)
```

**The marker is a prompt to look, not a verdict.** The search cannot say who wrote the
mention or whether it asks for anything: "thanks @user for catching that" matches exactly
as "@user what do you think?" does, and a bot that @-mentions the user matches too. Do not
spend calls resolving that during queue construction — surface it, and when offering the
PR say what the mention actually was if it turns out to be a credit rather than a question.

### Tier 4 — unreviewed, in admin+contributor repos

For each repo in `contributor-repos.json`, in priority order. **Use the search API's
`--review none` filter — one call per repo:**

```bash
gh search prs --repo <owner>/<repo> --state open --review none \
  --json number,title,author,createdAt,isDraft --limit 50
```

**Do not** list PRs and then call `pulls/<n>/reviews` once per PR. That is a round-trip
per open PR and times out on a busy repo — `paranext/paranext-core` alone has 33 open
unreviewed PRs, and the per-PR form exceeded two minutes across five repos on the first
live run.

`--review none` also excludes PRs the user has already reviewed, so tier 4 needs no
cross-check against local state to avoid re-offering finished work.

**`--review none` counts bot reviews, and this is the one place that costs coverage.**
A PR that only a review bot has touched is not `review:none` to the search API, so it
never reaches tier 4 — the opposite of the tier 2-3 rule above, and a silent gap rather
than a visible one. GitHub offers no "no human reviews" qualifier, so the only fix is the
per-PR walk this section rejects for busy repos.

<!-- PERSONAL: the per-PR walk is needed only in repos that run a review bot on every
PR. In the original author's set that is `sillsdev/interlinearizer-extension` (CodeRabbit).
List yours here. -->
Resolve it per repo rather than globally: for repos that run a review bot on everything
(`sillsdev/interlinearizer-extension` runs CodeRabbit), list open PRs and filter by human
reviews, still capped at three. For the rest, `--review none` stays correct and cheap,
because a PR with no bot on it and no human reviews is genuinely `review:none`.

Tier 4 is the sweep; there is no separate verb and no author filter.

## Ranking

| Tier | Order within tier |
|---|---|
| 0 | Oldest approval first. **The priority band does not apply** -- there is no work to schedule, so the author's own guess at importance says nothing about which button to press first |
| 1a, 1b | One interleaved list: priority band, then oldest response first |
| 2 | Priority band, then oldest first |
| 3 | Priority band, then oldest first |
| 4 | Priority band, then repo priority, then oldest first |

## Priority labels

<!-- PERSONAL: `🟥High` / `🟨Medium` / `🟩Low` / `🟪Idea` is the sillsdev label scheme.
Substitute your repos' priority labels; keep the five-band structure. -->
Some repos label a PR `🟥High`, `🟨Medium` or `🟩Low` (and `🟪Idea`). **The band sorts
before age in every tier.** Five bands:

| Band | What lands here |
|---|---|
| 1 | `🟥High` |
| 2 | unlabelled, in a repo with **no** priority labels |
| 3 | `🟨Medium` |
| 4 | unlabelled, in a repo that **has** them |
| 5 | `🟩Low`, then `🟪Idea` |

Bands 2 and 4 are deliberately different. Silence in a repo that never labels carries no
information, so those PRs sit between High and Medium. Silence in a repo that does label is
a signal, so those sit between Medium and Low.

<!-- PERSONAL: this table is the original author's contributor set. Rebuild it for your
own repos with the `gh label list` command below; it decides band 2 versus band 4. -->
**Repos carrying the labels**, verified 2026-09-18 by `gh label list`:

| Repo | Has them |
|---|---|
| `sillsdev/TheCombine` | yes |
| `sillsdev/liblcm` | yes |
| `sillsdev/python-sil-lift` | yes |
| `sillsdev/interlinearizer-extension` | yes |
| FieldWorks, libpalaso, chorus, flexbridge, genericinstaller, fw-nunitreport-action, fieldworks-analytics-reporting, ServeReedley, paranext-core | no |

Re-check with `gh label list --repo <owner>/<repo> --limit 200 --json name` when a repo
joins the docket; the set changes slowly, so cache it with the contributor set.

Labels come back from the same search that builds the queue -- add `labels` to `--json` and
read `[.labels[].name]`. No extra call:

```bash
gh search prs --review-requested=@me --state=open \
  --json repository,number,title,author,createdAt,isDraft,labels --limit 100
```

`gh pr list` and `gh pr view` expose `labels` the same way, so the tier 4 sweep and the
tier 1a checks need no extra round trip either.

**Say the band on the docket line** when a PR carries a label, so the order is legible
rather than mysterious:

```
tier 2   sillsdev/TheCombine#4371   Wait for the backend before restoring   (🟨Medium)
tier 2   sillsdev/TheCombine#4372   Bump Node.js from 22 to 24 LTS          (🟩Low)
```

Do not print a band for an unlabelled PR. Bands 2 and 4 are inferred from the repo, and
printing "(unlabelled)" on every line is noise.

**A PR that @-mentions the user is promoted to tier 3, unless a tier already claims it.**
Somebody typed their name, which is a person asking. See "Mentions" below for what that
means and why tier 3 is where it lands.

**A PR whose required checks are failing drops one tier.** A red build is work the author
already owns, so it does not deserve the next slot ahead of a PR that is genuinely waiting
on you -- and a review written against a broken build risks chasing failures that are not
the reviewer's to find. Check with `gh pr checks <n>` when ranking; only `fail` demotes,
`pending` does not. Say so on the docket line, so the drop is visible rather than silent:

```
tier 2   sillsdev/FieldWorks#1105   LT-22728: Scope local libraries    (dropped from tier 1a: build failing)
```

The two rules compose in one direction: a demotion moves the tier, then the mention floats
the PR to the top of the tier it landed in. A mention never cancels a demotion.

Check this before offering a PR, not after. `mergeStateStatus` alone will not tell you --
`BLOCKED` is also what a required-review gate looks like, so it reads the same whether the
build is red or merely unapproved.

**Repo priority applies to tier 4 only.** A direct review request is a person waiting,
regardless of which repo it came from.

<!-- PERSONAL: this is the original author's ranking of the repos they most want swept.
Replace items 1-5 with your own; keep item 6 as the fallback for everything else. The
same list appears in DESIGN.md section 3. -->
Tier 4 repo order:

1. `sillsdev/FieldWorks`
2. `sillsdev/liblcm`
3. `sillsdev/interlinearizer-extension`
4. `sillsdev/flexbridge`
5. `sillsdev/chorus`
6. everything else, **fewest contributors first**

Fewest-contributors-first means nobody else is likely to review it, which is where
discretionary attention is worth most.

**Cap tier 4 at three PRs per repo.** Show the oldest three, then a count of the
remainder:

```
paranext/paranext-core#2432   2026-06-18  Layout direction switcher shift click ...
paranext/paranext-core#2437   2026-06-18  docs: shared mental model for settings ...
paranext/paranext-core#2461   2026-06-25  docs: verse-aligned multi-translation ...
                              ... and 30 more unreviewed
```

Without the cap a single busy repo swamps the docket — `paranext/paranext-core` alone
has 33 open unreviewed PRs, mostly dependabot bumps and docs commits, and they would
bury everything ranked below them. The cap applies to tier 4 only; tiers 1-3 are
never truncated, because a person is waiting on each of those.

Never truncate silently. The "and N more" line is what makes the cap honest rather
than a hidden filter.

Contributor counts:

```bash
gh api repos/<owner>/<repo>/contributors?per_page=1 -i --jq 'empty'
# read the Link header's last page number; or count a small page directly
```

Cache these with the contributor-repo set; they change slowly.

## Output form

`list` prints addressable identifiers, because they are what the fuzzy resolver and the
user both consume:

```
tier 0   sillsdev/FieldWorks#1149   LT-22788: Parse-words-in-text submenu          (approved 1h ago, mergeable)
tier 1a  sillsdev/FieldWorks#1105   LT-22728: Scope local libraries to one build   (responded 2h ago)
tier 1b  sillsdev/liblcm#400        Drop the legacy writing-system cache           (reviewer replied 1d ago)
tier 2   sillsdev/liblcm#412        Fix writing system fallback                    (opened 3d ago)
tier 4   sillsdev/chorus#89         Bump dependencies                              (opened 11d ago)
tier 4   paranext/paranext-core     ... and 30 more unreviewed
```

Tier 4 is capped at three per repo (see Ranking). Every docket line, here and wherever
one is printed, starts with `tier <name>` using the names from the tier table. Show the
tier, the identifier, a truncated title, and the age or response time. Nothing else —
this is a menu, not a report.
