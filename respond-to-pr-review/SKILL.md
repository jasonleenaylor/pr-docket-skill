---
name: respond-to-pr-review
description: Respond to review feedback on your own pull request — evaluate each comment, fix what is sound, dispute what is wrong, ask about what is ambiguous, verify, commit, reply in-thread, and report. Use when handling reviewer comments, requested changes, Copilot feedback, or unresolved threads on a PR you authored, in any repository. Invoked directly or handed off from pr-docket.
argument-hint: "PR reference, or empty to use the current branch's PR"
user-invocable: true
---

# Respond to PR review

Feedback on your own pull request, worked one comment at a time.

Starts from technical evaluation, not agreement. A reviewer's comment is input to check
against the code, not an instruction to obey.

Repo-agnostic: discover each repository's conventions rather than assuming any one
repo's tooling.

## Hard rules

1. **Never claim validation that was not performed.** Not "tests should pass", not
   "this is covered" — either it was run and you saw the result, or the user confirmed
   it, or you say what was not checked. This is the single most common defect in PR
   bodies and review replies.
2. **Verify before disputing.** Telling a reviewer they are wrong requires reading the
   code first. The claim must be checked against source, not against your model of it.
3. **Never push without explicit approval.** Commit, then show what would be pushed and
   wait. Run push as a standalone command.
4. **One decision at a time.** No batching, including reply-only items.

## Intake

Work out which PR. If handed off from `pr-docket`, it is already known. Otherwise take a
PR reference, or infer from the current branch.

Collect unresolved inline threads, review comments, change requests, and general PR
comments that ask for something. Check all three sources — a push-back often arrives as
an inline reply rather than a review:

```bash
gh api repos/<owner>/<repo>/pulls/<n>/comments --paginate    # inline threads
gh api repos/<owner>/<repo>/pulls/<n>/reviews --paginate     # formal reviews
gh api repos/<owner>/<repo>/issues/<n>/comments --paginate   # general comments
```

## Orientation, then one at a time

Give a **brief overview** first: how many comments, from whom, and their rough shape.
A few lines. It tells the user whether this is a two-comment round or a twenty-comment
one, which changes how they want to spend the next half hour.

Then work **one comment at a time**. For each: classify it, propose what to do, show
**before and after** for any code change, and wait for approval. Do not proceed to the
next comment until this one is settled. Do not batch reply-only items — they are
decisions too.

## Classification

Five categories. Every comment gets exactly one.

| Category | Meaning |
|---|---|
| **Fix** | Sound, unambiguous, scoped, compatible with the repo's conventions |
| **Clarify** | Ambiguous, too broad, self-contradicting, or needs a product or architecture decision |
| **Dispute** | The reviewer is mistaken about the code, the codebase, or the consequence |
| **Reply only** | Already satisfied — point at where, and say nothing further |
| **Defer** | Valid, but belongs in separate work |

**Dispute is not Reply-only.** Telling a reviewer they are wrong is a different act from
telling them it is already done, and it carries hard rule 2: read the code, quote it,
and be willing to be wrong yourself. If verification does not support the dispute, it
was a Fix.

**Defer must name its destination.** An issue number, a ticket, a tracked follow-up. A
deferral with nowhere to go is a refusal wearing better clothes — say so honestly
instead.

Never partially implement ambiguous multi-item feedback while a Clarify is open.

## Fixing

1. Read the comment, the code it points at, and enough context to confirm the issue is
   real.
2. Apply the **smallest change that satisfies the reviewer.** No unrelated refactoring,
   no drive-by improvements.
3. Follow the repo's own rules — discover them (`AGENTS.md`, `CLAUDE.md`,
   `CONTRIBUTING.md`, repo skills) rather than assuming.
4. Record what changed, for the reply.

Show before and after. Get approval. Then apply.

## Verification

Tiered by what was touched. **Discover the commands per repo**; do not assume any
repo's tooling.

| Touched | Run |
|---|---|
| Docs, prompts, instructions, skills | Editor diagnostics; whitespace or lint check if the repo has one |
| Managed or application code | The narrowest reliable test scope; a build when project, resource or build inputs changed |
| Native or interop | Native tests plus a build |
| Build scripts or installer | The repo's own validation script, not an ad-hoc pipeline |

Where a repo requires an agent-specific check, run it. FieldWorks, for example, requires
`-CommentHygiene` on `build.ps1` / `test.ps1` for agent work.

**Hard rule 1 applies here.** State what was run and what was not.

## Git

Sequential git commands only — parallel invocations collide on `index.lock` on Windows.

Before staging:

1. `git status --short`
2. Separate pre-existing user changes from this workflow's changes.
3. Stage **explicitly by path.** Never stage broadly when unrelated or staged-deleted
   files are present.
4. If unrelated work is already staged, ask before absorbing it.

Environment constraints:

- **Worktrees** need `core.hooksPath` overridden with forward slashes before committing.
- **Commit with `-F <file>`, not `-m`** — PowerShell 5.1 mangles double quotes in `-m`.
- **Never bare `git stash` / `git stash pop`** — the stash stack is shared across
  worktrees and another session may pop yours. Prefer a temporary WIP commit.

### Commit message

**Subject names the dominant change specifically.** Not "address PR review comments" —
that tells `git log` a review happened, not what it did. The remaining changes go in the
body, one plain line each.

Two or three plain-word sentences on *how*, if the change needs it. Follow the repo's own
commit conventions where they exist — some enforce length limits in CI.

### Push

**Stop. Show what would be pushed. Wait for approval.** Run the push as a standalone
command. Never force-push without explicit approval, ever.

## Replying

Reply **in the thread**, never as an unrelated top-level comment.

| Category | Reply |
|---|---|
| Fix | The concrete change and what verification was run |
| Dispute | The evidence, with file:line. Concise, technical, no edge |
| Reply only | Where it is already handled |
| Clarify | The specific question needed to proceed |
| Defer | Why it belongs elsewhere, and where it is now tracked |

### Resolving

**Resolve only the mechanically obvious** — a typo corrected, a rename applied, a broken
link fixed. Anything involving judgement, disagreement or deferral stays open for the
reviewer to close.

**Never resolve a Dispute.** Telling a reviewer they are mistaken *and* closing the
thread on it takes both sides of a conversation that is theirs to finish.

## PR description

Update it when the work changed what a reviewer needs to know — otherwise leave it.

**Preserve the structure the PR already uses.** Do not impose a template. If the body has
collapsed sections, generated markers, or a house format, keep them intact.

Hard rule 1 applies: do not add validation claims to the description that were not
performed.

## Final report

Short. Group by outcome, not by reviewer:

- Fixed, and what verification ran
- Disputed, and on what evidence
- Replied without code change
- Deferred, and where it is tracked
- Still open, and what is blocking
- Commit SHA and whether it was pushed

Name what is still waiting on the reviewer versus what is waiting on the user.
