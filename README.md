# dev-genesis

<p align="center">
  <strong>Personal workstation automation powered by Ansible.</strong>
</p>

<p align="center">
  <img alt="Ansible" src="https://img.shields.io/badge/Ansible-automation-black?logo=ansible&logoColor=white">
  <img alt="Fedora" src="https://img.shields.io/badge/Fedora-primary-51A2DA?logo=fedora&logoColor=white">
  <img alt="Debian" src="https://img.shields.io/badge/Debian-supported-A81D33?logo=debian&logoColor=white">
  <img alt="KVM" src="https://img.shields.io/badge/KVM%2Flibvirt-virtualization-6E40C9">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green">
</p>

---

## Overview

`dev-genesis` is my Ansible project for rebuilding and maintaining a clean developer environment.

It replaces one-off shell setup scripts with a structured, repeatable, and safer automation model for:

* workstation setup
* package installation/removal
* dotfiles bootstrap
* desktop/display-specific configuration
* Android development setup
* Docker Engine setup
* KVM/libvirt host setup
* workstation backup and restore
* local Linux VM backup and restore

The project is designed to grow across distros and machine profiles without becoming a pile of disconnected scripts.

---

## Current scope

| Area               |          Status | Notes                                                    |
| ------------------ | --------------: | -------------------------------------------------------- |
| Fedora workstation |          Active | Main target right now                                    |
| Debian workstation |         Partial | Catalog and role paths exist, but Fedora is the priority |
| Dotfiles           |          Active | External repo, applied by convention                     |
| Docker Engine      |          Active | CLI/Engine setup, no Docker Desktop                      |
| Android Dev        |          Active | SDK, command-line tools, AVDs, Android Studio archives   |
| KVM/libvirt host   |          Active | QEMU, libvirt services, permissions, tuned profile       |
| Backup/restore     |          Active | Workstation data backup/restore with confirmation flags  |
| VM backup/restore  |          Active | Local Linux libvirt VM disk/XML backup and restore       |
| NVIDIA             |         Partial | Separate two-run playbook workflow, not runtime-tested   |
| VM guest config    | Planned/skipped | Separate playbook, not part of the current test path     |

---

## Repository model

```text
dev-genesis/
├── ansible.cfg
├── requirements.yml
├── inventories/
│   └── workstation/
├── playbooks/
├── roles/
├── vars/
├── templates/
├── files/
└── vault/
```

| Directory      | Purpose                                             |
| -------------- | --------------------------------------------------- |
| `playbooks/`   | User-facing entry points                            |
| `roles/`       | Reusable automation units                           |
| `vars/`        | Catalogs, package lists, VM data, role inputs       |
| `inventories/` | Machine identity and target groups                  |
| `templates/`   | Generated configuration files                       |
| `files/`       | Static support files                                |
| `vault/`       | Private/secret values, excluded from normal commits |

---

## Target identity

The workstation inventory declares the machine identity explicitly:

```yaml
machine_profile: workstation

target_distro: fedora
desktop_environment: kde
display_server: wayland
dotfiles_profile: modern
```

The `checkhost` role validates machine identity before running opinionated or destructive setup.

This avoids accidentally applying workstation automation to the wrong machine.

---

## Main playbooks

| Playbook                        | Purpose                                           |
| ------------------------------- | ------------------------------------------------- |
| `playbooks/workstation.yml`     | Full developer workstation setup                  |
| `playbooks/setup_dotfiles.yml`  | Dotfiles only                                     |
| `playbooks/backup.yml`          | Backup workstation data                           |
| `playbooks/restore.yml`         | Restore workstation data                          |
| `playbooks/setup_vms.yml`       | Prepare virtualization host and restore local VMs |
| `playbooks/backup_vms.yml`      | Back up local Linux libvirt VMs                   |
| `playbooks/restore_vms.yml`     | Restore local Linux libvirt VMs                   |
| `playbooks/setup_nvidia.yml`    | NVIDIA setup, intentionally isolated              |
| `playbooks/setup_vm_guests.yml` | Guest VM configuration, intentionally isolated    |

---

## Workstation flow

`playbooks/workstation.yml` runs focused roles directly:

| Role                  | Responsibility                                 |
| --------------------- | ---------------------------------------------- |
| `software`            | Install/remove generic software from catalogs  |
| `folders`             | Create standard home directories               |
| `dotfiles`            | Clone and apply the external dotfiles repo     |
| `desktop`             | Desktop-specific setup: KDE, GNOME, Cinnamon   |
| `display`             | Display-server-specific setup: Xorg or Wayland |
| `optimize`            | System-level workstation tuning                |
| `mise`                | Runtime/tool version manager setup             |
| `platformio`          | PlatformIO Core, shell commands, Linux udev rules |
| `android_dev`         | Android SDK, AVDs, Android Studio versions     |
| `docker`              | Docker Engine and user permissions             |
| `virtualization_host` | KVM/libvirt host setup                         |
| `tmux`                | tmux and plugin manager setup                  |

---

## Software catalog

Generic installs are catalog-driven.

```text
vars/software/fedora/install.yml
vars/software/fedora/remove.yml
vars/software/debian/install.yml
vars/software/debian/remove.yml
```

Example:

```yaml
software_install:
  - method: redhat/dnf
    package: bat

  - name: brave
    method: redhat/rpm_repository
    package: brave-browser
    repo:
      mode: repofile_url
      filename: brave-browser.repo
      url: https://brave-browser-rpm-release.s3.brave.com/brave-browser.repo

  - name: postman
    method: common/archive_to_opt
    url: https://dl.pstmn.io/download/latest/linux64
    archive_name: postman.tar.gz
    destination: "{{ ansible_user_dir }}/opt/postman"
    strip_components: 1
```

Supported install methods:

| Family             | Methods                                                                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Red Hat/Fedora     | `redhat/dnf`, `redhat/copr`, `redhat/rpm_url`, `redhat/rpm_repository`, `redhat/github_release_rpm`                                                                 |
| Debian/Ubuntu/Mint | `debian/apt`, `debian/ppa`, `debian/deb_url`, `debian/apt_repository`, `debian/github_release_deb`                                                                  |
| Common             | `common/package`, `common/appimage`, `common/archive_to_opt`, `common/archive_to_fonts`, `common/github_release_archive`, `common/git_repo`, `common/remote_script` |

Use the `software` role for simple install/remove operations. Use dedicated roles when setup has meaningful behavior beyond installing a package.

---

## Dedicated roles

Some tools get their own role because they need validation, configuration, generated files, permissions, services, or multi-step setup.

| Role                         | Why it is dedicated                                           |
| ---------------------------- | ------------------------------------------------------------- |
| `mise`                       | Bash activation, completions, global runtime support          |
| `platformio`                 | User-scope installer script, shell symlinks, Linux udev rules |
| `tmux`                       | TPM and theme repository setup                                |
| `android_dev`                | SDK root, command-line tools, licenses, AVDs, Studio archives |
| `docker`                     | Official repository, packages, service, group membership      |
| `virtualization_host`        | QEMU/libvirt packages, services, groups, ACLs, tuned profile  |
| `backup` / `restore`         | Explicit data movement with safety confirmation               |
| `vms_backup` / `vms_restore` | VM disk/XML backup and restore workflow                       |
| `nvidia`                     | Isolated two-run NVIDIA install and post-reboot verification  |

---

## Dotfiles

Dotfiles are intentionally kept in a separate repository.

`dev-genesis` only owns the automation flow:

```text
clone dotfiles repo
run dotfiles installer
let dotfiles own shell/desktop/app configuration
```

The current role trusts the dotfiles repo convention and expects an installer at the repository root.

---

## Android development

The `android_dev` role manages:

| Component                     | Path                                     |
| ----------------------------- | ---------------------------------------- |
| Android SDK                   | `~/opt/android-sdk`                      |
| Command-line tools            | `~/opt/android-sdk/cmdline-tools/latest` |
| AVD home                      | `~/.config/android/avd`                  |
| Android Studio install root   | `~/opt/android-studio`                   |
| Android Studio archive source | `~/Downloads/AndroidStudioVersions`      |

Android Studio versions are restored from local archives matching:

```text
android-studio-*.tar.gz
```

This keeps Android Studio installation reproducible without depending on a changing remote download page.

---

## Virtualization

The `virtualization_host` role prepares a local KVM/libvirt workstation host.

It handles:

* QEMU/KVM packages
* libvirt services
* VirtIO support for Windows guests
* user group permissions
* `/var/lib/libvirt/images` ACLs
* tuned virtualization profile
* optional IOMMU boot arguments
* `virt-host-validate qemu`

VM backup and restore are separated into dedicated roles:

| Role          | Responsibility                                              |
| ------------- | ----------------------------------------------------------- |
| `vms_backup`  | Export VM domain XML and copy QCOW2 disks                   |
| `vms_restore` | Restore QCOW2 disks, restore SELinux labels, define domains |

Current VM workflow is focused on local Linux libvirt VMs.

---

## Requirements

Install required Ansible collections:

```bash
ansible-galaxy install -r requirements.yml
```

Collections used:

```yaml
collections:
  - name: community.general
  - name: community.libvirt
  - name: community.docker
  - name: ansible.posix
```

---

## Example workstation inventory

```yaml
all:
  children:
    workstation:
      hosts:
        workstation_local:
          ansible_connection: local
          ansible_host: localhost
```

Example host vars:

```yaml
machine_profile: workstation

target_distro: fedora
desktop_environment: kde
display_server: wayland
dotfiles_profile: modern

confirm_workstation_setup: true
confirm_dotfiles_setup: true
confirm_backup: false
confirm_restore: false
confirm_nvidia_setup: false

dotfiles_repo_url: "https://github.com/Brunobrlk/dotfiles.git"

backup_restore_root: "/run/media/{{ ansible_user_id }}/Bruno/Backups/current"
```

Use a Linux-native filesystem for `backup_restore_root`. The backup checks allow
`ext4`, `btrfs`, and `xfs`, and fail early on `exfat`, `vfat`, `ntfs`, and
`ntfs3`. `ext4` is the recommended choice.

Quick backup/restore test:

```bash
ansible-playbook -i inventories/workstation/hosts.yml playbooks/backup.yml \
  -e confirm_backup=true \
  -e 'backup_restore_catalog_file={{ playbook_dir }}/../vars/backup_restore/workstation_test.yml'

ansible-playbook -i inventories/workstation/hosts.yml playbooks/restore.yml \
  -e confirm_restore=true \
  -e 'backup_restore_catalog_file={{ playbook_dir }}/../vars/backup_restore/workstation_test.yml'
```

The quick test catalog copies only a small sample set into
`{{ backup_restore_root }}/_quick_test/` so backup and restore can be verified
without copying the full workstation dataset.

NVIDIA workflow:

```bash
ansible-playbook -i inventories/workstation/hosts.yml \
  playbooks/setup_nvidia.yml \
  -e confirm_nvidia_setup=true

sudo reboot

ansible-playbook -i inventories/workstation/hosts.yml \
  playbooks/setup_nvidia.yml \
  -e confirm_nvidia_setup=true
```

The second invocation does not resume from a saved installer phase. It reruns
the same isolated playbook idempotently and performs post-reboot verification
when no additional reboot-requiring changes were made.

---

## License

MIT
