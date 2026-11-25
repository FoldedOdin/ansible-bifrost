# ansible-bifrost

[![Cross-Distro Ansible CI](https://github.com/FoldedOdin/ansible-bifrost/actions/workflows/ci.yml/badge.svg)](https://github.com/FoldedOdin/ansible-bifrost/actions/workflows/ci.yml)
![Ansible](https://img.shields.io/badge/Ansible-2.16%2B-EE0000?logo=ansible&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)

An idempotent, distro-agnostic Infrastructure-as-Code (IaC) playbook that provisions a complete **Hyprland Wayland desktop, modern CLI toolkit, and container development environment** across **Fedora, Arch Linux, and Ubuntu/Debian** machines from a single codebase.

Built to eliminate drift across multi-OS virtual machines and workstations without duplicating playbooks or littering tasks with OS conditionals.

---

## Supported Distributions

| OS Family (`ansible_os_family`) | Tested Distributions | Package Manager | Docker Package | `fd` Binary Handling |
| :--- | :--- | :--- | :--- | :--- |
| **`RedHat`** | Fedora 41 – 45 (Rawhide) | `dnf` + COPR | `moby-engine` | Native `/usr/bin/fd` (`fd-find`) |
| **`Archlinux`** | Arch Linux / CachyOS | `pacman` | `docker` | Native `/usr/bin/fd` (`fd`) |
| **`Debian`** | Ubuntu / Lubuntu 24.04+ | `apt` | `docker.io` | Symlink `/usr/bin/fdfind` → `/usr/local/bin/fd` |

---

## Architectural Highlights

* **Zero Task-Level OS Spaghetti:** Distro-specific package names are decoupled into `vars/{{ ansible_facts['os_family'] }}.yml` and loaded dynamically at runtime via `include_vars`. All package tasks use `ansible.builtin.package` without `when: ansible_os_family == ...` clutter.
* **Hardware & VM-Aware Jinja2 Templates:** `hyprland.conf.j2` dynamically iterates over display definitions (`monitors`), injects Wayland software-cursor and renderer flags (`WLR_NO_HARDWARE_CURSORS=1`, `WLR_RENDERER_ALLOW_SOFTWARE=1`) when `is_virtual_machine: true`, configures NVIDIA Wayland environment variables when `gpu_vendor: nvidia`, and selects the right Polkit authentication agent per distro.
* **Dynamic User Resolution:** Inherits per-host SSH users (`workstation_user: "{{ ansible_user }}"`) automatically, allowing heterogeneous inventories with different usernames across VMs or bare-metal hosts.
* **CI-Safe Secrets Management:** Sensitive values (such as `ansible_become_pass` and Git email) live in an encrypted, gitignored `vault.yml`, paired with `vault.yml.example` and `default(omit)` fallbacks so GitHub Actions CI runs unattended without decryption keys.
* **Automated 3-Distro Molecule CI:** Every push and pull request runs `yamllint`, `ansible-lint`, and an **Ansible Molecule** matrix against ephemeral **Fedora, Ubuntu, and Arch Linux** Docker containers—verifying syntax, convergence, idempotence (`changed=0`), and post-deploy assertions.

---

## Repository Structure

```text
ansible-linux-workstation/
├── .github/
│   └── workflows/
│       └── ci.yml                       # Linting + 3-distro Molecule Docker matrix
├── inventories/
│   └── vms/
│       ├── group_vars/
│       │   └── workstations/
│       │       ├── vars.yml             # Shared display, GPU, and toolchain settings
│       │       └── vault.yml.example    # Template for encrypted Ansible Vault secrets
│       ├── host_vars/
│       │   ├── arch.yml                 # Arch-specific overrides
│       │   ├── fedora.yml               # Fedora-specific overrides
│       │   └── ubuntu.yml               # Ubuntu/Lubuntu-specific overrides
│       └── hosts.ini                    # Multi-VM inventory
├── molecule/
│   └── default/
│       ├── molecule.yml                 # Docker driver & systemd container config
│       ├── prepare.yml                  # Container user & package cache prep
│       └── verify.yml                   # Assertions for dotfiles, groups, and binaries
├── roles/
│   ├── base_packages/                   # CLI tools, Wayland stack, COPR, symlinks, zsh
│   ├── dotfiles_templating/             # Jinja2 configs for Hyprland, Kitty, and Git
│   └── dev_toolchain/                   # Docker Engine, log rotation, uv, Rust, VS Code
├── vars/
│   ├── Archlinux.yml                    # Arch package mappings
│   ├── Debian.yml                       # Debian/Ubuntu package mappings
│   └── RedHat.yml                       # Fedora/RHEL package mappings
├── ansible.cfg                          # Default inventory & SSH pipelining config
├── requirements.yml                     # Ansible Galaxy collections
├── site.yml                             # Master playbook
└── .yamllint                            # YAML linting rules
```

---

## What Gets Provisioned

### 1. `base_packages`
* **Wayland Desktop:** `hyprland`, `waybar`, `kitty`, `wofi`, and Polkit agent (`hyprpolkitagent` / `policykit-1-gnome`), enabling COPR (`sdegler/hyprland`) automatically on Fedora Rawhide.
* **Modern CLI Suite:** `git`, `curl`, `wget`/`wget2-wget`, `unzip`, `zsh`, `tmux`, `fd`, `ripgrep`, `bat`, `eza`, `btop`, `neovim`, and native build toolchains (`base-devel` / `build-essential` / `@development-tools`).
* **Cross-Distro Consistency:** Normalizes `fdfind` → `fd` and `batcat` → `bat` symlinks on Debian/Ubuntu and sets `zsh` as the user's default login shell.

### 2. `dotfiles_templating`
* **`~/.config/hypr/hyprland.conf`:** Dynamic monitor rules, VM/GPU environment variables, Polkit autostart, Catppuccin-inspired window borders, and core keybindings, with a handler running `hyprctl reload` when a Wayland session is active.
* **`~/.config/kitty/kitty.conf`:** Configurable Nerd Font family/size with a Catppuccin Mocha color scheme.
* **`~/.gitconfig`:** Pre-configured Git identity, `main` default branch, `nvim` editor, and `pull.rebase = true`.

### 3. `dev_toolchain`
* **Docker Engine & Compose:** Installs the distro-appropriate Docker engine and Compose plugin, creates the `docker` group, appends the workstation user, configures `/etc/docker/daemon.json` with `json-file` log rotation (`10m` max size, `3` files), and enables the `docker` systemd service.
* **Language & Editor Tooling:** Installs Astral's `uv` Python package manager (`~/.local/bin/uv`), optional `rustup` toolchain, and Visual Studio Code (`com.visualstudio.code`) via Flatpak/Flathub.

---

## Getting Started

### Prerequisites
* Control node with `ansible-core >= 2.16` and Python `3.11+`
* SSH key-based access to target hosts/VMs with `sudo` privileges

### 1. Clone and Install Dependencies
```bash
git clone [https://github.com/FoldedOdin/ansible-linux-workstation.git](https://github.com/FoldedOdin/ansible-linux-workstation.git)
cd ansible-linux-workstation
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Inventory & Vault
Update `inventories/vms/hosts.ini` with your target host IPs and SSH usernames:
```ini
[workstations]
fedora ansible_host=192.168.1.8 ansible_user=ulmoreth
ubuntu ansible_host=192.168.1.7 ansible_user=morvathe
arch   ansible_host=192.168.1.11 ansible_user=orveth
```

Create and encrypt your local Ansible Vault file for `sudo` passwords and private variables:
```bash
cp inventories/vms/group_vars/workstations/vault.yml.example \
   inventories/vms/group_vars/workstations/vault.yml
ansible-vault encrypt inventories/vms/group_vars/workstations/vault.yml
```

### 3. Verify Connectivity & Run
```bash
# Test SSH connectivity across all hosts
ansible workstations -m ping

# Provision all workstations
ansible-playbook site.yml --ask-vault-pass

# Re-run to verify idempotence (changed=0)
ansible-playbook site.yml --ask-vault-pass
```

### Targeted Execution with Tags
Run specific layers independently using tags:
```bash
ansible-playbook site.yml --tags packages --ask-vault-pass
ansible-playbook site.yml --tags dotfiles --ask-vault-pass
ansible-playbook site.yml --tags dev --ask-vault-pass
```

---

## Linting & Molecule Testing

To run the same validation checks executed in GitHub Actions locally:

```bash
pip install ansible-lint yamllint molecule molecule-plugins[docker]

# Static linting & syntax verification
yamllint .
ansible-lint
ansible-playbook site.yml --syntax-check

# Full Molecule lifecycle against a specific distro image
MOLECULE_DISTRO_IMAGE=geerlingguy/docker-fedora41-ansible:latest molecule test
MOLECULE_DISTRO_IMAGE=geerlingguy/docker-ubuntu2404-ansible:latest molecule test
MOLECULE_DISTRO_IMAGE=geerlingguy/docker-archlinux-ansible:latest molecule test
```

---

## License

Distributed under the MIT License. See `LICENSE` for more information.
