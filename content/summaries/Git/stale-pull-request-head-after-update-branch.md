---
title: "Summary: The Pull Request That Forgot Its Head"
---

> **Full notes:** [[notes/Git/stale-pull-request-head-after-update-branch|The Pull Request That Forgot Its Head: Update Branch and a Stale Head SHA -->]]

After "Update branch", the head branch moved to the new merge commit but the PR record kept the **old head SHA**. No `synchronize`, no CI, `mergeable: null` indefinitely. A merge bot passing the record's SHA as `expected_head_sha` got the **same 422 every time it was asked**. A fresh push from a clone resynced the PR.

## Key Concepts

### Branches and pull requests
- A branch is a ref under `refs/heads/` holding one SHA. Moving a branch means swapping the pointer.
- **Base** = the branch you merge into. **Head** = the branch with your changes.
- A PR is a database record, not a git object: `head.ref`, `head.sha`, `base.ref`, `base.sha`, `mergeable`, reviews, timeline. GitHub also keeps `refs/pull/<n>/head` and `refs/pull/<n>/merge`.
- `head.ref` is stable. `head.sha` is a **copy** that only changes when GitHub processes a push to the head branch.

### What a processed push triggers
- Record `head.sha` updated, plus a `committed` timeline entry.
- Mergeability job builds the test merge commit and sets `mergeable`.
- `pull_request` webhook with `action=synchronize`. Workflows `on: pull_request` run on it (default types `opened`, `synchronize`, `reopened`), against the test merge commit.
- The webhook docs list "head branch was updated from the base branch" under `synchronize`, so Update branch should trigger all of it.

### Update branch
- Merges the base tip into the head on the server, creating merge commit `M` with parents `F` and `D`. The UI also has a rebase variant.
- API: `PUT /repos/{owner}/{repo}/pulls/{pull_number}/update-branch` returns **202 Accepted**. Queued is not done.
- Only offered for clean merges.

### The Files changed tab is not a signal
- It's a **three-dot diff**: `git diff main...feature` = diff from `merge-base(main, feature)` to `feature`.
- Merging base into head moves the merge base to `D`. Base's commits appear on both sides and cancel, so the diff is identical before and after.
- It looks the same whether or not the PR processed the push.

### The stuck state
- Ref at `M`, record at `F`.
- `mergeable: null` / `mergeable_state: unknown` that never resolves.
- 0 workflow runs for `M` and no timeline entry for it.
- No UI error. Checks just never start.

### Why the 422 never goes away
- `expected_head_sha` is an **optimistic concurrency guard**, like `If-Match` (RFC 9110 §13.1.1), a k8s `resourceVersion` precondition, or `--force-with-lease`.
- The bot read `F` from the **record**. GitHub checked it against the **ref** (`M`): "expected head sha didn't match current head ref".
- A normal CAS failure heals on re-read. Here the re-read was stale too, so asking again could never succeed. **Deterministic, not transient.**
- The record behaves as **edge-triggered** state with no resync: one lost event means drift until the next push.

### Fix
- Any client push that GitHub processes.
- Empty commit on top of `M`: fast-forward, no force.
- Or rebase onto base and `git push --force-with-lease=<branch>:<fetched-sha>`.
- New head SHA means CI reruns, and stale approvals may be dismissed.

### Don't
- Retry update-branch, by bot or by hand. The 422 is also documented for "the endpoint has been spammed".
- Drop `expected_head_sha`, or swap in the ref's SHA, to dodge the 422. That removes the guard and leaves the record broken.
- Treat the Files changed tab as proof.
- Poll `mergeable: null` with no deadline.

### Root cause: unknown
- Ready-for-review landed **one second before** the update-branch merge.
- Guesses: a **lost update** (one handler writes back the old head SHA) or a skipped or deduplicated sync job. Neither can be verified from outside.
- Collect click, draft-change and first-422 timestamps for support. Detection doesn't depend on the cause.

## Quick Reference

```
 git ref (truth)                     PR record (copy)
 refs/heads/feature ──► M            head.sha = F        <- stale
                                     mergeable = null
                                     runs for M = 0

 bot:    GET /pulls/{n}              -> head.sha = F   (record)
         PUT .../update-branch {F}   -> compared with M (ref) -> 422
         run again                   -> still F -> 422
```

**Detect (read-only)**
```bash
gh api repos/$REPO/pulls/$PR --jq '.head.sha, .head.ref, (.mergeable|tostring), .mergeable_state'
gh api repos/$HEAD_REPO/git/ref/heads/$BRANCH --jq .object.sha        # must equal head.sha
gh api "repos/$REPO/actions/runs?head_sha=$BRANCH_SHA" --jq .total_count
gh api repos/$REPO/issues/$PR/timeline --paginate --jq '.[] | [.event, (.sha // .commit_id // "-")] | @tsv'
git ls-remote origin "refs/heads/$BRANCH" "refs/pull/$PR/head"
```
- jq gotcha: `.mergeable // "null"` turns `false` into `"null"`. Use `tostring`.

**Fix**
```bash
git fetch origin && git switch feature && git merge --ff-only origin/feature
git commit --allow-empty -m "Resync pull request" && git push origin feature
# or
git rebase origin/main
git push origin feature --force-with-lease=feature:"$(git rev-parse origin/feature)"
```

**Missing CI, but `head.sha` matches the ref?** Then it's something else: merge conflict, path filters, `[skip ci]`, approval needed for first-time contributors, or a `GITHUB_TOKEN` push.

**Force-push variants**
| Flag | Overwrites when |
|---|---|
| `--force` | always |
| `--force-with-lease` | remote is still at the expected SHA (explicit, or `origin/<branch>`) |
| `--force-if-includes` (git 2.30+) | also, the remote tip is reachable from the local branch's reflog |

**Merge bot rules**
- Compare `head.sha` with the ref before acting. If they disagree past a grace period, stop and comment "push to resync".
- A 422 right after a push: re-read once and retry. The same 422 with the same `head.sha` twice: stuck, stop.
- Put a deadline on `mergeable: null`.
- Track a metric: PRs with `head.sha != ref` for more than N minutes. It should be 0.
