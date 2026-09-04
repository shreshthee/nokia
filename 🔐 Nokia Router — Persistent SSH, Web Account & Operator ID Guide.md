# 🔐 Nokia Router — Persistent SSH, Web Account & Operator ID Guide

> ⚠️ **Warning**
>
> This guide modifies persistent configuration, authentication settings, UBI partitions, and device operator information.
>
> Use only on hardware you own or are authorized to modify.
>
> **UART access is strongly recommended before making persistent changes.**
>
> Keep a backup of important configuration/partitions before proceeding.

---

## 📌 Overview

This guide covers the following workflow:

- Entering OpenWrt failsafe mode through UART
- Reading and changing the Operator ID
- Generating an ED25519 SSH key on Windows
- Mounting required UBI partitions
- Installing an SSH public key
- Configuring Dropbear for key-based authentication
- Enabling the hidden Web Account
- Changing the root password
- Rebooting and verifying the configuration

---

# 🖥️ Chapter 0 — UART → Failsafe → Operator ID

## Step 0.1 — Enter Failsafe Mode

Boot the router through UART and wait until the following message appears:

```text
Press the [f] key and hit [enter] to enter failsafe mode
```

Press:

```text
f
Enter
```

You should reach:

```text
root@(none):/#
```

---

## Step 0.2 — Mount the Root Filesystem

```sh
mount_root
```

---

## Step 0.3 — Create NANDRI Device

```sh
mknod /dev/nandri c 241 0
```

---

## Step 0.4 — Display Current Information

```sh
ritool dump
```

---

## Step 0.5 — Read Current Operator ID

```sh
ritool get OperatorID
```

---

## Step 0.6 — Set Operator ID

Example:

```sh
ritool set OperatorID ALCL
```

---

## Step 0.7 — Verify Operator ID

```sh
ritool get OperatorID
ritool dump
```

---
## Step 0.7 — factory reset

```sh
fw_setenv factory_reset_flag 2
```

---

## Step 0.8 — Save and Reboot

```sh
sync
reboot
```

---

# 🔑 Chapter 1 — Windows SSH Key Generation

After the router reboots, open **Windows PowerShell**.

## Step 1.1 — Check OpenSSH

```powershell
ssh -V
```

Example output:

```text
OpenSSH_for_Windows_9.x
LibreSSL ...
```

---

## Step 1.2 — Check Existing SSH Keys

```powershell
dir $env:USERPROFILE\.ssh
```

Possible files:

```text
id_ed25519
id_ed25519.pub
known_hosts
```

---

## Step 1.3 — Generate ED25519 Key Pair

```powershell
ssh-keygen -t ed25519
```

For the default filename/path, press:

```text
Enter
```

For no passphrase, press `Enter` at both passphrase prompts.

The resulting files are:

```text
id_ed25519
id_ed25519.pub
```

---

## Step 1.4 — Verify Generated Keys

```powershell
dir $env:USERPROFILE\.ssh
```

---

## Step 1.5 — Display Public Key

```powershell
type "$env:USERPROFILE\.ssh\id_ed25519.pub"
```

Copy the **entire single line** beginning with:

```text
ssh-ed25519
```

### Important

```text
id_ed25519      = PRIVATE KEY → Never share
id_ed25519.pub  = PUBLIC KEY  → Install on router
```

---

# 💾 Chapter 2 — Mount Required UBI Partitions

Run the following commands on the router:

```sh
mount_root
cd /tmp
mount_root

ubiattach /dev/ubi_ctrl -m 20 -d 20
mount -t ubifs ubi20_0 /logs

ubiattach /dev/ubi_ctrl -m 22 -d 22
mount -t ubifs ubi22_0 /opt

mount -o remount,rw /

mkdir -p /overlay/etc/config/

ubiattach /dev/ubi_ctrl -m 15 -d 15
mount -t ubifs ubi15_0 /configs

ubiattach /dev/ubi_ctrl -m 16 -d 16
mount -t ubifs ubi16_0 /backup
```

---

# 🔐 Chapter 3 — Install SSH Public Key

## Step 3.1 — Create Dropbear Directory

```sh
mkdir -p /configs/overlay/etc/dropbear
```

---

## Step 3.2 — Install Public Key

Replace:

```text
PASTE_PUBLIC_KEY_HERE
```

with your actual ED25519 public key.

```sh
echo "PASTE_PUBLIC_KEY_HERE" > /configs/overlay/etc/dropbear/authorized_keys
```

Example format:

```text
ssh-ed25519 AAAA... your-comment
```

---

## Step 3.3 — Set Permissions

```sh
chmod 600 /configs/overlay/etc/dropbear/authorized_keys
```

---

## Step 3.4 — Verify

```sh
cat /configs/overlay/etc/dropbear/authorized_keys
```

---

# ⚙️ Chapter 4 — Configure Dropbear

## Step 4.1 — Open Configuration

```sh
cd /configs/overlay/etc/config
vi dropbear
```

Inside `vi`:

```text
Esc
:%d
i
```

Paste:

```text
config dropbear
        option PasswordAuth 'off'
        option RootPasswordAuth 'off'
        option Port '22'
        option SSHKeepAlive '0'
        option IdleTimeout '300'
        option AuthorizedKeysFile '/configs/overlay/etc/dropbear/authorized_keys'
```

Save:

```text
Esc
:wq
Enter
```

---

## Step 4.2 — Verify Configuration

```sh
cat /configs/overlay/etc/config/dropbear
```

---

# 🌐 Chapter 5 — Enable Hidden Web Account

## Step 5.1 — Enable Account

```sh
cfgcli set InternetGatewayDevice.X_Authentication.WebAccount.Enable v true
```

## Step 5.2 — Set Username

```sh
cfgcli set InternetGatewayDevice.X_Authentication.WebAccount.UserName v SatarkUser
```

## Step 5.3 — Set Password

```sh
cfgcli set InternetGatewayDevice.X_Authentication.WebAccount.Password v 'Lokia @123'
```

> The password contains a space, so keep the single quotes.

---

# 🔒 Chapter 6 — Change Root Password

Run:

```sh
passwd root
```

Enter the new root password twice.

---

# 💾 Chapter 7 — Save Changes

```sh
sync
```

Then reboot:

```sh
reboot
```

---

# 💻 Chapter 8 — SSH Login from Windows

After the router has rebooted, open PowerShell.

Replace the IP address with the router's actual LAN address.

Example:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519" root@192.168.1.1
```

Another possible address:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519" root@192.168.18.1
```

---

# ✅ Chapter 9 — Verification

After logging in:

## Verify Root Access

```sh
whoami
```

Expected:

```text
root
```

---

## Verify SSH Public Key

```sh
cat /configs/overlay/etc/dropbear/authorized_keys
```

---

## Verify Dropbear Configuration

```sh
cat /configs/overlay/etc/config/dropbear
```

---

## Verify Operator ID

```sh
mknod /dev/nandri c 241 0 2>/dev/null
ritool get OperatorID
```

Expected:

```text
ALCL
```

---

## Verify Web Account

```sh
cfgcli get InternetGatewayDevice.X_Authentication.WebAccount.Enable
cfgcli get InternetGatewayDevice.X_Authentication.WebAccount.UserName
```

Expected username:

```text
SatarkUser
```

---

# 🔄 Complete Command Flow

The overall workflow is:

```text
UART
  ↓
Failsafe
  ↓
mount_root
  ↓
Create /dev/nandri
  ↓
Read Operator ID
  ↓
Set Operator ID → ALCL
  ↓
Verify
  ↓
sync
  ↓
reboot
  ↓
Generate Windows ED25519 SSH key
  ↓
Mount UBI partitions
  ↓
Install authorized_keys
  ↓
Configure Dropbear
  ↓
Enable Web Account
  ↓
Set Web Username
  ↓
Set Web Password
  ↓
passwd root
  ↓
sync
  ↓
reboot
  ↓
SSH key login
  ↓
Verify configuration
```

---

# ⚠️ Important Notes

### SSH Key

Never upload your private key to GitHub.

Do **not** publish:

```text
id_ed25519
```

Publishing the public key is normally fine:

```text
id_ed25519.pub
```

### Credentials

Do not commit real Web Account passwords or private credentials to a public repository.

For a public GitHub repository, replace example credentials with placeholders such as:

```text
USERNAME_HERE
PASSWORD_HERE
```

### Operator ID

Changing the Operator ID can affect firmware behavior, provisioning, connectivity, or other operator-specific functionality.

Only use an Operator ID appropriate for your device and intended firmware configuration.

### Recovery

Keep UART access available during testing. If configuration changes prevent normal access, use the UART/failsafe environment to inspect and restore the relevant configuration.

---

# 📚 Related Documentation

Useful companion documentation can include:

- UART Access Guide
- OpenWrt Failsafe Guide
- U-Boot Recovery Guide
- Factory Firmware Restore Guide
- OpenWrt Installation Guide
