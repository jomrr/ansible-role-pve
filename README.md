# Ansible Role: pve

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-pve) ![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-pve) ![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-pve) [![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-pve/dev.yml?branch=dev&event=push&label=dev)](https://github.com/jomrr/ansible-role-pve/actions/workflows/dev.yml?query=branch%3Adev) [![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-pve/main.yml?branch=main&event=push&label=main)](https://github.com/jomrr/ansible-role-pve/actions/workflows/main.yml?query=branch%3Amain)

Ansible role for managing Proxmox VE nodes and clusters.

## Purpose

This role installs and manages Proxmox VE nodes with a deliberately conservative operational profile. It configures the no-subscription repository and warning suppression, manages Proxmox VE HA service policy, and applies host hardening that is safe for PVE, KVM, LXC, and clustered installations.

## Scope

### Managed

- Proxmox VE package installation on Debian hosts.
- Proxmox VE no-subscription repository state.
- Proxmox VE APT keyring for the target Debian release.
- Proxmox VE no-subscription warning suppression script and APT hook.
- Service policy for `pve-ha-lrm.service`, `pve-ha-crm.service`, and `corosync.service`.
- PVE-safe package, kernel module, sysctl, sshd, auditd, temporary-directory, and mountpoint hardening.

### Not Managed

- Proxmox VE upgrades, cluster creation, or storage configuration.
- VM, container, backup, firewall, SDN, Ceph, or HA resource definitions.
- Aggressive CIS controls that can break PVE networking, storage, KVM, LXC, or cluster operation.

## Requirements

- Debian host with a Proxmox VE supported release codename.
- Root privileges for package, systemd, sysctl, auditd, and configuration-file management.
- `community.general >=12.0.0` and `ansible.posix >=2.1.0` installed on the controller.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: ansible.posix
    version: '>=2.1.0'
```

## Role Variables

The following variables are part of the public role interface.

| Name | Type | Required | Default | Description |
| ---- | ---- | -------- | ------- | ----------- |
| `pve_no_subscription` | `bool` | `false` | `True` | Enable the Proxmox VE no-subscription repository and suppress the no-subscription warning. |
| `pve_ha_services` | `dict` | `false` | pve-ha-lrm.service: false<br />pve-ha-crm.service: false<br />corosync.service: false | Map of Proxmox VE HA systemd units to booleans. True unmasks and enables a unit; false stops, disables, and masks it. |
| `pve_hardening_enabled` | `bool` | `false` | `True` | Enable PVE-safe host hardening tasks. |
| `pve_hardening_disable_usb_storage` | `bool` | `false` | `False` | Blacklist usb-storage. This is opt-in because removable media may be operationally required. |
| `pve_hardening_disable_overlayfs` | `bool` | `false` | `False` | Blacklist overlayfs. This is opt-in because it can affect container workflows. |
| `pve_hardening_disable_squashfs` | `bool` | `false` | `False` | Blacklist squashfs. This is opt-in because some operational workflows use squashfs images. |
| `pve_hardening_auditd_enabled` | `bool` | `false` | `True` | Install and enable auditd on non-container hosts. |
| `pve_hardening_auditd_halt_on_full` | `bool` | `false` | `False` | Halt the host when audit storage is full. Disabled by default for availability. |
| `pve_hardening_audit_immutable` | `bool` | `false` | `False` | Make audit rules immutable until reboot. Disabled by default for operational safety. |
| `pve_hardening_sshd_manage` | `bool` | `false` | `True` | Manage a small PVE-safe sshd hardening drop-in. |
| `pve_hardening_sshd_permit_root_login` | `str` | `false` | `prohibit-password` | Value for sshd PermitRootLogin in the managed drop-in. |
| `pve_hardening_sshd_disable_forwarding` | `bool` | `false` | `False` | Set sshd DisableForwarding yes. Disabled by default to avoid breaking administrative workflows. |
| `pve_hardening_tmp_acl_enabled` | `bool` | `false` | `True` | Ensure shared temporary directories have root ownership and the sticky bit where appropriate. |
| `pve_hardening_mounts_enabled` | `bool` | `false` | `True` | Enable conservative mountpoint hardening for existing safe mountpoints. |
| `pve_hardening_dev_shm_noexec` | `bool` | `false` | `True` | Ensure noexec is part of the /dev/shm mount hardening policy on real hosts. |
| `pve_cmdline` | `dict` | `false` | mitigations=auto: true<br />debugfs=off: true<br />init_on_alloc=1: true<br />init_on_free=1: true<br />kexec_load_disabled=1: true<br />lockdown=integrity: true<br />module.sig_enforce=1: true<br />page_alloc.shuffle=1: true<br />pti=on: true<br />randomize_kstack_offset=on: true<br />slab_nomerge: true<br />spec_store_bypass_disable=on: true<br />vsyscall=none: true<br />efi=disable_early_pci_dma: true<br />iommu=force: true<br />iommu.strict=1: true<br />intel_iommu=on: true<br />amd_iommu=force_isolation: true | Map of kernel command line arguments to booleans. Enabled arguments are added while existing arguments are preserved; Intel and AMD arguments are applied only on matching CPU vendors. Kernel command line changes require a reboot. |

## Managed Files

- `/usr/local/sbin/pve-disable-subscription-nag` idempotent no-subscription warning suppression script
- `/etc/apt/apt.conf.d/99-pve-disable-subscription-nag` APT hook that re-applies the suppression script after package operations
- `/etc/apt/sources.list.d/proxmox.sources` enabled Proxmox VE no-subscription repository in deb822 format
- `/usr/share/keyrings/proxmox-archive-keyring.gpg` Proxmox VE APT keyring for the target Debian release
- `/etc/sysctl.d/99-pve-hardening.conf` PVE-safe sysctl hardening values
- `/etc/modprobe.d/pve-hardening.conf` blacklist for safe unused protocols and uncommon filesystems
- `/etc/ssh/sshd_config.d/10-pve-hardening.conf` small sshd drop-in when sshd management is enabled
- `/etc/default/grub.d/10-pve-hardening.cfg` GRUB kernel command line drop-in used when /etc/kernel/cmdline is absent

## Security Notes

- Defaults are availability-preserving and avoid root lockout, audit halt-on-full, default SSH forwarding disablement, OverlayFS disablement, USB-storage disablement, and broad firewall policy changes.
- The role requires `ansible_facts.distribution` to be Debian; Debian-family derivatives are not accepted as PVE installation targets.
- Proxmox VE package installation is skipped in containers so CI can verify configuration behavior without turning a container into a PVE host.
- The hardening controls avoid sysctl values known to interfere with PVE bridges, forwarding, cluster traffic, KVM, or LXC.
- `pve_cmdline` arguments are enabled by default. Disable arguments such as `module.sig_enforce=1` or `lockdown=integrity` explicitly if they conflict with unsigned DKMS or third-party modules.

## Operational Notes

- `pve_no_subscription` enables `/etc/apt/sources.list.d/proxmox.sources` with `ansible_facts.distribution_release` as suite.
- `pve_no_subscription` removes legacy or conflicting Proxmox VE repository files so APT uses the managed deb822 source.
- Proxmox VE installation follows the Debian package set `proxmox-default-kernel`, `proxmox-ve`, `postfix`, `open-iscsi`, and `chrony`; reboot handling stays outside the role.
- `pve_ha_services` maps each HA unit to true for unmasked/enabled or false for stopped/disabled/masked.
- Audit immutable mode and halt-on-full are opt-in and should be tested against recovery procedures before use.
- Kernel module blacklists affect future loads; reboot or manually unload modules if immediate removal is required.
- `pve_cmdline` controls kernel command line hardening arguments individually; existing arguments are preserved and cmdline changes require a reboot.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| Debian | Debian | 13 | [jomrr/molecule-debian:13](https://hub.docker.com/r/jomrr/molecule-debian) |

## Example Playbook

### Conservative Proxmox VE node baseline

Apply the default PVE-safe baseline to Proxmox VE hosts.

```yaml
---
- name: Manage Proxmox VE nodes
  hosts: pve
  gather_facts: true
  become: true
  roles:
    - role: jomrr.pve
```

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
