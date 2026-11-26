# Guides

Step-by-step how-tos. One topic per file, lowercase-hyphenated names (e.g. `intune-win32-packaging.md`). Group into subfolders (`intune/`, `proxmox/`, …) once a topic has ~5+ files.

- [SSH keys in PowerShell 7](ssh-keys-pwsh.md): create, import, agent, servers, config, troubleshooting
- [WSL2 with Debian](wsl2-debian-setup.md): install, systemd, filesystem layout, infra tooling
- [Infra toolchain on Debian / WSL](infra-toolchain.md): what OpenTofu, Ansible, SOPS, age, pre-commit and the linters are for; install, check, update
- [Getting started with config as code](config-as-code-intro.md): tutorial on OpenTofu + Ansible concepts, syntax, secrets and hands-on exercises
- [Bitwarden SSH agent in WSL](bitwarden-ssh-agent-wsl.md): vault-held SSH keys from Windows and WSL via `npiperelay` + `socat`
- [Multiple GitHub accounts over SSH](git-multiple-github-accounts-ssh.md): per-account auth and commit signing, picked by folder
