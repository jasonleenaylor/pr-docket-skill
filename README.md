# pr-docket

A personal Claude Code skill that keeps a ranked, cross-repository docket of pull
requests waiting on you and drives one review at a time to a posted verdict.

Two skills ship here:

- `pr-docket/` — the docket: ranking, analysis, finding-by-finding decisions, posting,
  and the author-response loop.
- `respond-to-pr-review/` — answering feedback on your own PR. `pr-docket` hands off to
  it for tier 1b, so install both.

## Install

Clone, then link (or copy) each skill into your personal skills directory:

```bash
git clone https://github.com/jasonleenaylor/pr-docket-skill ~/pr-docket-skill
ln -s ~/pr-docket-skill/pr-docket ~/.claude/skills/pr-docket
ln -s ~/pr-docket-skill/respond-to-pr-review ~/.claude/skills/respond-to-pr-review
```

On Windows without symlink rights, copy the two directories into
`%USERPROFILE%\.claude\skills\` instead.

## Make it yours

The ranking is opinionated about which repositories matter. Every such spot is marked:

```bash
grep -rn PERSONAL pr-docket/
```

Replace the tier 4 repo order, the priority-label scheme, the table of repos that carry
those labels, the repo that runs a review bot on every PR, and the clone root with your
own before the first run.

## Where it keeps state

`~/.claude/reviews/`, created on first run. Caches, per-repo notes, and per-PR ledgers
live there. Nothing under it belongs in this repository.

## Run

```
/pr-docket list     the docket, read-only
/pr-docket          resume an in-progress review, else show the docket
/pr-docket next     take the top item
/pr-docket <text>   resolve to one PR and start it
```

`pr-docket/DESIGN.md` records why each rule is the way it is.
