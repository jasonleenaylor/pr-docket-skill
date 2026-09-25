# Posting a review

One API call carries inline comments and the summary together.

## Anchoring

**Anchor every finding that can be anchored.** Inline comments are what the author sees
first; a review whose substance sits in a wall of summary prose wastes that.

GitHub permits inline comments only on files present in the diff. This bites often,
because the most valuable findings frequently concern code the PR does not touch — a
nested restore in an unmodified `.targets`, a crash mechanism in a file the PR never
opened, a fail-closed check in an unrelated class.

**Anchor to the nearest changed line the finding is about**, and explain the chain in the
comment body. A finding about an unmodified `PackageRestore.targets` anchors to the
`build.ps1` line that reaches it.

Reserve the summary for findings with **no** anchor at all: scope objections, verification
demands, cross-cutting design push-back.

## Comment shape

An inline comment is short. Lead with the mechanism, citing the line and the identifier;
state the consequence; end with the concrete change — the replacement text, the missing
assertion, the changelog line. Not an open-ended discussion. Where the author has not yet
confirmed the mechanism, frame the finding as an observation rather than a verdict.

## Drafting

Write the summary and every inline comment to the PR's state directory (see
`state.md`) and open the files in the user's editor. The user edits in place. Before
posting, reload from disk and diff against what was last shown, then post the edited text
verbatim — never a regenerated version of it.

## The summary body

Always present. **Brief. Free-form.** It carries only what the inline comments do not.

Do not restate inline findings. Do not use a fixed skeleton. Say what needs saying about
the change as a whole.

The attribution trailer goes **on the summary only**, never on inline comments:

```
_This review was assisted by <model name>._
```

Attribution is a **per-repo setting, default on**, recorded in the repo's state
directory. Take the model name from the live session; never hardcode it.

## The call

Check the PR is still open in the same step, not earlier in the session (hard rule 5):

```bash
gh pr view <n> --repo <owner>/<repo> --json state,headRefOid --jq '{state, headRefOid}'
gh api repos/<owner>/<repo>/pulls/<n>/reviews -X POST --input review.json
```

If `state` is not `OPEN`, or `headRefOid` has moved since the draft was written, stop and
say so.

```json
{
  "commit_id": "<head sha>",
  "event": "REQUEST_CHANGES",
  "body": "<summary markdown>",
  "comments": [
    { "path": "build.ps1", "line": 591, "side": "RIGHT", "body": "<finding markdown>" }
  ]
}
```

`event` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`. Map from the verdict:

| Verdict | event | after |
|---|---|---|
| approve | `APPROVE` | — |
| nit pick | `APPROVE` | — |
| request changes | `REQUEST_CHANGES` | — |
| request redesign | `REQUEST_CHANGES` | — |
| reject | `COMMENT` | `gh pr close <n>` |

Pass `commit_id` explicitly so the review is pinned to the diff that was reviewed.

If an inline comment is rejected because its line is not in the diff, move that finding
to the summary rather than dropping it.

## Approval and verification

**Never post without explicit approval.** Show the complete body — summary and every
inline comment — and wait.

**Never name a withdrawn claim.** If a finding was investigated and dropped, it simply
does not appear. The author never saw the draft; explaining what was removed is noise.

After posting, verify:

```bash
gh pr view <n> --repo <owner>/<repo> \
  --json reviews --jq '.reviews[-1] | {author: .author.login, state, submittedAt}'
```

Confirm the author, the state, and the timestamp. Record the timestamp and the head SHA
in the PR's ledger — both drive tier 1a promotion later.

For a rejection, close the PR as a separate step and verify `state == "CLOSED"`.

## Rejections

A rejection closes a PR someone worked on. Give the full reasoning, and say plainly what
a version you would accept looks like — concretely, as a list, not as a gesture.

Credit what is genuinely right about the work first. If the diagnosis is sound and only
the approach is wrong, say so, because that is the part worth keeping.
