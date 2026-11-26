# Multiple GitHub accounts over SSH (auth + commit signing)

Use two or more GitHub accounts (e.g. personal and work) from one machine. Each gets the right SSH key for
push/pull and the right identity and key for signed commits, picked automatically by folder.
Last updated: 2026-11-26

Related: [Bitwarden SSH agent in WSL](bitwarden-ssh-agent-wsl.md) · [WSL2 with Debian](wsl2-debian-setup.md) · [SSH keys in PowerShell 7](ssh-keys-pwsh.md)

## The problem

An SSH agent (Bitwarden, 1Password, ssh-agent) offers **every** key it holds, in an order you don't
control. GitHub picks the account from the first key it recognizes, so you sometimes push as the wrong
account. Servers with a low `MaxAuthTries` may also reject you before the right key is tried.

The fix: point SSH and git at **public key files** per account. Private keys stay in the agent, and SSH asks
it to sign with the key matching the `.pub` file.

## 1. Keys: one per account

- One SSH key item per account in your vault, named clearly: `GitHub – personal`, `GitHub – work`.
- A key can be both an **Authentication** and a **Signing** key on GitHub, so one key per account covers
  both. A key can only be an auth key on **one** GitHub account, though.
- Prefer separate auth and signing keys? See [Separate auth and signing keys](#separate-auth-and-signing-keys).
- Save each public key in `~/.ssh`. They aren't secret:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/github-personal.pub   # paste "ssh-ed25519 AAAA... comment"
nano ~/.ssh/github-work.pub
chmod 644 ~/.ssh/*.pub
```

On GitHub, for each account: **Settings → SSH and GPG keys → New SSH key**, add it once as
**Authentication Key** and again as **Signing Key**.

## 2. SSH config: one host alias per identity

`~/.ssh/config`:

```sshconfig
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/github-personal.pub
  IdentitiesOnly yes

Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/github-work.pub
  IdentitiesOnly yes

# Same idea for servers, so the agent doesn't burn MaxAuthTries on other keys
Host myserver
  HostName server.example.com
  User deploy
  IdentityFile ~/.ssh/myserver.pub
  IdentitiesOnly yes
```

`IdentitiesOnly yes` is what makes this work: SSH offers only the key matching that `.pub` file and still
gets the signature from the agent.

```bash
chmod 600 ~/.ssh/config
ssh -T github-personal   # Hi <personal-user>!
ssh -T github-work       # Hi <work-user>!
```

## 3. Git: identity by folder

Keep repos in separate trees, e.g. `~/src/personal/` and `~/src/work/`.

`~/.gitconfig`:

```ini
[gpg]
  format = ssh
[commit]
  gpgsign = true
[tag]
  gpgsign = true
[gpg "ssh"]
  allowedSignersFile = ~/.ssh/allowed_signers

[includeIf "gitdir:~/src/personal/"]
  path = ~/.gitconfig-personal
[includeIf "gitdir:~/src/work/"]
  path = ~/.gitconfig-work
```

`~/.gitconfig-personal`:

```ini
[user]
  name = Your Name
  email = you@example.com
  signingkey = ~/.ssh/github-personal.pub
[url "git@github-personal:"]
  insteadOf = git@github.com:
```

`~/.gitconfig-work`: same shape, with the work email, `~/.ssh/github-work.pub`, and `git@github-work:`.

- The `insteadOf` rewrite makes existing repos in each tree use the right SSH alias, even when the remote
  says `git@github.com:`.
- The commit email must be a **verified** email on that GitHub account, or commits won't show as Verified.

### Cloning

When you clone, the repo isn't inside the folder yet, so `includeIf` doesn't apply. Clone with the alias:

```bash
cd ~/src/work && git clone git@github-work:org/repo.git
```

## 4. Signing

- Git signs with `ssh-keygen -Y sign`. When `signingkey` is a `.pub` file, the signature comes from the agent,
  so with Bitwarden approvals on you approve each commit (or each burst, depending on the setting).
- Local verification (`git log --show-signature`) needs the allowed-signers file:

```bash
echo "you@example.com $(cat ~/.ssh/github-personal.pub)" >> ~/.ssh/allowed_signers
echo "you@work.example $(cat ~/.ssh/github-work.pub)"    >> ~/.ssh/allowed_signers
```

### Separate auth and signing keys

Some people keep a dedicated signing key per account, so a leaked or rotated auth key doesn't touch commit
history (and vice versa). The SSH side doesn't change; only git's signing settings do.

1. Make a second SSH key item in the vault (`GitHub signing – personal`) and save its public half:

   ```bash
   nano ~/.ssh/github-personal-signing.pub
   chmod 644 ~/.ssh/github-personal-signing.pub
   ```

2. Keep `IdentityFile` in `~/.ssh/config` on the **auth** key (`github-personal.pub`).
3. Point git at the **signing** key in `~/.gitconfig-personal`:

   ```ini
   [user]
     signingkey = ~/.ssh/github-personal-signing.pub
   ```

4. Put the **signing** key (not the auth key) in `allowed_signers`:

   ```bash
   echo "you@example.com $(cat ~/.ssh/github-personal-signing.pub)" >> ~/.ssh/allowed_signers
   ```

5. On GitHub, add the auth key as **Authentication Key** only and the signing key as **Signing Key** only.

With Bitwarden approvals on, `git push` prompts for the auth key item and `git commit` for the signing key
item. If a commit prompts for the auth key, `signingkey` points at the wrong file.

## 5. Check

```bash
cd ~/src/personal/some-repo
ssh -T github-personal       # Hi <personal-user>!
git config user.email        # personal email
git config user.signingkey   # github-personal.pub (or the -signing.pub)
git config gpg.format        # ssh
git commit --allow-empty -m "test signing"
git log --show-signature -1  # Good "git" signature for you@example.com
git reset --hard HEAD~1      # throw the test commit away
```

If those `git config` lines print nothing, the include isn't applying. See the last troubleshooting row.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pushes as the wrong account | Remote isn't using the alias: `git remote -v`; fix with `git remote set-url origin git@github-<acct>:owner/repo.git` or check the `insteadOf` include applies (`git config --show-origin --get-regexp url`) |
| `Permission denied (publickey)` with alias | `.pub` file doesn't match a key in the agent: compare `ssh-add -L` with the file |
| `error: Load key ... invalid format` when signing | `signingkey` path wrong or the file has a stray line break. One line, `ssh-ed25519 AAAA... comment` |
| Commits show **Unverified** on GitHub | Key not added as a **Signing Key**, or commit email not verified on that account |
| `includeIf` not applying | Path needs the trailing slash, and the repo must be under it: `git config --show-origin user.email` |
| `git config user.email` prints nothing | No global config yet, the repo isn't under an `includeIf` folder, or you're outside a repo (`gitdir:` only matches inside one). Move the repo or fix the path |
| Commit prompts for the **auth** key item | `signingkey` points at the auth `.pub`; set it to the signing key (see [Separate auth and signing keys](#separate-auth-and-signing-keys)) |
| `Good signature` locally but **Unverified** on GitHub, or the reverse | `allowed_signers` and the GitHub Signing Key are different keys; both should be the signing key |
