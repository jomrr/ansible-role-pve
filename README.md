# Ansible Role: pve

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-pve) ![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-pve) ![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-pve) [![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-pve/dev-push-smoke.yml?branch=dev&event=push&label=dev)](https://github.com/jomrr/ansible-role-pve/actions/workflows/dev-push-smoke.yml?query=branch%3Adev) [![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-pve/main-full-gate.yml?branch=main&event=push&label=main)](https://github.com/jomrr/ansible-role-pve/actions/workflows/main-full-gate.yml?query=branch%3Amain)

Ansible role for managing Proxmox VE nodes and clusters.

## Purpose

This role manages Proxmox VE nodes with a deliberately conservative operational profile. It installs a persistent no-subscription warning suppression hook, can mask Proxmox VE HA services on nodes that intentionally do not run HA, and applies host hardening that is safe for PVE, KVM, LXC, and clustered installations.

## Scope

### Managed

- Proxmox VE no-subscription warning suppression script and APT hook.
- Optional masking of `pve-ha-lrm.service`, `pve-ha-crm.service`, and `corosync.service`.
- PVE-safe package, kernel module, sysctl, sshd, auditd, temporary-directory, and mountpoint hardening.

### Not Managed

- Proxmox VE repository setup, installation, upgrades, cluster creation, or storage configuration.
- VM, container, backup, firewall, SDN, Ceph, or HA resource definitions.
- Aggressive CIS controls that can break PVE networking, storage, KVM, LXC, or cluster operation.

## Requirements

- Debian 13 or Proxmox VE based on Debian.
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
| `pve_fail_when_not_proxmox` | `bool` | `false` | `True` | Fail the role when the target does not look like a Proxmox VE node. |
| `pve_no_subscription_nag_enabled` | `bool` | `false` | `True` | Install the Proxmox VE no-subscription warning suppression script and APT hook. |
| `pve_ha_mask_services` | `bool` | `false` | `True` | Stop, disable, and mask selected Proxmox VE HA services when PVE is detected. |
| `pve_ha_services` | `list` | `false` | - pve-ha-lrm.service<br />- pve-ha-crm.service<br />- corosync.service | Proxmox VE HA systemd units managed by the role. |
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
| `pve_hardening_kernel_cmdline_enabled` | `bool` | `false` | `False` | Add supported kernel command line hardening arguments. Disabled by default for module compatibility. |
| `pve_hardening_kernel_cmdline_include_base` | `bool` | `false` | `True` | Include the base kernel command line hardening argument set when cmdline hardening is enabled. |
| `pve_hardening_kernel_cmdline_include_metal` | `bool` | `false` | `True` | Include physical-host IOMMU hardening arguments when cmdline hardening is enabled. |
| `pve_hardening_kernel_cmdline_include_cpu_vendor` | `bool` | `false` | `True` | Include Intel or AMD IOMMU arguments based on the detected CPU vendor. |
| `pve_hardening_kernel_cmdline_reboot_required` | `bool` | `false` | `True` | Report that a reboot is required after kernel command line changes. |

## Managed Files

- `/usr/local/sbin/pve-disable-subscription-nag` idempotent no-subscription warning suppression script
- `/etc/apt/apt.conf.d/99-pve-disable-subscription-nag` APT hook that re-applies the suppression script after package operations
- `/etc/sysctl.d/99-pve-hardening.conf` PVE-safe sysctl hardening values
- `/etc/modprobe.d/pve-hardening.conf` blacklist for safe unused protocols and uncommon filesystems
- `/etc/ssh/sshd_config.d/10-pve-hardening.conf` small sshd drop-in when sshd management is enabled
- `/etc/default/grub.d/10-pve-hardening.cfg` optional GRUB kernel command line drop-in when explicitly enabled

## Security Notes

- Defaults are availability-preserving and avoid root lockout, audit halt-on-full, default SSH forwarding disablement, OverlayFS disablement, USB-storage disablement, and broad firewall policy changes.
- HA masking is refused when `/etc/pve/ha/resources.cfg` contains configured resources.
- PVE-only service actions are skipped when Proxmox VE is not detected and the Proxmox guard is explicitly disabled.
- The hardening profile avoids sysctl values known to interfere with PVE bridges, forwarding, cluster traffic, KVM, or LXC.
- Kernel command line hardening is disabled by default because `module.sig_enforce=1` and `lockdown=integrity` can block unsigned DKMS or third-party modules.

## Operational Notes

- `pve_fail_when_not_proxmox` defaults to true so accidental application to plain Debian fails.
- Review configured HA resources before enabling HA masking on real clusters.
- Audit immutable mode and halt-on-full are opt-in and should be tested against recovery procedures before use.
- Kernel module blacklists affect future loads; reboot or manually unload modules if immediate removal is required.
- Kernel command line hardening preserves existing arguments, adds missing hardening arguments, and requires a reboot.

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
