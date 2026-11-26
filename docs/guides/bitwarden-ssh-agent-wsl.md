# Bitwarden SSH agent in WSL

Keep SSH private keys in your Bitwarden vault and use them from Windows **and** WSL, with a Bitwarden
approval prompt on use. No private key files on disk.
Last updated: 2026-11-26

Related: [WSL2 with Debian](wsl2-debian-setup.md) · [Multiple GitHub accounts over SSH](git-multiple-github-accounts-ssh.md) · [SSH keys in PowerShell 7](ssh-keys-pwsh.md)

## How it works

```
ssh (WSL) → $SSH_AUTH_SOCK (unix socket) → socat → npiperelay.exe → \\.\pipe\openssh-ssh-agent → Bitwarden
```

On Windows, the Bitwarden desktop app serves the same named pipe the built-in OpenSSH agent uses, so
Windows `ssh` works directly. WSL can't talk to a Windows named pipe, so `socat` + `npiperelay` bridge a
Linux socket to it.

The same bridge works for 1Password's agent or the plain Windows OpenSSH agent, since they all use that pipe.

## 1. Windows: hand the pipe to Bitwarden

Only one program can own the pipe. Disable the built-in agent (PowerShell as Administrator). This undoes
the agent setup in [SSH keys in PowerShell 7 §6](ssh-keys-pwsh.md#6-set-up-ssh-agent) if you followed it:

```powershell
Stop-Service ssh-agent
Set-Service ssh-agent -StartupType Disabled
```

In the Bitwarden desktop app:

1. **Settings → Enable SSH agent**.
2. Set **Ask for authorization when using SSH agent** to taste. Keeping approvals on is a good idea:
   a prompt on each use acts like a 2FA push.
3. Add keys as **SSH key** items, either generated in Bitwarden or imported. Imports must be OpenSSH or PKCS#8
   format (`ssh-keygen` output is fine; PuTTY `.ppk` isn't).

Verify from PowerShell:

```powershell
Get-ChildItem \\.\pipe\ | Where-Object Name -like '*ssh*'   # openssh-ssh-agent
ssh-add -l                                                   # your vault keys
```

## 2. Windows: install npiperelay

```powershell
winget install albertony.npiperelay
```

(albertony's fork is the maintained one.) winget puts it on the Windows PATH, which WSL inherits.

> **Gotcha:** winget's portable apps are symlinks in `%LOCALAPPDATA%\Microsoft\WinGet\Links`, and WSL
> sometimes can't execute Windows symlinks. If `npiperelay.exe -h` fails from WSL but works in PowerShell,
> copy the real exe somewhere plain and use the full path in the script below:
>
> ```powershell
> New-Item -ItemType Directory -Force C:\tools | Out-Null
> Get-ChildItem "$env:LOCALAPPDATA\Microsoft\WinGet\Packages" -Recurse -Filter npiperelay.exe |
>   Select-Object -First 1 | Copy-Item -Destination C:\tools\
> ```
> Then use `/mnt/c/tools/npiperelay.exe` instead of `npiperelay.exe`.

## 3. WSL: the bridge

```bash
sudo apt install -y openssh-client socat
npiperelay.exe -h | head -1    # should print a version
```

Add to `~/.bashrc`:

```bash
# Bitwarden (Windows) SSH agent via named-pipe bridge
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
if ! ss -a 2>/dev/null | grep -q "$SSH_AUTH_SOCK"; then
  rm -f "$SSH_AUTH_SOCK"
  mkdir -p "$HOME/.ssh" && chmod 700 "$HOME/.ssh"
  ( setsid socat UNIX-LISTEN:"$SSH_AUTH_SOCK",fork \
      EXEC:"npiperelay.exe -ei -s //./pipe/openssh-ssh-agent",nofork & ) >/dev/null 2>&1
fi
```

- The `ss` check means only one relay runs no matter how many terminals you open.
- `setsid` keeps the relay alive after the terminal that started it closes.

## 4. Verify

```bash
exec bash -l
ssh-add -l                # same keys as Windows
ssh -T git@github.com     # Bitwarden prompts, then "Hi <user>!"
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `ssh-add: command not found` | WSL Debian ships without the OpenSSH client: `sudo apt install openssh-client` |
| `error fetching identities: communication with agent failed` | The WSL half works but the Windows half doesn't. Most often a **stale relay**: started before npiperelay was installed or on PATH, and the `ss` guard keeps it from restarting. Fix: `pkill -f 'socat UNIX-LISTEN'; rm -f ~/.ssh/agent.sock; exec bash -l` |
| `Could not open a connection to your authentication agent` | Relay isn't running. Same fix as above |
| `The agent has no identities` | Bridge works; vault is locked or has no SSH key items |
| `npiperelay.exe: command not found` / `Permission denied` | Not on PATH or winget symlink issue. Use the full path to a copied exe (step 2) |
| `Too many authentication failures` | The agent offered every vault key and the server's `MaxAuthTries` ran out. Pin a key per host with `IdentitiesOnly` (see [Multiple GitHub accounts over SSH](git-multiple-github-accounts-ssh.md#2-ssh-config-one-host-alias-per-identity)) |
| Windows `ssh-agent` won't start after leaving Bitwarden | Bitwarden can keep the pipe even with its agent turned off. Quit Bitwarden fully first |

To see the real error, run the relay in the foreground:

```bash
pkill -f 'socat UNIX-LISTEN'; rm -f ~/.ssh/agent.sock
socat -d -d UNIX-LISTEN:$HOME/.ssh/agent.sock,fork \
  EXEC:"npiperelay.exe -ei -s //./pipe/openssh-ssh-agent",nofork
# second terminal:
SSH_AUTH_SOCK=~/.ssh/agent.sock ssh-add -l
```

## Notes

- Bitwarden must be running and unlocked. A locked vault means SSH fails until you unlock it.
- If you replace keys, update the public key everywhere it's authorized (servers, GitHub, any
  config-as-code that manages `authorized_keys` exclusively) **before** relying on the new key.
