---
title: "Summary: SSH Certificates, Git and the Identity Offer Order"
---

> **Full notes:** [[notes/AuthNZ/ssh-certificates-git-and-identity-order|SSH Certificates, Git and the Identity Offer Order -->]]

## Key Concepts

### The two credentials
- Hardware-bound key: the private key lives in the secure element, can't be exported, and is different on every machine
- The agent holds the key; `IdentityFile` can point at the **public** key so ssh uses the matching agent key
- A CA-signed short-lived **certificate** wraps the same key, so it has the same fingerprint but is a different credential
- Plain key = on your GitHub account, not SSO-authorized; used for personal/public repos and verifying commit signatures
- Certificate = trusted by the org's CA; the only intended way into SSO-enforced org repos

### How git reaches the certificate
- GitHub's CA feature uses the SSH username `org-<ORG_ID>@github.com`
- `url.<new>.insteadOf <old>` rewrites URLs before ssh runs, longest prefix wins
- Git runs `ssh user@host 'git-upload-pack <repo>'`; `GIT_TRACE=1` shows the exact command

### Why the order matters
- Phase 1, SSH authentication: "which user is this?" Any key on your account passes
- Phase 2, GitHub authorization at `git-upload-pack`: "may this credential read this org repo?" SSO is enforced here
- ssh stops at the first accepted identity and never retries after Phase 2 fails
- `Host *` matches every connection; `IdentityFile` accumulates; explicit identities are offered before agent ones
- So `Host * / IdentityFile key.pub` puts the plain key ahead of the certificate and triggers the SAML SSO error

### False positives and traps
- `ssh -T` "successfully authenticated" only proves Phase 1
- `ssh -T github.com` always exits 1, which breaks `| grep -q` under `pipefail`
- Old machine worked with the same config, most likely because its old key was SSO-authorized (unverified), which hid the bug
- The signing key must be the plain key; `ssh-add -L` lists the cert first, so use `tail -1`
- Setup scripts that copy dotfiles undo hand fixes on the next run

## Quick Reference

```sh
# What's in the agent / is there a cert?
ssh-add -L | awk '{print $1}'
ssh-add -L | grep cert | ssh-keygen -L -f /dev/stdin

# Effective config and actual offer order
ssh -G org-<ORG_ID>@github.com | grep -i '^identityfile'
ssh -vT org-<ORG_ID>@github.com 2>&1 | grep -E 'Offering|accepts'

# What git really runs
git config --show-origin --get-regexp '^url\.'
GIT_TRACE=1 git ls-remote git@github.com:<org>/repo.git HEAD

# Decisive test: ignore the config, agent order only (cert first)
GIT_SSH_COMMAND='ssh -F /dev/null -o IdentityAgent=~/path/to/agent.socket' \
  git ls-remote git@github.com:<org>/repo.git HEAD

# Signing key = plain key
git config --global user.signingKey "key::$(ssh-add -L | tail -1)"
```

```
# Fix: keep the explicit IdentityFile away from org-* connections
Match host github.com user org-*
    IdentityAgent ~/path/to/agent.socket

Match !user org-*
    IdentityFile ~/.ssh/agent-key.pub
    IdentityAgent ~/path/to/agent.socket
```
