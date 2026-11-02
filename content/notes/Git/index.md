---
title: Git & GitHub
---

Notes on git refs and history, and on how GitHub layers pull requests, checks and merge automation on top of them.

## Pull requests & automation
- [[notes/Git/stale-pull-request-head-after-update-branch|The Pull Request That Forgot Its Head]] — "Update branch" moved the branch to a new merge commit but the PR kept its old head SHA, so no `synchronize`, no CI and `mergeable: null`; why a merge bot's `expected_head_sha` call returned the same 422 every time, base vs head, branches as pointers, the three-dot Files changed diff, detecting the mismatch through the API, resyncing with a fresh push, and the unknown root cause
