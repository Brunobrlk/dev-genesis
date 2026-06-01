# dev-genesis

Ansible project for rebuilding and maintaining my developer environment.

`dev-genesis` automates workstation setup, dotfiles integration, backup restoration, NVIDIA setup, and virtual machine orchestration. Dotfiles and standalone tools stay in separate repositories; this repo owns automation.

## Workflows

```text
playbooks/workstation.yml          # setup real developer machine
playbooks/setup_dotfiles.yml       # apply dotfiles/configuration only
playbooks/backup_restore.yml       # restore backup folders
playbooks/setup_nvidia.yml         # setup NVIDIA drivers/config
playbooks/setup_vms.yml            # create/manage VMs from host
playbooks/setup_vm_guests.yml      # configure inside guest VMs
```

## Target identity

Workstation identity is explicit in `inventories/workstation/host_vars/workstation_local.yml`:

```yaml
target_distro: fedora        # fedora | debian
desktop_environment: kde     # gnome | kde | cinnamon
display_server: wayland      # xorg | wayland
dotfiles_profile: modern
machine_profile: workstation
```

`checkhost` validates this before dangerous or opinionated setup.

## Workstation flow

```text
roles/workstation/tasks/main.yml
  remove_software.yml
  setup_folders.yml
  install_generic_software.yml
  setup_desktop.yml
  setup_display.yml
  setup_specialized_software.yml
  setup_configuration.yml
  optimize.yml
```

## Software model

Generic software is catalog-driven:

```text
vars/software/fedora/install.yml
vars/software/fedora/remove.yml
vars/software/debian/install.yml
vars/software/debian/remove.yml
```

Method names map directly to task paths:

```text
method: redhat/dnf              -> roles/software/tasks/install/redhat/dnf.yml
method: common/archive_to_opt   -> roles/software/tasks/install/common/archive_to_opt.yml
```

Use `roles/software` for simple installs. Use dedicated roles for software/environments with meaningful setup steps: `mise`, `tmux`, `lunarvim`, `android_dev`, `docker`, `virtualization_host`, and `nvidia`.

## Dotfiles

Dotfiles are external. `DOTFILES_SPEC.md` defines the contract. The dotfiles role is expected to clone/pull the repo, validate it, then apply base, desktop, display, distro, and profile-specific configuration.

## Commands

```bash
ansible-galaxy install -r requirements.yml

ansible-playbook -i inventories/workstation/hosts.yml playbooks/workstation.yml --check --diff
ansible-playbook -i inventories/workstation/hosts.yml playbooks/workstation.yml

ansible-playbook -i inventories/workstation/hosts.yml playbooks/setup_dotfiles.yml
ansible-playbook -i inventories/workstation/hosts.yml playbooks/backup_restore.yml
ansible-playbook -i inventories/workstation/hosts.yml playbooks/setup_nvidia.yml

ansible-playbook -i inventories/vms/hosts.yml playbooks/setup_vms.yml
ansible-playbook -i inventories/vms/hosts.yml playbooks/setup_vm_guests.yml
```

## Rules

```text
Generic installs go through the software catalog.
Complex setup gets a dedicated role.
Machine identity must be declared.
Dotfiles stay external and follow DOTFILES_SPEC.md.
```
