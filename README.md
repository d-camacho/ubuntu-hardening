# Ubuntu 24.04 OS STIG Hardening (DISA)

This repository now focuses exclusively on **Ubuntu 24.04 LTS OS hardening** using DISA STIG-aligned automation.

The hardening pipeline is two-stage:
- `playbooks/pre_openscap.yml`: pre-remediation controls to address known OpenSCAP gaps
- `playbooks/openscap_remediation.yml`: OpenSCAP-driven remediation wrapper and reboot flow

`os_hardening_main.yml` imports both in the required order.

## Project Scope

- Target platform: Ubuntu 24.04 LTS servers
- Automation scope: OS hardening only
- Validation scope: post-reboot compliance validation with DISA SCC
- SCC can also be run before hardening to capture a baseline score; this does not affect playbook execution.

## Repository Structure

- `os_hardening_main.yml`: main entry point for Ubuntu OS STIG hardening
- `bootstrap.yml`: one-time SSH key bootstrap for fresh hosts
- `playbooks/pre_openscap.yml`: pre-remediation hardening tasks
- `playbooks/openscap_remediation.yml`: OpenSCAP remediation wrapper
- `roles/os_hardening/`: install, scan, remediate, post-remediate, reboot sequence
- `STIG Documents/scc_scanner.md`: manual DISA SCC scanning instructions for Body of Evidence (BoE)

## Quick Start

> NOTE
> The `-i` flag is optional when running from project root because `ansible.cfg` sets:
> `inventory = inventories/hosts.yml`.

> NOTE
> `inventories/hosts.yml` is intentionally untracked by git. Copy `example.hosts.yml`
> to that path on your control node and keep site-specific host data local.

1. Copy `example.hosts.yml` to `inventories/hosts.yml` and update hostnames/IPs.
2. Set required values in `inventories/group_vars/all/vault.yml`.
3. Install required Ansible collections:

> NOTE
> `requirements.yml` is only needed for external collections used by this repo
> (`community.general` and `ansible.posix`). If you are running Ansible Core only,
> the playbooks will still work as long as those collections are already installed
> on the control node by some other method.

```bash
ansible-galaxy collection install -r requirements.yml
```

4. Bootstrap SSH keys to fresh hosts (one-time):

```bash
ansible-playbook bootstrap.yml --ask-pass
```

5. Run OS hardening:

```bash
ansible-playbook os_hardening_main.yml
```

6. After reboot, run manual SCC validation per `STIG Documents/scc_scanner.md`.

## Required Vault Variables

Define these in `inventories/group_vars/all/vault.yml` and encrypt with Ansible Vault:

- `ansible_user`
- `ansible_ssh_pass` (if password auth is used)
- `ansible_become_pass`
- `vault_root_recovery_password`
- `vault_audispd_remote_server`
- `vault_ntp_server`

Example encryption command:

```bash
ansible-vault encrypt inventories/group_vars/all/vault.yml
```

## Prerequisites

- Control node with Ansible installed
- `sshpass` installed on the control node if you plan to use `bootstrap.yml`
- Ubuntu 24.04 target host(s)
- Sudo access on target host(s)
- SSH access from the control node to each target host
- Required Ansible collections available on the control node

## Execution Notes

- `bootstrap.yml` is for fresh hosts that still require password-based SSH access.
- `os_hardening_main.yml` is the primary entrypoint for the hardening workflow.
- After OS remediation and reboot, the DoD consent banner can interfere with normal Ansible re-entry.
- Final compliance evidence should be collected manually with DISA SCC after reboot.

## Notes

- OpenSCAP remediation is a one-way operation for hardened targets.
- Post-hardening SSH behavior can prevent fully automated re-entry due to interactive consent banner requirements.
- Final compliance evidence should be collected with DISA SCC after reboot.
