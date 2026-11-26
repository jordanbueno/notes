# SSH Keys in PowerShell 7

A beginner-friendly guide to working with SSH keys on Windows from PowerShell 7 (pwsh) in Windows Terminal: creating a new key, adding a key you already have, loading it into the agent, putting it on servers, and fixing the common problems.

Everything here uses the **OpenSSH client built into Windows**, so nothing extra needs to be installed.

---

## Contents

1. [How SSH keys work](#1-how-ssh-keys-work)
2. [Check your setup](#2-check-your-setup)
3. [Create a new key](#3-create-a-new-key)
4. [Add a private key you already have](#4-add-a-private-key-you-already-have)
5. [Lock down key permissions](#5-lock-down-key-permissions)
6. [Set up ssh-agent](#6-set-up-ssh-agent)
7. [Put your public key on a server](#7-put-your-public-key-on-a-server)
8. [The SSH config file](#8-the-ssh-config-file)
9. [Known hosts](#9-known-hosts)
10. [Use your key with Git and GitHub](#10-use-your-key-with-git-and-github)
11. [Managing keys](#11-managing-keys)
12. [Troubleshooting](#12-troubleshooting)
13. [Cheat sheet](#13-cheat-sheet)

---

## 1. How SSH keys work

An SSH key is a **pair** of files:

| File | Example name | Share it? | What it does |
|---|---|---|---|
| Private key | `id_ed25519` | **Never** | Proves you are you. Stays on your machine. |
| Public key | `id_ed25519.pub` | Yes, freely | Goes on every server or service you want to log in to. |

When you connect, the server checks that you hold the private key that matches a public key it has on file. No password crosses the network.

Your keys live in the `.ssh` folder in your user profile:

```
C:\Users\<you>\.ssh\
├── id_ed25519          private key
├── id_ed25519.pub      public key
├── config              optional per-host settings (section 8)
└── known_hosts         servers you've connected to before (section 9)
```

> **PowerShell shorthand:** `~` and `$HOME` both mean `C:\Users\<you>`, so `~\.ssh` is your SSH folder.

**Which key type?** Use **ed25519** for new keys: it's short, fast, and secure. Only fall back to RSA (`-t rsa -b 4096`) for old systems that don't support ed25519.

---

## 2. Check your setup

Run these in a Windows Terminal tab that's open to **PowerShell**. That's pwsh 7, not "Windows PowerShell" 5.1.

**Confirm you're in pwsh 7:**
```powershell
$PSVersionTable.PSVersion
```
Expect `7.x.x`.

**Confirm the OpenSSH client is installed:**
```powershell
Get-Command ssh, ssh-keygen, ssh-add
```
Each should resolve to a path under `C:\Windows\System32\OpenSSH\`. If they're missing, install the client from an **elevated** pwsh:
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

**Make sure your `.ssh` folder exists:**
```powershell
Test-Path ~\.ssh
```
If it prints `False`, create it:
```powershell
New-Item -ItemType Directory -Path ~\.ssh
```

---

## 3. Create a new key

```powershell
ssh-keygen -t ed25519 -C "you@example.com"
```

| Part | Meaning |
|---|---|
| `-t ed25519` | Key type |
| `-C "..."` | A comment stored in the public key so you can tell keys apart later. Your email or `user@machine` is typical. |

`ssh-keygen` then asks two things:

1. **File location.** Press **Enter** to accept the default (`~\.ssh\id_ed25519`), or type a full path to give the key its own name, e.g. `C:\Users\<you>\.ssh\id_ed25519_homelab`.
2. **Passphrase.** This encrypts the private key file, so a stolen copy is useless without it. Use one. The agent (section 6) remembers it, so you won't be typing it constantly. Press Enter twice only if you deliberately want no passphrase, e.g. for automation.

To do it all in one line with a custom name:
```powershell
ssh-keygen -t ed25519 -C "you@example.com" -f "$HOME\.ssh\id_ed25519_homelab"
```

**Check the result:**
```powershell
Get-ChildItem ~\.ssh
```
You should see both the private key and its `.pub` file.

**View your public key** (this is what you'll paste into servers and websites):
```powershell
Get-Content ~\.ssh\id_ed25519.pub
```

> **One key or many?** One key per machine is common and fine. Separate keys per purpose (work, homelab, GitHub) make it easier to revoke one without touching the others. Either works.

---

## 4. Add a private key you already have

Use this when the key already exists somewhere else: another computer, a password manager, a vault, an email from a teammate. You need it saved as a file in `~\.ssh`.

### Option A: The key is already a file

Copy it into place:
```powershell
Copy-Item -Path "C:\path\to\somekey" -Destination "$HOME\.ssh\id_mykey"
```
Then skip to [Step 6 below](#step-6-check-it-and-recreate-the-public-key-if-needed).

### Option B: The key is text you can copy (clipboard method)

**Step 1: Choose a filename and store it in a variable.**
```powershell
$key = "$HOME\.ssh\id_mykey"
```
This doesn't create anything. It just saves the path in `$key` so you don't retype it. Type `$key` and press Enter to see the stored value.

**Step 2: Copy the whole key.** Include the first line, which starts with `-----BEGIN`, the last line, which starts with `-----END`, and everything between them.

**Step 3: Check what's on the clipboard.**
```powershell
Get-Clipboard
```
You should see the full key, BEGIN line to END line. If part is missing, copy it again.

**Step 4: Save the clipboard into the file.**
```powershell
Get-Clipboard -Raw | Set-Content -Path $key
```
- `Get-Clipboard -Raw` reads the clipboard as a single block of text, with its line breaks intact.
- `|` (pipe) passes that text to the next command.
- `Set-Content -Path $key` writes it to the file.

In pwsh 7, `Set-Content` writes plain UTF-8 (no BOM) and ends the file with a newline, which is exactly what OpenSSH expects.

> **Avoid Notepad and Windows PowerShell 5.1 for this.** Both can save the file with a BOM or the wrong encoding, which gives `invalid format` / `error in libcrypto` errors later.

**Step 5: Check the file.**
```powershell
Get-Content $key
```
It should match what you saw in Step 3.

#### Step 6: Check it and recreate the public key if needed

First lock the file down ([section 5](#5-lock-down-key-permissions)), because ssh may refuse to read a key with loose permissions. Then:
```powershell
ssh-keygen -y -f $key
```
- **Prints one line starting with `ssh-ed25519` or `ssh-rsa`:** the key is valid, and that line is its public key.
- **Asks for a passphrase:** the key is encrypted. Enter it.
- **`invalid format`:** the copy was incomplete or mangled. Redo Option B.

If you don't have the matching `.pub` file, create it from the private key:
```powershell
ssh-keygen -y -f $key | Set-Content "$key.pub"
```

### Converting a PuTTY key (`.ppk`)

OpenSSH can't read PuTTY's `.ppk` format. Open the file in **PuTTYgen** → **Conversions** → **Export OpenSSH key**, save it into `~\.ssh`, then continue from Step 6.

---

## 5. Lock down key permissions

Windows OpenSSH refuses to use a private key that other accounts can read, and reports:
```
WARNING: UNPROTECTED PRIVATE KEY FILE!
```
Keys created by `ssh-keygen` get the right permissions automatically. Keys you **copy or create yourself** usually need fixing.

Run these one at a time (with `$key` set to your key path, as in section 4):

```powershell
$me = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
```
This stores your full account name (e.g. `DESKTOP-ABC\you`, `AzureAD\you`, or `DOMAIN\you`). Using the full name matters on domain-joined and Entra-joined machines.

```powershell
icacls $key /inheritance:r
```
This stops the file from inheriting permissions from the `.ssh` folder and removes the inherited entries.

```powershell
icacls $key /grant:r "${me}:(F)"
```
This gives **only you** full control.

```powershell
icacls $key
```
This lists who has access now. You should see just your account, and possibly `NT AUTHORITY\SYSTEM` and `BUILTIN\Administrators`, which are fine.

### Reusable function

Paste this into your profile (`notepad $PROFILE`) to get a one-word fix for any key:

```powershell
function Set-SshKeyPermission {
    param([Parameter(Mandatory)][string]$Path)
    $me = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
    icacls $Path /inheritance:r | Out-Null
    icacls $Path /grant:r "${me}:(F)" | Out-Null
    icacls $Path
}
```
Usage: `Set-SshKeyPermission ~\.ssh\id_mykey`

---

## 6. Set up ssh-agent

The agent holds your unlocked keys in the background. You enter a passphrase once, and every later `ssh` command uses the key automatically. On Windows the agent is a **service** that's disabled by default.

**One-time setup (elevated pwsh: right-click Terminal → Run as administrator):**
```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic -PassThru | Start-Service
```
This sets the service to start with Windows, then starts it now.

**Check it's running** (any pwsh window):
```powershell
Get-Service ssh-agent
```
Expect `Status: Running`.

**Add a key** (normal, non-admin pwsh):
```powershell
ssh-add ~\.ssh\id_ed25519
```
Running `ssh-add` with no path adds the default key names (`id_ed25519`, `id_rsa`, …).

**List loaded keys:**
```powershell
ssh-add -l
```

**Remove keys from the agent:**
```powershell
ssh-add -d ~\.ssh\id_ed25519   # one key
ssh-add -D                     # all keys
```

> **Windows difference:** unlike Linux and macOS, the Windows agent **remembers added keys across reboots** (stored securely in your user profile). You add each key once, not every session.

> **Using a password manager as the agent instead?** Bitwarden (and 1Password) can hold your keys in the vault and serve the same agent pipe, but only one program can own it, so the Windows `ssh-agent` service must be **disabled**. See [Bitwarden SSH agent in WSL](bitwarden-ssh-agent-wsl.md), which also covers using those keys from WSL.

---

## 7. Put your public key on a server

The server needs your **public** key (`.pub`) in the right file. Never copy the private key to a server.

### Linux servers

Windows doesn't ship `ssh-copy-id`, so pipe the key over instead. You'll log in with your password one last time:

```powershell
Get-Content ~\.ssh\id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

| Piece | What it does |
|---|---|
| `Get-Content ~\.ssh\id_ed25519.pub` | Reads your public key |
| `\| ssh user@server "..."` | Sends it to the server and runs the quoted Linux commands there |
| `mkdir -p ~/.ssh && chmod 700 ~/.ssh` | Makes sure the server's `.ssh` folder exists with the right permissions |
| `cat >> ~/.ssh/authorized_keys` | **Appends** your key (`>>`, so existing keys aren't overwritten) |
| `chmod 600 ...` | Sets the permissions sshd requires |

**Test it:**
```powershell
ssh user@server
```
If it logs you in without a password prompt (or only asks for your key passphrase), it worked.

### Windows servers (OpenSSH Server)

Which file the key goes in depends on whether the account is an **administrator** on that server:

| Account type | File on the server |
|---|---|
| Standard user | `C:\Users\<user>\.ssh\authorized_keys` |
| Member of local Administrators | `C:\ProgramData\ssh\administrators_authorized_keys` |

For an admin account, in an elevated pwsh **on the server**, append the public key and set the permissions it requires:
```powershell
$path = "C:\ProgramData\ssh\administrators_authorized_keys"
Add-Content -Path $path -Value "ssh-ed25519 AAAA...your-public-key... you@example.com"
icacls $path /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"
```
If that file's permissions are wrong, sshd silently ignores it, so the `icacls` line is not optional.

### Websites and web UIs (GitHub, GitLab, Proxmox, cloud consoles)

Copy the public key to the clipboard and paste it into the site's "SSH keys" page:
```powershell
Get-Content ~\.ssh\id_ed25519.pub | Set-Clipboard
```

---

## 8. The SSH config file

`~\.ssh\config` saves per-host settings, so `ssh myserver` replaces `ssh -i ~\.ssh\id_mykey -p 2222 admin@192.168.1.50`.

Create or edit it:
```powershell
notepad ~\.ssh\config
```
> Save it with **no file extension**. If Notepad saves `config.txt`, rename it with `Rename-Item ~\.ssh\config.txt config`.

Example:
```
# Applies to every host unless overridden below
Host *
    AddKeysToAgent yes
    ServerAliveInterval 60

Host myserver
    HostName 192.168.1.50
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_mykey
    IdentitiesOnly yes

Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
```

| Setting | Meaning |
|---|---|
| `Host` | The nickname you type (`ssh myserver`). `*` matches everything. |
| `HostName` | The real IP or DNS name |
| `User` | The username to log in as |
| `Port` | The port, if not 22 |
| `IdentityFile` | Which private key to use. Forward slashes are fine on Windows. |
| `IdentitiesOnly yes` | Offer **only** that key. This prevents "Too many authentication failures" when the agent holds several keys. |
| `AddKeysToAgent yes` | Adds a key to the agent automatically the first time you use it |
| `ServerAliveInterval 60` | Sends a keep-alive every 60 seconds so idle sessions don't drop |

Order matters: ssh uses the **first** value it finds for each setting, so put specific hosts before broad patterns, or keep `Host *` to settings no specific host overrides.

---

## 9. Known hosts

The first time you connect to a server, ssh shows its fingerprint:
```
The authenticity of host '192.168.1.50' can't be established.
ED25519 key fingerprint is SHA256:abc123...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```
Type `yes`, and the server's key is saved to `~\.ssh\known_hosts`. Next time, ssh checks the server still has the same key. This protects you from connecting to an impostor.

If a server is **rebuilt** (reinstalled VM, new IP owner), its key changes and ssh blocks you with:
```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```
If you **know** the server was rebuilt, remove the old entry and reconnect:
```powershell
ssh-keygen -R 192.168.1.50
```
If you did **not** expect a change, stop and find out why before connecting.

---

## 10. Use your key with Git and GitHub

**Add your public key to GitHub:** GitHub → Settings → SSH and GPG keys → New SSH key → paste the output of:
```powershell
Get-Content ~\.ssh\id_ed25519.pub | Set-Clipboard
```

**Test it:**
```powershell
ssh -T git@github.com
```
Expect: `Hi <username>! You've successfully authenticated...`

**Make Git use Windows OpenSSH.** Git for Windows ships its **own** ssh, which can't see the Windows agent, so it keeps asking for your passphrase. Point Git at the built-in one:
```powershell
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

**Clone over SSH** (use the SSH URL, not HTTPS):
```powershell
git clone git@github.com:<user>/<repo>.git
```

---

## 11. Managing keys

| Task | Command |
|---|---|
| Show a key's fingerprint | `ssh-keygen -lf ~\.ssh\id_ed25519.pub` |
| Add or change a passphrase | `ssh-keygen -p -f ~\.ssh\id_ed25519` |
| Recreate a missing `.pub` | `ssh-keygen -y -f ~\.ssh\id_ed25519 \| Set-Content ~\.ssh\id_ed25519.pub` |
| Copy a public key to the clipboard | `Get-Content ~\.ssh\id_ed25519.pub \| Set-Clipboard` |
| List everything in `.ssh` | `Get-ChildItem ~\.ssh` |

**Retiring a key:** remove its public key line from each server's `authorized_keys` and each website's SSH key list, run `ssh-add -d` on it, then delete both files.

**Backing up keys:** store the private key in a password manager (most support SSH keys or secure notes) rather than in cloud-synced folders or email. If you lose a private key, there's no recovery. You generate a new one and redistribute the public key.

---

## 12. Troubleshooting

**First step for any problem: verbose mode.**
```powershell
ssh -v user@server
```
This shows which config it read, which keys it offered, and where it failed. Add more `v`s (`-vvv`) for more detail.

| Error | Likely cause | Fix |
|---|---|---|
| `UNPROTECTED PRIVATE KEY FILE` / `bad permissions` | Others can read the key | [Section 5](#5-lock-down-key-permissions) |
| `invalid format` / `error in libcrypto` | Incomplete copy, wrong encoding, BOM, or missing final newline | Redo [section 4](#4-add-a-private-key-you-already-have) with the clipboard method in pwsh 7 |
| `Permission denied (publickey)` | The server doesn't have your public key, it's in the wrong file, or the wrong key was offered | Check `authorized_keys` on the server (for Windows admins, `administrators_authorized_keys`). Use `ssh -v` to see which key was offered. Add `IdentityFile` in your config. |
| `Too many authentication failures` | The agent offered too many keys before the right one | Add `IdentitiesOnly yes` and `IdentityFile` for that host in your config |
| `Error connecting to agent: No such file or directory` | The ssh-agent service isn't running | [Section 6](#6-set-up-ssh-agent) one-time setup |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | The server's host key changed | `ssh-keygen -R <host>` if expected ([section 9](#9-known-hosts)) |
| Git keeps asking for the passphrase | Git is using its bundled ssh | Set `core.sshCommand` ([section 10](#10-use-your-key-with-git-and-github)) |
| `Connection refused` / `timed out` | Network, firewall, or sshd not running, not a key problem | Check the server is up and listening: `Test-NetConnection <host> -Port 22` |

---

## 13. Cheat sheet

```powershell
# Create a key
ssh-keygen -t ed25519 -C "you@example.com"

# Import a key from the clipboard
$key = "$HOME\.ssh\id_mykey"
Get-Clipboard -Raw | Set-Content -Path $key

# Lock down permissions
$me = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
icacls $key /inheritance:r
icacls $key /grant:r "${me}:(F)"

# Validate the key / recreate the .pub
ssh-keygen -y -f $key | Set-Content "$key.pub"

# Agent (setup is a one-time admin command)
Get-Service ssh-agent | Set-Service -StartupType Automatic -PassThru | Start-Service
ssh-add $key
ssh-add -l

# Send the public key to a Linux server
Get-Content "$key.pub" | ssh user@server "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Copy the public key for a website
Get-Content "$key.pub" | Set-Clipboard

# Debug a connection
ssh -v user@server

# Forget a rebuilt server's host key
ssh-keygen -R <host>
```
