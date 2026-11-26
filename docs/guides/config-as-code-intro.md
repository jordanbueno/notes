# Getting started with config as code (OpenTofu + Ansible)

A hands-on introduction to managing servers as code: what config as code is, how OpenTofu and Ansible
divide the work, the syntax you'll read and write every day, how secrets fit in, and two small exercises
you can run locally without any cloud account.
Last updated: 2026-11-26

Related: [Infra toolchain on Debian / WSL](infra-toolchain.md) · [WSL2 with Debian](wsl2-debian-setup.md)

**Before you start:** install the tools from [Infra toolchain on Debian / WSL](infra-toolchain.md).

## 1. What config as code means

Instead of clicking through a cloud console and SSHing in to run commands, you write down **what should
exist** in text files, keep them in git, and let tools make reality match.

Four ideas carry most of the weight:

| Idea | Meaning | Why it matters |
|---|---|---|
| **Declarative** | You describe the end state ("nginx installed, port 443 open"), not the steps | The tool works out what to do from where things stand now |
| **Idempotent** | Running the same code twice changes nothing the second time | Safe to re-run; "changed=0" proves the box matches the code |
| **Desired state vs. drift** | The code is the desired state; any manual change on the box is drift | Tools can detect drift and put it back |
| **Reviewable** | Every change is a commit, often a PR, with a preview (`plan`, `--check --diff`) | You see what will happen before it happens, and can roll back |

Compare a shell script: `apt install nginx && echo "listen 443" >> nginx.conf` appends a duplicate line
every time it runs. A declarative tool says "this line should be in this file" and adds it only if it's
missing.

## 2. Who does what

```
┌────────────────────────────────────────────────────────────────┐
│ OpenTofu: the infrastructure (talks to cloud APIs)             │
│   VM / instance, cloud firewall, DNS records, storage buckets  │
│                                                                │
│   ┌──────────────────────────────────────────────────────┐     │
│   │ Ansible: the operating system (talks SSH to the VM)  │     │
│   │   packages, users, SSH config, ufw, Docker, files    │     │
│   │                                                      │     │
│   │   ┌────────────────────────────────────────────┐     │     │
│   │   │ The application's own config               │     │     │
│   │   │   compose files, app settings, its CLI     │     │     │
│   │   └────────────────────────────────────────────┘     │     │
│   └──────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

- **OpenTofu** is good at things that have an API and a lifecycle (create, update, destroy). It can't log in
  to a server.
- **Ansible** is good at the inside of a machine. It *can* call cloud APIs, but it keeps no record of what
  it created, so cleaning up and spotting drift are much harder than with OpenTofu.
- Use each for its own layer and hand off at the server's IP address and SSH access.

## 3. OpenTofu

### Core concepts

| Concept | What it is |
|---|---|
| **Provider** | A plugin that knows one API (`linode/linode`, `cloudflare/cloudflare`, `hashicorp/local`) |
| **Resource** | Something OpenTofu creates and owns: `resource "linode_instance" "web" { ... }` |
| **Data source** | Something that already exists that you only *read*: `data "cloudflare_zone" "main" { ... }` |
| **Variable** | An input (`var.region`), set in `terraform.tfvars`, `-var` or `TF_VAR_region` |
| **Output** | A value printed after apply, for people or other tools (`tofu output ip`) |
| **State** | OpenTofu's record mapping each resource in code to a real object ID. **Contains secrets**; keep it in a remote backend, never in git |
| **Plan** | The diff between code, state and reality: create (`+`), update in place (`~`), destroy (`-`), replace (`-/+`) |
| **Lock file** | `.terraform.lock.hcl` pins exact provider versions and checksums. Commit it |

### Syntax (HCL)

```hcl
# versions.tf: which providers, which versions
terraform {
  required_version = ">= 1.8.0"

  required_providers {
    linode = {
      source  = "linode/linode"
      version = "~> 4.9"          # any 4.x from 4.9 up, never 5.0
    }
  }
}

# Credentials come from the environment (LINODE_TOKEN), never from files in git
provider "linode" {}

# variables.tf: inputs with types, defaults and validation
variable "region" {
  type        = string
  default     = "us-ord"
  description = "Linode region"
}

variable "admin_cidrs" {
  type = list(string)
  validation {
    condition     = length(var.admin_cidrs) > 0
    error_message = "Give at least one admin CIDR, or you'll lock yourself out."
  }
}

# main.tf: resources.  Syntax: resource "<TYPE>" "<LOCAL NAME>" { arguments }
resource "linode_instance" "web" {
  label  = "web-1"
  region = var.region               # reference a variable
  type   = "g6-nanode-1"
  image  = "linode/debian12"
  tags   = ["managed-by-tofu"]

  lifecycle {
    prevent_destroy = true          # any plan that would destroy this fails
    ignore_changes  = [image]       # don't rebuild just because the image name changed
  }
}

# outputs.tf: reference another resource's attribute: <TYPE>.<NAME>.<ATTRIBUTE>
output "web_ip" {
  value = tolist(linode_instance.web.ipv4)[0]
}
```

Things to notice:

- Blocks are `type "label" "label" { key = value }`. Strings take double quotes; lists `[]`; maps `{}`.
- **References** (`var.region`, `linode_instance.web.ipv4`) are how OpenTofu works out ordering. If B
  references A, A gets created first. You rarely need `depends_on`.
- `${...}` interpolates inside strings: `"web-${var.region}"`.
- Every `.tf` file in a folder is loaded together. File names like `main.tf` and `variables.tf` are
  convention only.
- `count` and `for_each` make several copies of a resource. `locals { }` names intermediate values.

### Adopting things that already exist

Built something by hand first? Don't let OpenTofu make a second one. Write the resource block to match
it, then add an `import` block:

```hcl
import {
  to = linode_instance.web
  id = "12345678"               # the real Linode ID
}
```

`tofu plan` then shows **import** plus whatever differences remain between your code and reality. Adjust
the code until the plan shows only imports and harmless in-place updates. Then apply, and delete the import
block.

### The workflow

```bash
tofu init        # download providers, set up the backend (once per clone, again after provider changes)
tofu fmt         # format files
tofu validate    # syntax and type check, no API calls
tofu plan        # preview. READ IT
tofu apply       # shows the plan again, asks "yes", then makes the changes
tofu output      # print outputs
tofu state list  # what OpenTofu manages
tofu destroy     # tear everything down (prevent_destroy blocks it)
```

Reading a plan:

```
  # linode_instance.web will be updated in-place
  ~ resource "linode_instance" "web" {
      ~ label = "web-1" -> "web-01"
    }

  # linode_instance.web must be replaced
-/+ resource "linode_instance" "web" {
      ~ image = "linode/debian12" -> "linode/debian13" # forces replacement
    }
```

(This is the change the `ignore_changes = [image]` line in the example above exists to prevent.)

`~` is a safe edit. **`-/+` with "forces replacement" means destroy and recreate**: for a server, that
means a new disk and lost data. Stop and work out why before applying.

### OpenTofu rules of thumb

- Never edit state by hand. Use `tofu state mv`, `tofu state rm` or `import` blocks.
- Never run `apply` without reading the plan. In CI, apply the saved plan (`tofu plan -out=tfplan` then
  `tofu apply tfplan`) so what runs is exactly what you reviewed.
- Keep secrets out of `.tf` and `.tfvars` files. Provider tokens come from environment variables.
- Use a remote, versioned backend (S3-compatible bucket) with locking, so two applies can't collide and a
  bad apply can be rolled back.

## 4. Ansible

### Core concepts

| Concept | What it is |
|---|---|
| **Control node** | The machine you run Ansible from (your workstation/WSL). Nothing is installed on the servers except Python |
| **Inventory** | The list of hosts and groups to manage, plus per-host connection info |
| **Module** | One unit of work: `ansible.builtin.apt`, `ansible.builtin.copy`, `community.general.ufw`. Modules are idempotent |
| **Task** | One call to a module, with a name |
| **Play** | Tasks applied to a set of hosts |
| **Playbook** | A YAML file of one or more plays (`site.yml`) |
| **Role** | A reusable folder of tasks, handlers, templates and defaults (`roles/docker/`) |
| **Handler** | A task that runs only when notified by a change, once at the end of the play ("restart sshd if its config changed") |
| **Facts** | Info gathered from each host at the start (`ansible_distribution`, `ansible_memtotal_mb`) |
| **Collection** | A package of modules and roles (`community.general`), installed with `ansible-galaxy` from `requirements.yml` |

### Inventory

```yaml
# inventory/hosts.yml
all:
  children:
    web:
      hosts:
        web-1:
          ansible_host: 203.0.113.20
          ansible_user: deploy
```

### A playbook

```yaml
# site.yml
- name: Baseline for web servers
  hosts: web                    # group from the inventory
  become: true                  # sudo for every task in this play
  vars:
    packages: [curl, htop, fail2ban]

  tasks:
    - name: Install base packages
      ansible.builtin.apt:
        name: "{{ packages }}"  # Jinja2 variable
        state: present
        update_cache: true

    - name: Disable SSH password logins
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/10-hardening.conf
        content: |
          PasswordAuthentication no
          PermitRootLogin no
        mode: "0644"
      notify: Restart ssh       # runs the handler only if this task changed something

    - name: Allow HTTPS
      community.general.ufw:
        rule: allow
        port: "443"
        proto: tcp

    - name: Show memory (facts example)
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} has {{ ansible_memtotal_mb }} MB"
      when: ansible_memtotal_mb < 2048   # conditional

  handlers:
    - name: Restart ssh
      ansible.builtin.service:
        name: ssh
        state: restarted
```

Things to notice:

- It's YAML: indentation is structure, and `- ` starts a list item. Quote values that start with `{{` or
  look like numbers (`"0644"`, `"443"`).
- Use **fully qualified module names** (`ansible.builtin.copy`, not `copy`). ansible-lint enforces it.
- `{{ }}` is Jinja2: variables, filters (`{{ name | upper }}`, `{{ list | join(',') }}`), and facts.
- `state: present` / `absent` / `started` / `restarted` is declarative: say what should be true.
- `loop:` repeats a task over a list (`{{ item }}`), and `register:` saves a task's result to use later.

### Roles

When a playbook grows, split it into roles. Each folder has a fixed purpose:

```
roles/docker/
├── defaults/main.yml    # default variables (lowest precedence, meant to be overridden)
├── tasks/main.yml       # the tasks
├── handlers/main.yml    # handlers
├── templates/           # Jinja2 templates (*.j2) for the template module
└── files/               # static files for the copy module
```

```yaml
# site.yml using roles
- name: Web servers
  hosts: web
  become: true
  roles:
    - base
    - hardening
    - docker
```

### Where variables come from

From lowest to highest precedence (simplified; the full list has 20+ levels):

1. Role `defaults/main.yml`
2. Inventory `group_vars/all.yml`, then `group_vars/<group>.yml`
3. Inventory `host_vars/<host>.yml`
4. Play `vars:`
5. Role `vars/main.yml`
6. `-e` on the command line (always wins)

Rule of thumb: role defaults hold sane generic values, and `group_vars/` holds the values for *your*
environment.

### The workflow

```bash
ansible-galaxy collection install -r requirements.yml   # once per clone
ansible all -m ansible.builtin.ping                     # can I reach everything?
ansible-playbook site.yml --check --diff                # dry run: what WOULD change, with file diffs
ansible-playbook site.yml                               # do it
ansible-playbook site.yml --limit web-1 --tags ssh      # one host, only tasks tagged "ssh"
ansible-lint                                            # lint
```

The run ends with a recap. Run it a second time: if your roles are idempotent, it says `changed=0`.

```
PLAY RECAP *****************************************************
web-1 : ok=12  changed=0  unreachable=0  failed=0  skipped=1
```

### Ansible rules of thumb

- **Always run `--check --diff` first** on a server you care about. Some tasks (`command`, `shell`) can't
  predict their results in check mode and are skipped. Read what's left.
- **Prefer modules over `command`/`shell`.** Modules are idempotent; `shell: echo x >> file` is not. When you
  must use `command`, add `creates:` or `changed_when:` so re-runs report correctly.
- **Changing SSH or firewall config?** Keep a second SSH session open until a fresh login works.
- Lists like "authorized keys" can be **exclusive** (`exclusive: true`): anything not in the list gets
  removed, including the key you're using.

## 5. Secrets: SOPS + age

Code goes in git; secrets can't go there in plain text. SOPS encrypts the **values** in a YAML file and
leaves the **keys** readable, so diffs still show *which* secret changed:

```yaml
# secrets/infra.sops.yaml as committed
LINODE_TOKEN: ENC[AES256_GCM,data:Tr6x...,type:str]
sops:
  age:
    - recipient: age1qyq...
```

```yaml
# .sops.yaml at the repo root: who can decrypt which files
creation_rules:
  - path_regex: secrets/.*\.sops\.yaml$
    age: >-
      age1workstationkey...,
      age1backupkey...
```

Daily use:

```bash
sops secrets/infra.sops.yaml                          # opens decrypted in $EDITOR, re-encrypts on save
sops -d secrets/infra.sops.yaml                       # print decrypted (don't redirect it into the repo)
sops exec-env secrets/infra.sops.yaml 'tofu plan'     # run a command with the secrets as env vars
sops updatekeys secrets/infra.sops.yaml               # after changing recipients in .sops.yaml
```

- `sops exec-env` is how OpenTofu gets provider tokens: they exist only as environment variables for that
  one command and never touch disk.
- Ansible reads SOPS files directly with the `community.sops` collection (a `vars` plugin or the
  `community.sops.sops` lookup).
- age keys are explained in the [toolchain guide](infra-toolchain.md#6-age-key-for-sops). Back up the
  private key or lose the secrets.

## 6. Guardrails: pre-commit, linters, CI

A `.pre-commit-config.yaml` in the repo lists checks that run on every commit:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.0
    hooks:
      - id: gitleaks            # block committed secrets
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck
  - repo: https://github.com/tofuutils/pre-commit-opentofu
    rev: v2.1.0
    hooks:
      - id: tofu_fmt
```

```bash
pre-commit install           # once per clone: adds the git hook
pre-commit run --all-files   # run every hook against the whole repo
git commit --no-verify       # skip hooks (emergencies only; CI still runs them)
```

Hooks that fix files themselves (formatters, end-of-file fixers) fail the first time and pass on the
second run. `git add` the fixes and commit again. Run the same checks in CI so a skipped hook can't sneak
through.

## 7. Exercise 1: OpenTofu without a cloud account

The `local` and `random` providers manage files and random values, so you can practice the whole lifecycle
offline.

```bash
mkdir -p ~/src/sandbox/tofu-hello && cd ~/src/sandbox/tofu-hello
```

`main.tf`:

```hcl
terraform {
  required_providers {
    local  = { source = "hashicorp/local" }
    random = { source = "hashicorp/random" }
  }
}

variable "greeting" {
  type    = string
  default = "Hello"
}

resource "random_pet" "name" {
  length = 2
}

resource "local_file" "hello" {
  filename        = "${path.module}/hello.txt"
  content         = "${var.greeting} from ${random_pet.name.id}\n"
  file_permission = "0644"
}

output "pet" {
  value = random_pet.name.id
}
```

Then try each step and watch what the plan says:

1. `tofu init`, then `tofu plan` (2 to add), then `tofu apply`, then `cat hello.txt`.
2. Run `tofu plan` again: "No changes". That's idempotence.
3. `tofu apply -var greeting=Howdy`: `local_file` gets **replaced** (`-/+`), because that provider can't edit
   a file in place. `random_pet` is untouched.
4. Change `length = 2` to `3` and plan: `random_pet` is replaced, and because `hello.txt` references it, the
   file is replaced too. That's the dependency graph at work.
5. Delete `hello.txt` by hand and plan: OpenTofu notices the **drift** and plans to recreate it.
6. `tofu state list` and `tofu state show random_pet.name`: this is what lives in state.
7. `tofu destroy`.

## 8. Exercise 2: Ansible against your own machine

Ansible can manage `localhost` without SSH, which is enough to learn idempotence, templates, handlers and
check mode.

```bash
mkdir -p ~/src/sandbox/ansible-hello/templates && cd ~/src/sandbox/ansible-hello
```

`inventory.yml`:

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
      ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

`templates/motd.j2`:

```jinja
Managed by Ansible. Do not edit by hand.
Host: {{ inventory_hostname }}
OS:   {{ ansible_distribution }} {{ ansible_distribution_version }}
Owner: {{ owner }}
```

`site.yml`:

```yaml
- name: Practice play
  hosts: localhost
  vars:
    owner: "{{ lookup('env', 'USER') }}"
    demo_dir: "{{ lookup('env', 'HOME') }}/ansible-demo"

  tasks:
    - name: Create a directory
      ansible.builtin.file:
        path: "{{ demo_dir }}"
        state: directory
        mode: "0755"

    - name: Render a template
      ansible.builtin.template:
        src: motd.j2
        dest: "{{ demo_dir }}/motd.txt"
        mode: "0644"
      notify: Announce change

    - name: Make sure a setting line exists
      ansible.builtin.lineinfile:
        path: "{{ demo_dir }}/app.conf"
        line: "log_level = info"
        create: true
        mode: "0644"

  handlers:
    - name: Announce change
      ansible.builtin.debug:
        msg: "motd.txt changed, a real handler would restart a service here"
```

Then:

1. `ansible-playbook -i inventory.yml site.yml`: the tasks report `changed`, and the handler runs at the end.
2. Run it again: `changed=0`, and the handler **doesn't** run. Nothing changed, so nothing was notified.
3. Edit `templates/motd.j2` (add a line), then run with `--check --diff`: you see the file diff, but nothing
   is written.
4. Run it for real: only the template task changes, and the handler fires.
5. Append a line to `~/ansible-demo/app.conf` by hand and run again: `lineinfile` leaves your line alone
   (it only ensures its own line exists). Compare with `template`, which overwrites the whole file. Pick the
   module that matches how much of the file you want to own.
6. Run `ansible-lint site.yml` and fix anything it flags.
7. Clean up: `rm -rf ~/ansible-demo ~/src/sandbox`.

## 9. How a real repo is laid out

```
infra-repo/
├── infra/                 # OpenTofu: versions.tf, main.tf, variables.tf, outputs.tf, backend.tf
├── ansible/
│   ├── ansible.cfg
│   ├── requirements.yml   # pinned collections
│   ├── inventory/hosts.yml
│   ├── group_vars/all.yml # your environment's values
│   ├── site.yml
│   └── roles/<role>/...
├── secrets/*.sops.yaml    # encrypted only
├── scripts/               # small wrappers (install, drift check)
├── docs/runbooks/         # step-by-step procedures
├── docs/decisions/        # ADRs: why things are the way they are
├── .sops.yaml
└── .pre-commit-config.yaml
```

The day-to-day loop:

1. Branch → change code → `pre-commit` runs on commit.
2. `tofu plan` and/or `ansible-playbook --check --diff` → read the preview.
3. Apply → run the check again and confirm it's clean (`No changes` / `changed=0`).
4. PR → merge. The repo is the record of what's deployed.
5. Changed something by hand during an incident? Put it back into the code, or the next run undoes it.

## Cheat sheet

| Task | OpenTofu | Ansible |
|---|---|---|
| Set up a fresh clone | `tofu init` | `ansible-galaxy collection install -r requirements.yml` |
| Format / lint | `tofu fmt` / `tofu validate` | `ansible-lint` |
| Preview | `tofu plan` | `ansible-playbook site.yml --check --diff` |
| Apply | `tofu apply` | `ansible-playbook site.yml` |
| Target one thing | `-target=linode_instance.web` (sparingly) | `--limit host` / `--tags tag` |
| See what's managed | `tofu state list` | `ansible-inventory --graph` |
| Proof it matches the code | Plan says "No changes" | Recap says `changed=0` |

## Further reading

- OpenTofu docs: <https://opentofu.org/docs/>
- Ansible docs and module index: <https://docs.ansible.com/ansible/latest/>
- Ansible best practices ("Tips and tricks"): <https://docs.ansible.com/ansible/latest/tips_tricks/index.html>
- SOPS: <https://getsops.io/docs/>
- pre-commit: <https://pre-commit.com/>
