# Troubleshooting Guide

Common failure scenarios and resolutions for the ubuntu-hardening pipeline.

---

## 1. OpenSCAP remediation script fails mid-run

**Symptom:** The "Apply STIG remediation script" task fails after 10–40 minutes with a non-zero exit code or a connection error.

**What happened:** The generated fix script (`/tmp/os_stig_fix.sh`) ran partially. Some controls may have been applied; others were not. The sentinel file at `/etc/os_stig_remediated` was **not** written (sentinel is only written on `rc == 0`).

**Resolution:**
1. SSH into the host and review `/var/log/oscap/` for any partial scan output.
2. Check what the script was doing when it failed:
   ```bash
   journalctl -n 100 --no-pager
   ```
3. Re-run the full pipeline — the sentinel check will allow a fresh remediation attempt:
   ```bash
   ansible-playbook os_hardening_main.yml
   ```
4. If the script itself is corrupt, delete it so it regenerates on the next run:
   ```bash
   rm -f /tmp/os_stig_fix.sh
   ```

---

## 2. Ansible loses connection during the remediation async task

**Symptom:** "Apply STIG remediation script" task shows "FAILED - RETRYING" or times out waiting for the async job result.

**What happened:** The SSH connection dropped (network issue, sshd restart triggered by the fix script, etc.) while Ansible was polling the async job.

**Resolution:**
1. SSH back to the host and check if the remediation script is still running:
   ```bash
   ps aux | grep os_stig_fix
   ```
2. If still running, wait for it to complete naturally (up to 45 minutes total).
3. If it has exited, check whether the sentinel was written:
   ```bash
   cat /etc/os_stig_remediated
   ```
4. If the sentinel exists, proceed to post-reboot validation with DISA SCC.
5. If the sentinel is absent, re-run the playbook to apply a fresh remediation.

---

## 3. Host unreachable after reboot (DoD banner)

**Symptom:** After the fire-and-forget reboot, all subsequent Ansible plays fail with "UNREACHABLE" or hang indefinitely.

**What happened:** The STIG-required DoD/USG consent banner (`/etc/issue.net`) requires interactive acknowledgement. Ansible's `ansible.builtin.reboot` module (which was deliberately NOT used) would hang waiting for the prompt.

**Resolution:**
This is expected and intentional. Post-reboot compliance validation is performed manually:
1. Wait 2–5 minutes for the host to finish booting.
2. Validate compliance with DISA SCC (see `STIG Documents/scc_scanner.md`).
3. If you need to re-run any Ansible tasks after the reboot, use the `openscap_remediation.yml` play which includes the sentinel check and `ignore_unreachable: true` guard.

---

## 4. AIDE initialization times out

**Symptom:** "Initialize AIDE database" task fails or exceeds the 600-second async timeout.

**What happened:** AIDE initialization scans the entire filesystem; on large or slow VMs it can take longer than 10 minutes.

**Resolution:**
1. Increase the async timeout in `playbooks/tasks/aide_init.yml`:
   ```yaml
   async: 1200   # increase to 20 minutes
   poll: 30
   ```
2. Alternatively, run AIDE init manually after the playbook completes:
   ```bash
   aideinit -y -f
   ```

---

## 5. SSG version mismatch — `ssg-ubuntu2404-ds.xml` not found

**Symptom:** "Fail if Ubuntu 24.04 datastream is missing" task fails.

**What happened:** The `os_stig_ssg_version` variable does not match the expected filename in the zip, or the zip was corrupted during download.

**Resolution:**
1. Check the current SSG version variable:
   ```bash
   grep os_stig_ssg_version roles/os_hardening/defaults/main.yml
   ```
2. Verify the release on the ComplianceAsCode GitHub releases page.
3. Delete the cached zip to force a fresh download:
   ```bash
   # On the target host:
   rm -rf /opt/scap/scap-security-guide-*.zip
   ```
4. Re-run the install play:
   ```bash
   ansible-playbook os_hardening_main.yml --tags packages
   ```

---

## 6. SSH locked out after PAM changes

**Symptom:** SSH authentication fails for all users after the PAM hardening tasks run.

**What happened:** The `common-auth` rewrite or `faillock` configuration may have broken authentication. Common causes:
- `pam_sss.so` returns a hard failure when SSSD is not fully configured.
- `faillock` triggered on the Ansible build user during the run.

**Resolution:**
1. Boot into recovery/single-user mode (see `RECOVERY.md`).
2. From root shell, reset faillock for all users:
   ```bash
   faillock --user <username> --reset
   ```
3. Restore the previous `common-auth` (a `.bak` backup is created by `blockinfile`):
   ```bash
   cp /etc/pam.d/common-auth.bak /etc/pam.d/common-auth
   ```
4. Re-run with the Ansible user exempted:
   ```bash
   ansible-playbook os_hardening_main.yml --tags pam
   ```

---

## 7. GRUB password breaks boot

**Symptom:** System prompts for GRUB username/password on boot and will not proceed automatically.

**What happened:** The `--unrestricted` flag on normal boot entries was not applied correctly (usually because `/etc/grub.d/10_linux` had an unusual CLASS= line format that the regex did not match).

**Resolution:**
1. At the GRUB prompt, enter `root` as username and the `vault_root_recovery_password` value as the password.
2. Boot into single-user mode (see `RECOVERY.md`).
3. From root shell, verify `/etc/grub.d/10_linux` contains `--unrestricted`:
   ```bash
   grep unrestricted /etc/grub.d/10_linux
   ```
4. If not present, add it manually and re-run `update-grub`.

---

## 8. auditd fails to start after audit rules load

**Symptom:** "Restart auditd to apply rules" task fails, or subsequent SSH sessions hang.

**What happened:** The `-e 2` (immutable) flag in `99-finalize.rules` prevents rule changes after auditd loads. If auditd is restarted with conflicting rules, it may fail to start. The system may also refuse rule modifications until reboot.

**Resolution:**
1. The `-e 2` flag takes effect at runtime. Rules can still be loaded by a full auditd restart (which re-reads all rule files from scratch):
   ```bash
   service auditd restart
   ```
2. If auditd still fails, check for rule conflicts:
   ```bash
   augenrules --check
   ```
3. Duplicate rules from a legacy `stig-rules.rules` file (pre-rename) will cause errors — delete it:
   ```bash
   rm -f /etc/audit/rules.d/stig-rules.rules
   augenrules --load
   ```

---

## 9. Sentinel file present but compliance is low

**Symptom:** The playbook skips remediation (sentinel exists), but SCC shows many failures.

**What happened:** Remediation ran previously but may have been incomplete, or a system change reverted hardened settings.

**Resolution:**
1. Remove the sentinel to allow a fresh remediation pass:
   ```bash
   rm -f /etc/os_stig_remediated
   ```
2. Re-run the full pipeline:
   ```bash
   ansible-playbook os_hardening_main.yml
   ```

   > **Warning:** This will trigger another 30–45 minute OpenSCAP remediation run and a system reboot.
