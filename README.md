# Ubuntu 24.04 OS STIG Hardening (DISA)

This repository focuses exclusively on **Ubuntu 24.04 LTS OS hardening** using DISA STIG-aligned automation.

The hardening pipeline is two-stage. Stage 1 always runs; choose one Stage 2 engine:

| Stage | Playbook | Description |
|-------|----------|-------------|
| 1 | `playbooks/pre_openscap.yml` | Manual STIG controls and gap fixes (shared by both engines) |
| 2A | `playbooks/openscap_remediation.yml` | OpenSCAP + SSG remediation — no subscription required |
| 2B | `playbooks/ubuntu_pro_remediation.yml` | Ubuntu Pro USG remediation — requires Ubuntu Pro subscription |

`os_hardening_main.yml` is the entry point. Stage 2A is active by default. To switch engines, comment out Stage 2A and uncomment Stage 2B in that file.

## Project Scope

- Target platform: Ubuntu 24.04 LTS servers
- Automation scope: OS hardening only
- Validation scope: post-reboot compliance validation with DISA SCC
- SCC can also be run before hardening to capture a baseline score; this does not affect playbook execution.

## Repository Structure

- `os_hardening_main.yml`: main entry point — choose OpenSCAP or Ubuntu Pro engine here
- `playbooks/bootstrap.yml`: one-time SSH key bootstrap for fresh hosts
- `playbooks/pre_openscap.yml`: Stage 1 — manual STIG pre-hardening tasks (shared)
- `playbooks/openscap_remediation.yml`: Stage 2A — OpenSCAP remediation wrapper
- `playbooks/ubuntu_pro_remediation.yml`: Stage 2B — Ubuntu Pro USG remediation wrapper
- `roles/os_hardening/`: install, scan, remediate, post-remediate, reboot sequence (used by Stage 2A)
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
ansible-playbook playbooks/bootstrap.yml --ask-pass
```

5. Run OS hardening:

```bash
ansible-playbook os_hardening_main.yml
```

6. After reboot, run manual SCC validation per `STIG Documents/scc_scanner.md`.

## Required Vault Variables

Define these in `inventories/group_vars/all/vault.yml` and encrypt with Ansible Vault:

**All runs (Stage 1 + either Stage 2):**
- `ansible_user`
- `ansible_ssh_pass` (if password auth is used)
- `ansible_become_pass`
- `vault_root_recovery_password`
- `vault_audispd_remote_server`
- `vault_ntp_server`

**Stage 2B (Ubuntu Pro) only:**
- `vault_ubuntu_pro_token` — Ubuntu Pro subscription token from [ubuntu.com/pro](https://ubuntu.com/pro)

Example encryption command:

```bash
ansible-vault encrypt inventories/group_vars/all/vault.yml
```

## Prerequisites

- Control node with Ansible installed
- `sshpass` installed on the control node if you plan to use `playbooks/bootstrap.yml`
- Ubuntu 24.04 target host(s)
- Sudo access on target host(s)
- SSH access from the control node to each target host
- Required Ansible collections available on the control node

## Execution Notes

- `playbooks/bootstrap.yml` is for fresh hosts that still require password-based SSH access.
- `os_hardening_main.yml` is the primary entrypoint for the hardening workflow.
- After OS remediation and reboot, the DoD consent banner can interfere with normal Ansible re-entry.
- Final compliance evidence should be collected manually with DISA SCC after reboot.

### Running Playbooks Independently

All pipeline stages can be run in isolation when needed:

```bash
# Stage 1 only — pre-hardening tasks (safe to run multiple times — idempotent)
ansible-playbook playbooks/pre_openscap.yml

# Stage 2A only — OpenSCAP remediation (skipped if sentinel file exists)
ansible-playbook playbooks/openscap_remediation.yml

# Stage 2B only — Ubuntu Pro USG remediation (skipped if sentinel file exists)
ansible-playbook playbooks/ubuntu_pro_remediation.yml
```

Use `pre_openscap.yml` alone to apply or re-apply manual STIG controls without triggering a full remediation run and reboot. Use either Stage 2 playbook alone if pre-hardening has already been applied and you only need to re-run the fix pass.

Individual sections of `pre_openscap.yml` can also be targeted with tags:

```bash
ansible-playbook playbooks/pre_openscap.yml --tags ssh
ansible-playbook playbooks/pre_openscap.yml --tags pam
ansible-playbook playbooks/pre_openscap.yml --tags audit
```

Available tags: `packages`, `openscap`, `ssh`, `pam`, `kernel`, `permissions`, `audit`, `auditd`, `rsyslog`, `aide`, `ntp`, `ufw`, `journald`, `cleanup`.

## SSG Version Management

This project uses the **SCAP Security Guide (SSG)** from [ComplianceAsCode/content releases](https://github.com/ComplianceAsCode/content/releases) to drive OpenSCAP remediation. The Ubuntu apt package (`ssg-debderived`) does **not** include Ubuntu 24.04 content, so the zip must be downloaded from GitHub.

The current pinned version is set in `roles/os_hardening/defaults/main.yml`:

```yaml
os_stig_ssg_version: "0.1.80"
```

**To upgrade to a newer SSG release:**

1. Check the [ComplianceAsCode releases page](https://github.com/ComplianceAsCode/content/releases) for the latest version.
2. Confirm the release includes `ssg-ubuntu2404-ds.xml` (not all releases do).
3. Update `os_stig_ssg_version` in `roles/os_hardening/defaults/main.yml`.
4. Optionally, set the checksum for integrity verification (recommended for production):
   ```yaml
   os_stig_ssg_checksum: "sha256:<hex-from-release-page>"
   ```
5. Delete any cached zip on the target host before re-running:
   ```bash
   rm -rf /opt/scap/scap-security-guide-*.zip
   ```

> **Note:** After upgrading SSG, review the release notes for any new or changed Ubuntu 24.04 STIG controls that may affect your pre-hardening tasks in `playbooks/pre_openscap.yml`.

### Air-Gapped Environments

OpenSCAP installation and SSG staging happens in `pre_openscap.yml` (Section 2). By the time `openscap_remediation.yml` runs, all external dependencies are already satisfied — it has no internet requirements of its own.

To pre-stage the SSG zip from an internal mirror or local file share, set `os_stig_ssg_local_src` in your inventory or via `-e`:

```yaml
# In inventories/group_vars/all/vars.yml or passed with -e
os_stig_ssg_local_src: "/path/to/scap-security-guide-0.1.80.zip"
```

When `os_stig_ssg_local_src` is set, the playbook copies the zip from the control node instead of downloading from GitHub. Leave it empty (default) for internet-connected installs.

## Notes

- OpenSCAP remediation is a one-way operation for hardened targets.
- Post-hardening SSH behavior can prevent fully automated re-entry due to interactive consent banner requirements.
- Final compliance evidence should be collected with DISA SCC after reboot.
