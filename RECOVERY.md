# Recovery Guide

This guide documents how to recover access to a hardened Ubuntu 24.04 system
when normal SSH login is not available. Two recovery paths are documented below.

---

## Overview of Access Restrictions Applied by This Hardening

| Control | What It Does | Recovery Impact |
|---------|-------------|-----------------|
| V-270724 | Root account locked (`passwd -l root`) | Cannot `su -` or log in as root normally |
| V-270675 | GRUB bootloader password set | Must authenticate at GRUB to edit boot entries |
| V-270691 | DoD consent banner on SSH | Interactive banner; Ansible cannot reconnect post-reboot |
| UBTU-24-400110 | `PermitRootLogin no` in sshd | Cannot SSH directly to root |
| UBTU-24-200610 | faillock deny=3, unlock_time=0 | Accounts lock permanently after 3 failures; admin must reset |

---

## Path 1: Single-User / Maintenance Mode via GRUB (No Network Required)

Use this when:
- SSH is unavailable (network issue, sshd misconfiguration, PAM lockout)
- Root account needs to be unlocked
- PAM configuration needs to be restored

### Step-by-Step

1. **Access the console.** Physical console or hypervisor console (vSphere, KVM, Proxmox, etc.).

2. **Reboot the system** (or power cycle if unresponsive):
   ```
   # From a shell if you have any access:
   sudo reboot
   ```

3. **At the GRUB menu**, press `e` to edit the default boot entry.
   - GRUB will prompt: `Enter username:` → type `root`
   - GRUB will prompt: `Enter password:` → enter the value of `vault_root_recovery_password` from your vault file.

4. **Edit the boot entry** to enter single-user mode:
   - Find the line beginning with `linux` (or `linuxefi`).
   - At the end of that line, add: `single` (or `init=/bin/bash` for a bare shell)
   - Example:
     ```
     linux   /boot/vmlinuz-... root=UUID=... ro quiet splash single
     ```

5. **Press `Ctrl+X`** (or `F10`) to boot with the modified entry.

6. **You will land in a root shell.** The root account is locked (`passwd -l`) but in single-user mode the kernel drops you to a shell without PAM authentication.
   - If prompted for the maintenance password, use the value of `vault_root_recovery_password`.

7. **Perform recovery actions** (examples below).

8. **Resume normal boot:**
   ```bash
   exec /sbin/init
   # or just:
   reboot
   ```

---

### Common Recovery Actions from Single-User Shell

**Reset a locked user account (faillock):**
```bash
faillock --user clab_admin --reset
```

**Unlock the root account temporarily:**
```bash
passwd -u root
# When done with recovery, re-lock:
passwd -l root
```

**Restore PAM common-auth from backup:**
```bash
ls /etc/pam.d/common-auth*
cp /etc/pam.d/common-auth.bak /etc/pam.d/common-auth
```

**Fix sshd configuration:**
```bash
# Remount filesystem read-write if needed:
mount -o remount,rw /
# Edit the config:
vi /etc/ssh/sshd_config.d/00-stig-hardening.conf
```

**Remove OpenSCAP remediation sentinel (to allow re-run):**
```bash
rm -f /etc/os_stig_remediated
```

**Check and reset faillock for all interactive users:**
```bash
for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    faillock --user "$user" --reset
    echo "Reset faillock for $user"
done
```

---

## Path 2: Emergency SSH Key Recovery (Network Available)

Use this when you can still reach the host over SSH but your credential is locked.

**Prerequisites:** You have a second user account with `sudo` rights, OR the Ansible service account (`clab_admin`) is exempted from faillock (it should be — see `faillock.conf: ignore_users`).

1. **Test the Ansible account directly:**
   ```bash
   ssh clab_admin@<host>
   ```
   If `ignore_users = clab_admin` is set in `/etc/security/faillock.conf`, this account will never lock.

2. **From that shell, reset other accounts:**
   ```bash
   sudo faillock --user <locked_user> --reset
   ```

3. **Re-enable an SSH key if the authorized_keys was cleared:**
   ```bash
   mkdir -p ~/.ssh
   echo "<your-public-key>" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

---

## Path 3: Hypervisor Console / Cloud Provider Recovery

For cloud VMs (AWS, Azure, GCP) or VMware environments:

| Platform | Recovery Method |
|----------|----------------|
| AWS EC2 | Stop instance → detach root EBS → attach to rescue instance → edit files → reattach |
| Azure VM | Use Azure Serial Console or az vm repair commands |
| GCP Compute | Use rescue disk or metadata-based startup scripts |
| VMware vSphere | Use VM console in vCenter; boot from rescue ISO if needed |
| Proxmox | Use `qm terminal` for serial console access |

---

## GRUB Password Reference

The GRUB bootloader password is set from `vault_root_recovery_password` during hardening.

- **Username at GRUB prompt:** `root`
- **Password at GRUB prompt:** the value of `vault_root_recovery_password` from your Ansible vault

> **Important:** This is the GRUB bootloader password, not the Linux root account password (though both are set to the same `vault_root_recovery_password` value by this playbook for simplicity). Store this credential securely — without it, physical/console access to modify boot parameters requires booting from external media.

---

## Re-Hardening After Recovery

After recovery actions that modified PAM, sshd, or other hardened configs:

1. Remove the sentinel if a fresh OpenSCAP pass is needed:
   ```bash
   sudo rm -f /etc/os_stig_remediated
   ```

2. Re-run the pre-hardening playbook (safe to run multiple times):
   ```bash
   ansible-playbook playbooks/pre_openscap.yml
   ```

3. Re-run the full pipeline if OpenSCAP remediation is needed:
   ```bash
   ansible-playbook os_hardening_main.yml
   ```
