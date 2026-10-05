---
title: "SSH Certificates, Git and the Identity Offer Order"
---

I moved to a new laptop, ran my setup script, and every clone from my company's GitHub organization failed. The odd part was that SSH itself looked healthy. `ssh -T` greeted me by name, the agent was running, the key was on my GitHub account, and the exact same config cloned fine on the old laptop.

The error was this:

```
ERROR: The '<org>' organization has enabled or enforced SAML SSO.
To access this repository, you must use the HTTPS remote with a personal access token
or SSH with an SSH key and passphrase that has been authorized for this organization.
fatal: Could not read from remote repository.
```

The cause turned out to be the **order in which ssh offers identities**. Explaining why that order matters means going through how SSH authentication works, how git drives ssh, what an SSH certificate is, and where in the connection GitHub actually checks SSO. This note covers each of those, then the debugging sequence and the fix.

## Part 1: the moving pieces

### Plain SSH keys

An SSH key pair is a private key, which stays with you, and a public key, which you hand out. To log in, the client proves it holds the private key by signing a challenge, and the server checks the signature against a public key it already trusts. On GitHub, "trusts" means the public key is listed under your account's SSH keys, and that list is how GitHub maps a key to a username.

### Hardware-bound keys and the agent

Usually the private key is a file such as `~/.ssh/id_ed25519`. With a **hardware-bound key**, it lives in the machine's secure element (Secure Enclave or TPM) and can never be exported. Nothing, not even root on the same machine, can read it. The only thing you can do is ask the hardware to sign.

An **SSH agent** sits in front of that hardware. ssh talks to it over a Unix socket (`SSH_AUTH_SOCK`, or `IdentityAgent` in the ssh config) and asks it to sign challenges. `ssh-add -L` lists the public halves of whatever the agent holds.

Two consequences:

- **The key can't be migrated.** A new laptop means a new key, and every place that trusted the old key has to learn the new one.
- **There's no private key file.** That's why the config points `IdentityFile` at a **public** key file. ssh then uses the matching private key from the agent, which is an ssh feature that's easy to miss: it lets you name which agent key to use without having a private key file on disk.

### SSH certificates

An SSH certificate is your public key plus metadata (principals, a validity window, extensions), all signed by a **certificate authority (CA)**. A server that trusts the CA accepts any certificate the CA signed, without having each user's key listed individually.

The corporate setup I had looks like this:

1. The agent holds one hardware-bound key.
2. After you sign in to the company identity provider, an internal CA signs that key and returns a **short-lived certificate**. The agent renews it automatically.
3. GitHub is configured to trust that CA for the company's organizations.

So the agent offers **two identities built from the same key**:

```
$ ssh-add -L | awk '{print $1}'
ecdsa-sha2-nistp256-cert-v01@openssh.com   <- the certificate
ecdsa-sha2-nistp256                        <- the plain key
```

Both have the **same fingerprint**, because a certificate wraps the key rather than replacing it. That's why ssh's debug output shows the same `SHA256:...` for both lines and only the type differs (`ECDSA` vs `ECDSA-CERT`).

The two identities have different jobs:

| Identity | GitHub trusts it because... | Used for |
| --- | --- | --- |
| Plain key | it's listed on your user account | personal and public repos, and verifying signed commits |
| Certificate | it's signed by a CA the org trusts | the org's repos, which satisfies the org's SSO requirement |

The plain key is deliberately **not** SSO-authorized for the org. The certificate is meant to be the only way into org repos.

### How the certificate path is selected: a different SSH username

GitHub's CA feature uses a special SSH username, `org-<ORG_ID>@github.com` instead of the usual `git@github.com`. The username tells GitHub which org's CA to consider.

Nobody wants to type that into every clone URL, so a git URL rewrite in `~/.gitconfig` does it:

```ini
[url "org-<ORG_ID>@github.com:<org>/"]
    insteadOf = git@github.com:<org>/
[url "git@github.com:"]
    insteadOf = https://github.com/
```

Git applies `insteadOf` before it ever launches ssh, using the **longest matching prefix**. So `git@github.com:<org>/repo.git` becomes `org-<ORG_ID>@github.com:<org>/repo.git`. Repos outside the org don't match and keep `git@`. `GIT_TRACE=1` shows the exact ssh command git runs:

```
$ GIT_TRACE=1 git ls-remote git@github.com:<org>/some-repo.git HEAD 2>&1 | grep run_command
trace: run_command: ... ssh -o SendEnv=GIT_PROTOCOL org-<ORG_ID>@github.com 'git-upload-pack '\''<org>/some-repo.git'\'''
```

Git is just a client of ssh here. It hands ssh a username and host, then runs `git-upload-pack` (for fetch and clone) or `git-receive-pack` (for push) as the remote command.

## Part 2: where it breaks

### SSH config: every matching block applies

The config, as the setup guide wrote it:

```
Match host github.com user org-*
    IdentityAgent ~/path/to/agent.socket

Host *
    IdentityFile ~/.ssh/agent-key.pub
    IdentityAgent ~/path/to/agent.socket
```

Two ssh_config rules are easy to get wrong:

1. **Every matching `Host`/`Match` block applies**, not just the first. `Host *` matches every connection, including `org-*@github.com`.
2. For most options, **the first value obtained wins**. `IdentityFile` is the exception: values **accumulate**.

So for an org connection, ssh ends up with `IdentityFile ~/.ssh/agent-key.pub`, which is the **plain** key. `ssh -G` prints the effective config after every block has been applied, so you can confirm it instead of reasoning it out:

```
$ ssh -G org-<ORG_ID>@github.com | grep -i '^identityfile'
identityfile ~/.ssh/agent-key.pub
```

### The offer order

ssh tries identities one at a time:

1. Identities named by `IdentityFile` come first. If one also exists in the agent, debug output marks it `explicit agent`.
2. Then any other agent identities, in the order the agent lists them.

With the config above, the plain key goes first because it's explicit. The certificate is second, even though the agent lists it first.

```
$ ssh -vT org-<ORG_ID>@github.com 2>&1 | grep -E 'Offering|accepts|Authenticated'
debug1: Offering public key: ~/.ssh/agent-key.pub ECDSA SHA256:... explicit agent
debug1: Server accepts key: ~/.ssh/agent-key.pub ECDSA SHA256:... explicit agent
Authenticated to github.com using "publickey".
```

The certificate never appears, because ssh stops at the first identity the server accepts.

### Why the first accepted identity is final: authentication vs authorization

This is the core of the problem. An SSH session to GitHub happens in two separate phases:

**Phase 1: user authentication (the SSH protocol).** The client offers a public key. If the server would accept it, it replies `PK_OK` (the "Server accepts key" line). The client then signs with that key and the session is authenticated. The only question GitHub answers here is **"which GitHub user is this?"** A plain key that's on your account answers that fine, whatever the username.

**Phase 2: authorization (GitHub's application logic).** The authenticated session runs `git-upload-pack '<org>/repo.git'`. Only now does GitHub ask **"may this credential read this org's repo?"** The org enforces SSO, so the credential must be either an SSO-authorized key or a certificate from the org's CA. The plain key is neither, so GitHub rejects it with the SAML error.

ssh can't go back after Phase 2 fails. Phase 1 succeeded, ssh has no idea a later check depends on which identity it used, and the remote command fails over an already-authenticated session. The certificate never gets offered.

Think of a building with two checkpoints. The front desk accepts any valid ID. The office door wants a company badge. You showed your personal ID at the front desk, got in, and the office door turned you away. Your badge was in your pocket the whole time, but nobody asks for a second ID once you're past the front desk.

### Why `ssh -T` gave a false positive

```
$ ssh -T org-<ORG_ID>@github.com
Hi <you>! You've successfully authenticated, but GitHub does not provide shell access.
```

This only exercises Phase 1. It proves *some* identity mapped to your user, not that the certificate was used or that SSO will pass. I misread this output during the investigation: it looked like proof that the certificate worked, but the plain key was enough to produce it. The test that actually means something is a real git operation against an org repo (`git ls-remote`), or `ssh -v` with an eye on which identity got `Server accepts key`.

`ssh -T` against GitHub also **always exits 1**, because "no shell access" counts as a failure. In a script with `set -o pipefail`, `ssh -T ... | grep -q 'successfully authenticated'` therefore never succeeds, even when the text matches. Capture the output and match on the text:

```sh
cert_ok() { [[ $(ssh -T org-<ORG_ID>@github.com 2>&1 || true) == *"successfully authenticated"* ]]; }
```

### Why the old laptop worked with the same config

The old laptop also offered the plain key first, and that's exactly what produced the confusion. Its clones succeeded anyway, so offer order looked like it couldn't be the cause.

The most likely explanation, which I didn't verify, is that the old laptop's plain key had been **SSO-authorized for the org** at some point, probably before the certificate setup existed. Then Phase 2 passed even with the "wrong" identity, and the config bug stayed hidden. A new laptop means a new hardware key that isn't SSO-authorized, so the bug finally showed up.

This kind of failure is common after migrations: **an old credential with extra grants hides a config bug, and a fresh credential without those grants exposes it.** You can check for the hidden grant on GitHub's SSH keys settings page, where an SSO-authorized key shows a "Configure SSO" dropdown listing the org.

## Part 3: proving it, then fixing it

### The decisive experiment: bypass the config

Comparing the two laptops kept giving ambiguous answers. What settled it was one test that removed the config from the picture:

```sh
GIT_SSH_COMMAND='ssh -F /dev/null -o IdentityAgent=~/path/to/agent.socket' \
  git ls-remote git@github.com:<org>/some-repo.git HEAD
```

- `-F /dev/null` means no config file, so there's no `IdentityFile` and nothing explicit to offer first.
- With only the agent, ssh offers identities in agent order, and the agent lists the **certificate first**.
- `GIT_SSH_COMMAND` makes git use that ssh invocation while still applying the `insteadOf` rewrite.

That printed a commit hash. Offering the certificate first works and offering the plain key first doesn't, so the order is the cause.

### The fix: keep `IdentityFile` away from org connections

```
Match host github.com user org-*
    IdentityAgent ~/path/to/agent.socket

Match !user org-*
    IdentityFile ~/.ssh/agent-key.pub
    IdentityAgent ~/path/to/agent.socket
```

`Match !user org-*` means "every connection whose SSH username isn't `org-*`". Match criteria can be negated with `!`. Org connections now have no explicit `IdentityFile`, so ssh offers agent identities in agent order and the certificate goes first. `git@github.com` connections still pin the plain key, so personal and public repos work as before.

Verify it with:

```sh
ssh -G org-<ORG_ID>@github.com | grep -i '^identityfile'   # the .pub must be gone (default id_* entries are harmless)
ssh -vT org-<ORG_ID>@github.com 2>&1 | grep -E 'Offering|accepts'   # first offer must be ECDSA-CERT
git ls-remote git@github.com:<org>/some-repo.git HEAD               # the real test
```

Another option is `CertificateFile` or an `IdentityFile` pointing at a saved copy of the certificate, but the certificate is short-lived and rotates, so a saved copy goes stale. Relying on agent order is simpler.

### A related trap: the commit signing key

Signing commits with SSH (`gpg.format = ssh`) uses `user.signingKey = "key::<public key>"`. It has to be the **plain** key, because GitHub verifies signatures against the keys on your account. `ssh-add -L` lists the certificate line first, so a script that takes "the first ECDSA line" picks up the certificate. Take the plain key explicitly; with this agent that's the last line:

```sh
git config --global user.signingKey "key::$(ssh-add -L | tail -1)"
```

### A related trap: setup scripts that copy dotfiles

My setup script copied a bundled `~/.ssh/config` onto the new machine on every run. After I fixed the live config by hand, running the script again quietly restored the broken one, and the "fix didn't work" report that followed cost another round of debugging. If a script owns a dotfile, fix it in the script's bundled copy, or the next run puts the bug back.

## Debugging toolkit

| Question | Command |
| --- | --- |
| What does the agent hold? | `ssh-add -L \| awk '{print $1}'` (look for `-cert-v01@openssh.com`) |
| Is the certificate valid, and for whom? | `ssh-add -L \| grep cert \| ssh-keygen -L -f /dev/stdin` |
| Which fingerprint is my key? | `ssh-add -L \| tail -1 \| ssh-keygen -lf /dev/stdin` |
| What does the config resolve to for this host and user? | `ssh -G user@host \| grep -i identityfile` |
| Which identity was offered and accepted? | `ssh -vT user@host 2>&1 \| grep -E 'Offering\|accepts'` |
| What URL and ssh command does git actually use? | `GIT_TRACE=1 git ls-remote <url> HEAD` |
| Which `insteadOf` rules apply, and where are they defined? | `git config --show-origin --get-regexp '^url\.'` |
| Is the config the problem? | `GIT_SSH_COMMAND='ssh -F /dev/null -o IdentityAgent=...' git ls-remote <url> HEAD` |

## Key takeaways

- **SSH authentication and GitHub authorization are separate phases.** ssh stops at the first identity the server accepts, and the org's SSO check happens afterwards, so the wrong-but-valid identity going first is a hard failure with no retry.
- **A certificate and its key share a fingerprint but are different credentials** with different grants. Debug output is the only place you see which one was used (`ECDSA` vs `ECDSA-CERT`).
- **`Host *` applies to everything**, and `IdentityFile` accumulates. An explicit `IdentityFile` jumps ahead of every agent identity in the offer order.
- **`ssh -T` "successfully authenticated" doesn't test authorization.** Test with a real git operation against the repo you care about.
- **Remove a variable to settle an argument.** `-F /dev/null` turned an ambiguous two-laptop comparison into one yes/no answer.
- **Migrations expose hidden grants.** If old hardware works and new hardware doesn't with identical config, look for something the old credential was given outside the config.

---

## Related Notes

- [[notes/AuthNZ/oauth-oidc-and-workload-identity|OAuth, OIDC & Workload Identity Federation]]: SSO, SAML and identity federation, the other half of why the org rejected the plain key
- [[notes/Networking/tls-1.3-handshake|TLS 1.3 Handshake]]: another protocol where authentication finishes before the application decides what you're allowed to do
