---
title: "The Pull Request That Forgot Its Head: Update Branch and a Stale Head SHA"
---

## The short version

A pull request got stuck right after someone clicked GitHub's "Update branch" button. On the git side the button did its job: it merged the base branch into the PR branch, and the branch moved to the new merge commit. The pull request never noticed. Its stored head SHA still pointed at the old commit, no CI ran on the new one, and `mergeable` stayed `null` (`mergeable_state: unknown`) for as long as anyone watched.

Then someone asked the repo's merge bot to update the branch again, using its comment command. The bot called the update-branch API with `expected_head_sha` set to the head SHA it read from the pull request, and GitHub rejected the call with `422 expected head sha didn't match current head ref`. The command was run again 18 minutes later. The bot read the same stale SHA and got the same 422.

What fixed it was an ordinary push from a clone: an extra commit, or a rebase followed by `git push --force-with-lease`. GitHub processed that push like any other, the pull request picked up the new head, CI ran, and the PR merged.

The underlying problem is simple once you see it. A branch is a pointer, and the pull request keeps its own copy of where that pointer was. The copy only changes when GitHub processes a push to the branch. If that processing doesn't happen, the two disagree, and anything that reads one and checks against the other will fail forever.

This note covers base and head, what "Update branch" does to the commit graph, why the Files changed tab is no help, how to detect the stuck state through the API, and what not to do. The root cause inside GitHub is unknown. There's one clue, covered near the end, and it's only a guess.

---

## Part 1: base, head and what a branch is

### A branch is a pointer

In git a branch is a ref: a name under `refs/heads/` that holds one commit SHA. Commits are immutable and point at their parents. When a branch "moves", the ref now holds a different SHA and nothing about the old commits changes.

```
 refs/heads/main     ──►  D
 refs/heads/feature  ──►  F

 A ── B ── C ── D        main
  \
   E ── F                feature
```

A push asks the server to update a ref from one SHA to another. The server checks the request (fast-forward or not, branch protection, permissions) and swaps the pointer.

### Base and head

A pull request names two branches. The base is the branch you want to merge into (`main` here). The head is the branch that carries your changes (`feature`).

"Head" is overloaded, which is part of why this gets confusing. In the REST API, `head.ref` is the head branch's name, which doesn't change. `head.sha` is the commit the pull request believes is at the tip of that branch, and it changes with every push. `base.ref` and `base.sha` work the same way for the base.

### The pull request keeps its own copy

A pull request isn't a git object. It's a record in GitHub's application database holding the branch names, a head SHA, a base SHA, a mergeability result, reviews, the timeline and so on. GitHub also maintains `refs/pull/<n>/head` and, when the PR merges cleanly, `refs/pull/<n>/merge` in the base repository so that CI and other integrations can fetch the PR.

From the outside, the record behaves like a cache that gets refreshed by push processing. When GitHub handles a push to the head branch, it updates the record's head SHA, adds the commit to the timeline, starts a mergeability job that builds a fresh test merge commit, and emits a `pull_request` webhook with action `synchronize`. CI hangs off that last step.

```
 push to refs/heads/feature
          │
          ▼
 GitHub processes the push for each open PR whose head is `feature`
          │
          ├── PR record: head.sha = <new tip>
          ├── timeline: `committed` entry for the new commit
          ├── mergeability job: build refs/pull/<n>/merge, set `mergeable`
          └── webhook: pull_request, action = synchronize
                    │
                    └── workflows `on: pull_request` (default types:
                        opened, synchronize, reopened) run against
                        the test merge commit
```

That's a description of observable behaviour, not of GitHub's internals. The point that matters is that the git ref and the PR's head SHA are two separate pieces of state, and push processing is the only thing keeping them in step.

### What "Update branch" does

The button, and the `PUT /repos/{owner}/{repo}/pulls/{pull_number}/update-branch` endpoint behind it, merges the current base tip into the head branch on the server and moves the head branch to the result. The UI also offers "Update with rebase". The merge variant is the default and it's the one involved here.

```
 before                                  after "Update branch"

 A ── B ── C ── D       main             A ── B ── C ── D        main
  \                                       \              \
   E ── F               feature            E ── F ─────── M      feature

                                         M = "Merge branch 'main' into feature"
                                         parents: F and D
```

GitHub's webhook docs name this case explicitly under `synchronize`: the action fires when "the head branch was updated from the base branch". So the expected sequence after the click is: the ref moves from F to M, the push is processed, the PR's head becomes M, CI runs on M, and mergeability is recomputed.

The API returns `202 Accepted` (RFC 9110 §15.3.3). The merge happens in the background, so the response tells you the request was queued, not that the branch or the PR has caught up.

---

## Part 2: why the Files changed tab doesn't move

The first thing most people look at after clicking the button is the Files changed tab. It looks the same before and after. Normally that's fine. In this case it hid the fact that the pull request hadn't caught up.

The tab shows a three-dot diff: the head compared with the merge base of base and head, rather than with the base tip. In git terms that's `git diff main...feature`, which is shorthand for `git diff $(git merge-base main feature) feature`.

```
 before update    merge-base(D, F) = A    diff(A, F) = the changes from E and F
 after update     merge-base(D, M) = D    diff(D, M) = the changes from E and F
```

After the merge, B, C and D are on both sides of the comparison, so they drop out, and what's left is the same set of changes the PR started with. That's the purpose of the three-dot view: reviewers see what the PR introduces, not what landed on `main` in the meantime. The button is only offered when the merge is clean, so there's no conflict resolution in M to show up either.

The result is that the Files changed tab looks identical whether the pull request processed the push or not. It can't tell you anything about this failure. The things that do change are the PR's head SHA, the checks attached to the new commit, and the timeline.

---

## Part 3: what went wrong

### The ref moved and the record didn't

After the click, the two sources disagreed:

```
 git (source of truth for the branch)        PR record
 ────────────────────────────────────        ───────────────────────────────
 refs/heads/main     ──►  D                  base.ref         = main
 refs/heads/feature  ──►  M    (new)         head.ref         = feature
                                             head.sha         = F   (stale)
                                             mergeable        = null
                                             mergeable_state  = unknown
```

The merge commit existed and the branch pointed at it. `git ls-remote` and the git refs API both returned M. The pull request returned F.

### Everything downstream stalled

Each symptom is something that normally happens because a push was processed:

- No workflow runs for M. `on: pull_request` workflows start on `synchronize`, and no `synchronize` was processed for M.
- `mergeable` stayed `null`. The REST docs say `null` means a background job is computing mergeability and you should ask again after giving it time. That's normal for a few seconds after a push. Here it never resolved, which fits with no job ever starting for the new head. `pull_request` workflows run against the test merge commit that this job produces, so they were blocked on it too.
- The timeline had no entry for M.

None of this shows up as an error in the UI. There's no banner. The PR just sits there with checks that never start.

### Why the bot's call failed every time

`expected_head_sha` is an optimistic concurrency guard. The docs describe it as "the expected SHA of the pull request's HEAD ref", return 422 if it doesn't match, and default it to "SHA of the pull request's current HEAD ref". It's the same idea as HTTP `If-Match` (RFC 9110 §13.1.1) or a `resourceVersion` precondition on a Kubernetes update: do this only if the thing is still at the version I looked at. It stops a bot from merging `main` into a branch that someone pushed to after the bot last checked.

When asked to update the branch, the bot followed the documented pattern. It read the PR, took `head.sha`, and passed it as the guard. The trouble was where each side of the comparison came from:

```
 bot                              GitHub                           git ref
  │                                 │                                 │
  │  GET /pulls/{n}  ──────────────►│                                 │
  │◄───────────── head.sha = F ─────│  (read from the PR record)      │
  │                                 │                                 │
  │  PUT /pulls/{n}/update-branch   │                                 │
  │  { expected_head_sha: F } ─────►│  compare F with current ref ───►│  M
  │◄──── 422 expected head sha ─────│  F != M                         │
  │      didn't match current       │                                 │
  │      head ref                   │                                 │
  │                                 │                                 │
  │  run again: read again, still F, same 422                         │
```

The bot read F from the record, and GitHub compared it with M from the ref. The error message gives this away with the words "current head ref".

An ordinary precondition failure fixes itself. You re-read, get the new version, and the next attempt passes. That only works when the read and the check use the same source. Here they didn't, and the read kept returning the stale value, so asking again could never succeed. The failure was deterministic, but it looked like a transient one.

It also didn't need doing. The branch already contained D. The update was only requested because, going by the pull request, nothing had happened yet.

### Edge-triggered state with no resync

From the outside, the PR's head SHA behaves like edge-triggered state: it changes when a push event is processed. If that one event is lost, nothing goes back later to compare the record with the ref, so the drift stays until the next push comes along. Kubernetes controllers are level-triggered and resync periodically for exactly this reason: a missed watch event gets corrected on the next full reconcile. The PR record got no second chance of that kind.

---

## Part 4: detecting it

There are three read-only checks.

1. Compare the PR's head SHA with the branch ref. They should match within seconds of a push. A mismatch that lasts for minutes is the tell.
2. Count workflow runs for the branch tip. Zero runs for a commit that has been on the branch for minutes, in a repo where every PR gets CI, means no `synchronize` was processed.
3. Read the timeline. A processed push shows up as a `committed` event for the new SHA.

```bash
#!/usr/bin/env bash
# pr-head-check.sh OWNER/REPO PR_NUMBER
# Read-only. Compares the pull request's head SHA with the actual branch ref.
set -euo pipefail
repo=$1 pr=$2

pr_json=$(gh api "repos/$repo/pulls/$pr")
record_sha=$(jq -r '.head.sha'             <<<"$pr_json")
head_repo=$(jq -r  '.head.repo.full_name'  <<<"$pr_json")   # differs for forks
head_ref=$(jq -r   '.head.ref'             <<<"$pr_json")
mergeable=$(jq -r  '.mergeable | tostring' <<<"$pr_json")   # true, false or null
state=$(jq -r      '.mergeable_state'      <<<"$pr_json")

branch_sha=$(gh api "repos/$head_repo/git/ref/heads/$head_ref" --jq '.object.sha')
runs=$(gh api "repos/$repo/actions/runs?head_sha=$branch_sha" --jq '.total_count')

printf 'pr record head : %s\n' "$record_sha"
printf 'branch ref     : %s\n' "$branch_sha"
printf 'mergeable      : %s (%s)\n' "$mergeable" "$state"
printf 'runs on ref    : %s\n' "$runs"

if [[ "$record_sha" != "$branch_sha" ]]; then
  echo "MISMATCH: the pull request has not picked up the latest push to its branch"
fi
```

(`.mergeable | tostring` is deliberate. The more obvious `.mergeable // "null"` also turns `false` into `"null"`, because jq's `//` treats `false` as missing.)

In the stuck state the output looks like this, with placeholder SHAs:

```
pr record head : aaaaaaa...    (F, the commit before the button)
branch ref     : bbbbbbb...    (M, the merge commit the button created)
mergeable      : null (unknown)
runs on ref    : 0
MISMATCH: the pull request has not picked up the latest push to its branch
```

The timeline lives under the issues API, because every pull request is also an issue. `committed` events carry `sha` and a committer date rather than `created_at`, so the filter has to handle both shapes:

```bash
gh api "repos/$repo/issues/$pr/timeline" --paginate \
  --jq '.[] | [.event, (.sha // .commit_id // "-"), (.created_at // .committer.date // "-")] | @tsv'
```

Two more places are worth looking at. `git ls-remote origin refs/heads/feature "refs/pull/$pr/head"` prints GitHub's own PR ref next to the branch, which gives a third view of where the head is. And if the repository has a webhook subscribed to `pull_request` events, its Recent Deliveries list shows whether a `synchronize` delivery went out for the PR around the time of the click.

On timing: a `null` mergeable for the first several seconds after a push is normal, so don't alarm on it straight away. A head SHA that still doesn't match the ref after a few minutes is a different matter. In this case it stayed wrong until someone pushed.

---

## Part 5: the fix

Give GitHub a push to process. There are two ways to do it from a normal clone.

The first is an empty commit on top of the merge commit. It needs no force push and rewrites no history:

```bash
git fetch origin
git switch feature
git merge --ff-only origin/feature        # pick up M, the merge commit the button made
git commit --allow-empty -m "Resync pull request"
git push origin feature                   # fast-forward from M to the new commit
```

The second is to rebase onto the base and force-push with a lease. That drops the merge commit and leaves linear history:

```bash
git fetch origin
git switch feature
git rebase origin/main                    # replays E and F onto the tip of main
git push origin feature \
  --force-with-lease=feature:"$(git rev-parse origin/feature)"
```

The explicit lease value says: replace the remote branch only if it's still at M, the tip I just fetched. If someone pushed in between, the push is rejected instead of silently throwing their commit away.

The empty commit is the less invasive option. The rebase is better if the repo wants linear history or nobody wants the merge commit. Either way the PR gets a new head SHA, so CI runs from scratch, and if branch protection dismisses stale approvals, existing reviews will be dismissed.

After the push, the timeline showed the new commit, the record's head SHA moved to it, CI started, and the PR merged about a minute later.

Why a client push worked when the button's server-side merge didn't isn't clear. A client push goes through the ordinary receive path, which is the most heavily used trigger GitHub has for PR sync. Operationally, the stuck state was a missing event and the fix was a fresh one.

---

## Part 6: what not to do

Retrying update-branch doesn't help, whether a bot does it or a person does. Retries help with transient failures, and this one isn't transient. The record says F, the ref says M, and another call changes neither. Automation that retries on a schedule just piles up failed attempts, and the endpoint's 422 is also documented as the response when "the endpoint has been spammed", so hammering it makes the logs harder to read.

Removing `expected_head_sha` to get past the 422 is the wrong fix. The guard is what stops the bot from acting on a branch state it hasn't seen. Dropping it trades a stuck PR for a race, and it doesn't touch the real problem, which is the PR record. Passing the branch ref's SHA instead of the record's would also get past the check, but it hides the inconsistency rather than fixing it, and there's no reason to expect a second server-side merge to be processed better than the first.

The Files changed tab isn't evidence of anything here, for the reasons in Part 2.

Polling `mergeable: null` with no deadline is how a bot gets stuck without telling anyone. Polling for a while is correct. Polling forever without comparing the head SHA to the ref is not.

---

## Part 7: what caused it (unknown)

The root cause inside GitHub isn't known. The one clue is timing: the PR was marked ready for review one second before the Update branch merge landed. Two state changes to the same pull request, one from the draft toggle and one from the push, arrived almost together.

One plausible explanation is a lost update. If both handlers read the PR record, changed their own field, and wrote the whole record back, the ready-for-review write could have carried the old head SHA and overwritten the push's update. Another is that the push's sync job was skipped or deduplicated while the PR was changing state. Both are guesses. Nothing visible through the API can tell them apart, and a single occurrence doesn't make a pattern.

What can be said with confidence is narrower: the branch ref moved, the PR record didn't, and a new push brought them back in line. If it happens again, write down the time of the click, the time of the draft change and the time of the first failed call, along with the PR and the commit, and send them to GitHub support. They can see the internal job logs and nobody outside GitHub can. If your own automation marks PRs ready and updates branches, putting a few seconds between the two costs nothing while the cause is unknown.

---

## Part 8: making a merge bot robust to it

- Before calling update-branch, fetch the branch ref and compare it with the PR's `head.sha`. If they've disagreed for longer than a short grace period, the PR is out of sync and no API call the bot can make will fix it.
- Treat the 422 differently depending on what happened before it. "expected head sha didn't match" right after a known push is a normal race, so re-read once and retry. The same error twice with the same `head.sha` means the PR is stuck, and the bot should stop.
- Tell a human what to do. A comment saying the pull request is out of sync with its branch and that a push will resync it turns a silent loop into a quick fix.
- Put a deadline on `mergeable: null` and report it when the deadline passes.

---

## Interview Prep

### Q: In a pull request, what are the base and the head, and what does GitHub store for each?

**A:** The base is the branch the change will be merged into and the head is the branch that carries the change. Both are git refs, which are names that point at one commit each.

The pull request itself is a database record, not a git object. It stores the branch names (`base.ref`, `head.ref`), the commit it believes is at each tip (`base.sha`, `head.sha`), a mergeability result, reviews and a timeline. GitHub also keeps `refs/pull/<n>/head` and `refs/pull/<n>/merge` in the base repo for integrations.

The detail that matters in practice: the ref name is stable, but the SHA is a copy. It changes when GitHub processes a push to the head branch. If that processing doesn't happen, the record and the branch can disagree.

### Q: You click "Update branch" and the Files changed tab looks exactly the same. Did it work?

**A:** The tab can't answer that. It shows a three-dot diff, which compares the head with the merge base of base and head:

```
 git diff main...feature  ==  git diff $(git merge-base main feature) feature
```

Merging `main` into the branch moves the merge base up to the current `main` tip. The commits from `main` then appear on both sides and drop out, so the diff is the PR's own changes, before and after. That's intended, because reviewers want to see what the PR introduces.

To check whether the update worked, I'd look at the PR's head SHA against the branch ref, the checks on the new commit, and the timeline.

### Q: A commit is on a PR's branch, no CI has run for it, and `mergeable` is `null`. How do you investigate?

**A:** First I'd separate "slow" from "never processed". `mergeable: null` for a few seconds after a push is normal. After that:

1. Compare `head.sha` from `GET /pulls/{n}` with the ref from `GET /git/ref/heads/{branch}`.
2. List workflow runs filtered by `head_sha` for the branch tip.
3. Read the issue timeline for a `committed` event for that SHA, and check webhook deliveries for a `synchronize` if there's a webhook.

If the head SHA matches the ref, the PR knows about the commit and the missing CI has some other cause. The usual ones: a merge conflict (`pull_request` workflows don't run until it's resolved), path filters that exclude the change, a `[skip ci]` in the commit message, workflows that need approval for first-time contributors, or a push made by a workflow with `GITHUB_TOKEN`, whose runs don't start on their own.

If the head SHA doesn't match the ref, the PR never processed the push. Waiting won't fix that. A new push will.

### Q: What is `expected_head_sha` for, and why did the bot get the same 422 every time it was asked?

**A:** It's a compare-and-swap guard: only merge base into the branch if the branch is still at the SHA I expect. It's the same pattern as `If-Match` with ETags, Kubernetes `resourceVersion` preconditions, or `git push --force-with-lease`. It prevents the bot from acting on a branch someone has pushed to since it last looked.

A CAS failure normally recovers on the next attempt, because you re-read and get the current version. That assumes the read and the check use the same source. Here the bot read `head.sha` from the PR record, which was stale at F, and GitHub checked it against the actual ref, which was at M. Every re-read returned F, so every check failed. Asking again couldn't help, because the input never changed.

The general lesson: before you retry a precondition failure, make sure your re-read can actually observe the new state.

### Q: Why does pushing a new commit fix it, and what kind of push would you use?

**A:** The stuck state was a push that never got processed for the PR. A new push is a new event. GitHub handled it normally, the record's head moved to the new SHA, `synchronize` fired, CI ran and mergeability resolved.

I'd pick between two pushes. An empty commit on top of the merge commit is a fast-forward, so it needs no force and rewrites nothing. A rebase onto `main` with `--force-with-lease=<branch>:<sha-I-fetched>` gives linear history and drops the merge commit, and the lease means the push is refused if someone pushed in the meantime. Either way the head SHA changes, so CI reruns, and stale approvals may be dismissed if branch protection is set up that way.

### Q: Explain `--force`, `--force-with-lease` and `--force-if-includes`.

**A:** `--force` overwrites the remote ref unconditionally. If a teammate pushed a commit you never fetched, it's gone from the branch.

`--force-with-lease` makes the overwrite conditional on the remote ref still being at an expected SHA, which is a compare-and-swap on the ref. With an explicit value (`--force-with-lease=feature:<sha>`) you state the expected SHA yourself. Without one, git uses your remote-tracking ref, `origin/feature`.

The gap in the bare form is that a background fetch (from an IDE, say) can update `origin/feature` to include a teammate's commit you've never looked at. The lease then passes and you overwrite their work anyway. `--force-if-includes` (git 2.30+) closes that gap by also requiring that the remote tip be reachable from your local branch's reflog, meaning you actually had it locally at some point.

It's the same idea as `expected_head_sha`, applied by the git client instead of the GitHub API.

### Q: How would you design a merge bot so this becomes a two-minute fix instead of an infinite loop?

**A:** Make it level-triggered, and make it check its own inputs.

On each pass it reads both the PR record and the branch ref. If they disagree past a grace period, the PR is out of sync, and the bot stops and comments that a push will resync it. It doesn't call update-branch on a PR in that state.

It classifies errors instead of retrying everything. A 422 on `expected_head_sha` right after a push gets one re-read and one retry. The same 422 with the same `head.sha` twice in a row is a stuck PR. `mergeable: null` gets polled with backoff and a deadline, and then it gets reported.

And it emits a metric for PRs whose `head.sha` has disagreed with the branch ref for more than a few minutes. Most of the time that number should be zero. When it isn't, you find out from a graph instead of from someone asking why their PR has been waiting all afternoon.

### Q: What do you think caused it, and how would you confirm it?

**A:** I don't know, and I'd say so. The only clue is that the PR went from draft to ready for review one second before the update-branch merge landed.

My leading guess is a lost update: two handlers each read the PR record, each changed one field, and each wrote the whole record back. The ready-for-review write carried the old head SHA and overwrote the push's change. That's the classic read-modify-write race, and the usual defences are field-level updates or a version check on write. The other possibility is that the push's sync job was dropped or deduplicated while the PR was changing state.

From outside GitHub you can't tell those apart. To confirm it, I'd collect the exact timestamps of the click, the draft change and the first failed call, plus the PR and the commit, and send them to GitHub support, who can see the job logs. Meanwhile I'd make the bot detect the state, because detection doesn't depend on knowing the cause.

---

## Related Notes

- [[notes/AuthNZ/ssh-certificates-git-and-identity-order|SSH Certificates, Git and the Identity Offer Order]]: how git pushes reach GitHub over SSH, and another case where a quick sanity check (`ssh -T`) said everything was fine while the real operation failed
- [[notes/K8s/server-side-apply-replicas-collapse|When Server-Side Apply Deletes Your Replicas]]: two views of the same object that look consistent until you read the metadata underneath
- [[notes/K8s/kubebuilder-controllers-and-webhooks|Kubebuilder Controllers, Webhooks & Extension APIs]]: reconcile loops, requeues and optimistic locking, the level-triggered design that would have caught this drift
- [[notes/K8s/config-sync-rootsync-cluster-selectors-and-field-ownership|Config Sync at Fleet Scale]]: more systems that treat a git ref as the source of truth and cache their own view of it
