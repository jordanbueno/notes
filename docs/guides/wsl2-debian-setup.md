# WSL2 with Debian

Set up WSL2 with Debian on Windows 11 as a Linux workstation for infra work (Ansible, OpenTofu, git, ssh).
Last updated: 2026-11-26

Related: [Bitwarden SSH agent in WSL](bitwarden-ssh-agent-wsl.md) · [Multiple GitHub accounts over SSH](git-multiple-github-accounts-ssh.md) · [SSH keys in PowerShell 7](ssh-keys-pwsh.md)

## Why Debian

- If your servers run Debian, the workstation matches them: same package names, apt behavior, `/etc`
  layout and systemd units. Scripts and Ansible tasks behave the same locally as on the server.
- Minimal: no snap, no cloud-init.
- Ubuntu LTS works just as well. Pick it if you already have an Ubuntu WSL setup you like.

Ansible's control node doesn't run on native Windows, so WSL is the practical way to run it from a Windows
machine.

## 1. Install

PowerShell as Administrator:

```powershell
wsl --install --no-distribution     # WSL platform only, no distro yet
wsl --install --distribution Debian # or: wsl --install -d Debian
```

Reboot if prompted. The first launch asks for a Linux username and password (separate from your Windows
account).

Check the release you got. The Store image has sometimes lagged a release behind:

```bash
cat /etc/os-release
```

## 2. Base packages

The WSL Debian image is minimal. In particular it has **no OpenSSH client** (`ssh`, `ssh-add`,
`ssh-keygen` are missing until you install it).

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y openssh-client socat git curl ca-certificates gnupg unzip jq pipx
pipx ensurepath
```

## 3. Enable systemd

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

From PowerShell, restart WSL so the change takes effect:

```powershell
wsl --shutdown
```

Check with `systemctl is-system-running` (`running` or `degraded` both mean systemd is up).

Leave `appendWindowsPath` at its default (`true`). It lets WSL find Windows tools like `npiperelay.exe`
on the Windows PATH.

## 4. Work in the Linux filesystem

WSL opens in your Windows profile (`/mnt/c/Users/<you>`). Don't keep repos there:

- Permissions and line endings break (everything looks executable; CRLF sneaks in).
- git, Ansible and anything that touches many files run much slower across the `/mnt/c` boundary.

```bash
cd ~
mkdir -p ~/src/personal ~/src/work
```

From Windows, the Linux files are under `\\wsl$\Debian\home\<you>` in Explorer.

## 5. Editor

Install VS Code on Windows plus the **WSL** extension, then run `code .` from inside a WSL folder. The editor
runs on Windows and the extensions, terminal and git run in Linux.

## 6. Common tools

```bash
# Ansible + lint
pipx install --include-deps ansible-core
pipx inject ansible-core ansible-lint
pipx install pre-commit

# age (for SOPS)
sudo apt install -y age
```

OpenTofu and SOPS aren't in Debian's repos:

- OpenTofu: follow the official Debian/apt instructions at <https://opentofu.org/docs/intro/install/deb/>.
- SOPS: download the `linux.amd64` binary from <https://github.com/getsops/sops/releases> to
  `/usr/local/bin/sops` and `chmod +x` it.

## Handy commands (PowerShell)

| Command | What it does |
|---|---|
| `wsl -l -v` | List distros and WSL versions |
| `wsl --shutdown` | Stop all distros (applies `wsl.conf` changes) |
| `wsl --update` | Update the WSL platform |
| `wsl --set-default Debian` | Default distro for `wsl` |
| `wsl --export Debian debian.tar` | Back up a distro |
