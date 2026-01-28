# Le Potato (AML-S905X-CC) — Debian First-Boot Recovery & SSH Bootstrap

This document describes how I recovered SSH access on a fresh Debian install for the Le Potato (AML-S905X-CC) when no user/password was usable on first boot, and how I finished hardening the system **without** needing a monitor long-term.

> TL;DR: Debian on ARM can boot with SSH running but no valid credentials. SSH blocks empty passwords. The fix was offline `/etc/shadow` surgery to set a real root hash, then finishing setup over SSH.

---

## Environment
- **Board:** Le Potato (AML-S905X-CC)
- **OS:** Debian (current stable at install time)
- **Access:** Ethernet + SSH
- **Constraints:** No spare keyboard/monitor during recovery

---

## Symptoms
- System booted normally
- `nmap` showed **port 22 open**
- SSH prompted for a password but rejected all attempts
- Local login rejected root (no known password)
- GRUB/U-Boot menu not accessible on this image

---

## Root Cause
- Debian image booted with **sshd enabled** but **no completed user setup**
- Root account existed but credentials were unusable
- Empty passwords work locally (if allowed) but **OpenSSH blocks empty passwords** by default (`PermitEmptyPasswords no`)

---

## Recovery Strategy (No Monitor Required)
Because local access wasn’t available, I fixed authentication **offline** by editing the SD card.

### 1) Mount the SD Card on Another Machine
Mount the **root filesystem** (usually the second partition).

### 2) Generate a Valid Password Hash
On a Linux host:
```bash
openssl passwd -6
```
This produces a SHA-512 crypt hash (`$6$...`).

### 3) Edit `/etc/shadow`
Replace the root line:
```text
root::20479:0:99999:7:::
```
with:
```text
root:$6$<salt>$<hash>:20479:0:99999:7:::
```
**Rules:**
- One line only
- No spaces
- Don’t alter the trailing fields

### 4) Reinsert SD Card & Boot
SSH now accepts the password corresponding to the hash.

---

## Finish Setup Over SSH
Once logged in as `root`:

### Create a Normal User
```bash
adduser justin
apt update
apt install sudo
usermod -aG sudo justin
```

### Lock Down Root SSH
```bash
sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart ssh
```

(Optional, after keys are set)
```bash
sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh
```

---

## Locale Warnings (Cleanup)
Initial installs produced locale warnings during `apt` operations. Fixed with:
```bash
apt install locales dialog
dpkg-reconfigure locales
```
Selected `en_US.UTF-8` and set as default.

---

## Lessons Learned
- Open port 22 ≠ usable credentials
- SSH **will not** allow empty passwords even if local login does
- ARM images may bypass GRUB entirely
- Editing `/etc/shadow` offline is a valid, time-tested recovery method
- Always create a non-root user early

---

## Status
- ✅ SSH access restored
- ✅ System hardened
- ✅ No reflash required

If this helps you recover a headless Debian ARM install, steal it shamelessly.

