# Infra toolchain on Debian / WSL

The command-line tools for infrastructure-as-code work (Ansible, OpenTofu, SOPS, age, pre-commit and the
linters): what each one is for, how to install it on Debian (including WSL2), and how to check and update
it.
Last updated: 2026-11-26

Related: [WSL2 with Debian](wsl2-debian-setup.md) · [Getting started with config as code](config-as-code-intro.md) · [Multiple GitHub accounts over SSH](git-multiple-github-accounts-ssh.md)

## What each tool is for

| Tool | What it is | Why you need it |
|---|---|---|
| **OpenTofu** (`tofu`) | Open-source fork of Terraform, maintained under the Linux Foundation | Creates and changes cloud resources (VMs, firewalls, DNS) through provider APIs, from `.tf` files |
| **Ansible** (`ansible-playbook`) | Agentless configuration management over SSH | Configures what's *inside* a server: packages, users, files, services |
| **ansible-lint** | Linter for playbooks and roles | Catches bugs, deprecated syntax and bad practice before a run |
| **SOPS** (`sops`) | Encrypts the *values* in YAML/JSON/env files, leaving keys readable | Lets secrets live in git next to the code that uses them |
| **age** (`age`, `age-keygen`) | Small, modern file encryption; a simpler stand-in for GPG | The key SOPS encrypts and decrypts with |
| **pre-commit** | Framework that runs checks on every `git commit` | Blocks commits that leak secrets, break formatting or fail lint |
| **gitleaks** | Secret scanner | Finds tokens and keys in files and git history |
| **shellcheck** | Linter for shell scripts | Catches quoting bugs, unset variables and other bash footguns |
| **pipx** | Installs Python CLI tools, each in its own virtualenv, onto your PATH | Debian blocks `pip install` into the system Python; apt's Ansible lags behind |
| **git, curl, unzip, jq** | Basics | Cloning, downloading release binaries, unpacking, reading JSON |

OpenTofu and Ansible split the work: **OpenTofu handles everything outside the VM, Ansible everything
inside it.** The [config-as-code tutorial](config-as-code-intro.md) explains how they fit together.

## 1. apt packages

```bash
sudo apt update
sudo apt install -y git curl unzip jq ca-certificates gnupg age shellcheck pipx
pipx ensurepath   # adds ~/.local/bin to PATH; open a new shell afterwards
```

`age` and `shellcheck` are recent enough in Debian's repos. The rest below aren't packaged, or the
packaged versions are too old.

## 2. Ansible + ansible-lint + pre-commit (pipx)

```bash
pipx install --include-deps ansible-core
pipx inject --include-apps ansible-core ansible-lint
pipx install pre-commit
```

What those lines do:

- **`pipx install ansible-core`** creates a virtualenv at `~/.local/share/pipx/venvs/ansible-core` and puts
  its commands on your PATH.
- **`--include-deps`** also exposes commands that come from dependencies. You need it for the full `ansible`
  package (see below), and it does no harm with `ansible-core`.
- **`pipx inject ansible-core ansible-lint`** installs ansible-lint *into the same virtualenv*. ansible-lint
  imports Ansible's own libraries, so it has to see the exact version you run. In a separate venv it could
  pull a different ansible-core and lint against the wrong one.
- **`--include-apps`** puts the injected package's commands (`ansible-lint`) on your PATH. Without it
  the package is installed but the command isn't found.
- **`pipx install pre-commit`** puts pre-commit in its own venv. It's per-user and works in every repo.

**`ansible-core` or `ansible`?**

| | `ansible-core` | `ansible` |
|---|---|---|
| Contains | The engine plus `ansible.builtin` modules | `ansible-core` plus ~80 community collections |
| Size | Small | Large |
| Collections | You install what each repo lists in `requirements.yml` | Bundled, at whatever version came with the release |

Prefer `ansible-core` plus a `requirements.yml` per repo: the collection versions are then pinned in the
repo instead of depending on what happens to be installed on your workstation. If you already installed
`ansible`, that works too; still run `ansible-galaxy collection install -r requirements.yml` in each repo.
Either way, use the same name in `pipx inject` as in `pipx install` (`pipx list --short` shows which you have).

## 3. OpenTofu

The official installer adds OpenTofu's apt repo, so `apt upgrade` keeps it current:

```bash
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh -o install-opentofu.sh
chmod +x install-opentofu.sh
./install-opentofu.sh --install-method deb
rm install-opentofu.sh
```

Read the script before running it if you like. Manual apt steps:
<https://opentofu.org/docs/intro/install/deb/>.

## 4. SOPS and gitleaks (release binaries)

Neither is in Debian's repos. Download the latest release from GitHub:

```bash
# SOPS
SOPS_VERSION=$(curl -fsSL https://api.github.com/repos/getsops/sops/releases/latest | jq -r .tag_name)
curl -fsSLo /tmp/sops "https://github.com/getsops/sops/releases/download/${SOPS_VERSION}/sops-${SOPS_VERSION}.linux.amd64"
sudo install -m 0755 /tmp/sops /usr/local/bin/sops

# gitleaks (asset names drop the leading "v")
GL_VERSION=$(curl -fsSL https://api.github.com/repos/gitleaks/gitleaks/releases/latest | jq -r .tag_name)
curl -fsSL "https://github.com/gitleaks/gitleaks/releases/download/${GL_VERSION}/gitleaks_${GL_VERSION#v}_linux_x64.tar.gz" \
  | sudo tar -xz -C /usr/local/bin gitleaks
```

Swap `amd64`/`x64` for `arm64` on ARM machines.

pre-commit downloads its own copies of hook tools (including gitleaks and shellcheck) into
`~/.cache/pre-commit`, so hooks work without these binaries. Install them anyway if you want to run the
tools by hand (`gitleaks detect`, `shellcheck -x script.sh`).

## 5. Optional helpers

```bash
pipx install linode-cli   # look up Linode IDs, manage Object Storage
pipx install awscli       # S3 API calls against S3-compatible storage (e.g. bucket versioning)
```

## 6. age key for SOPS

SOPS looks for age keys in `~/.config/sops/age/keys.txt`:

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt   # prints "Public key: age1..."
chmod 600 ~/.config/sops/age/keys.txt
```

- The **public** key (`age1...`) goes into each repo's `.sops.yaml`. It isn't secret.
- The **private** key (the `AGE-SECRET-KEY-1...` line) can decrypt everything. Store a copy in your password
  manager straight away. Lose it and every secret encrypted to it is gone.
- Add a second recipient (a backup key kept only in the password manager) to `.sops.yaml`, so losing a
  machine doesn't lock you out.

## 7. Check everything

```bash
tofu version          # OpenTofu v1.x
ansible --version     # ansible [core 2.x] ... python version = ...
ansible-lint --version
pre-commit --version
sops --version
age --version
gitleaks version
shellcheck --version | head -2
```

## Updating

| Tool | Update with |
|---|---|
| apt packages, OpenTofu | `sudo apt update && sudo apt upgrade` |
| Ansible, ansible-lint, pre-commit | `pipx upgrade-all` (`--include-injected` to also upgrade ansible-lint) |
| SOPS, gitleaks | Re-run the download commands in section 4 |
| Hook versions inside a repo | `pre-commit autoupdate`, then commit `.pre-commit-config.yaml` |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ansible: command not found` after pipx install | `pipx ensurepath`, then open a new shell |
| `ansible-lint: command not found` but `pipx list` shows it injected | Injected without `--include-apps`: `pipx inject --include-apps --force ansible-core ansible-lint` (or `ansible`) |
| `error: externally-managed-environment` | You ran `pip install`; use `pipx` (Debian protects the system Python) |
| `ansible-lint` complains about a missing collection | `ansible-galaxy collection install -r requirements.yml` in the repo |
| `[WARNING]: Ansible is being run in a world writable directory ... ignoring ansible.cfg` | The repo is under `/mnt/c`; move it into `~` |
| `sops: failed to get the data key` / `no identity matched` | Your age key isn't in `~/.config/sops/age/keys.txt`, or its public key isn't a recipient in `.sops.yaml` (then `sops updatekeys <file>` from a machine that can decrypt) |
| `tofu: Unsupported argument "use_lockfile"` | OpenTofu older than 1.10; upgrade |
