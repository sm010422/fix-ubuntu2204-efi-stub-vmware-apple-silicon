# Fix: Ubuntu 22.04 ARM Stuck at "EFI stub: Exiting boot services..." on VMware Fusion (Apple Silicon)

> **Environment**: MacBook (Apple M2) + VMware Fusion Pro 26H1 + Ubuntu 22.04.5 ARM64 Server  
> **Symptom**: After installation, VM hangs permanently at `EFI stub: Exiting boot services...` on every boot

---

## Table of Contents

1. [Problem Description](#1-problem-description)
2. [Root Cause](#2-root-cause)
3. [Things That Did NOT Work](#3-things-that-did-not-work)
4. [The Fix](#4-the-fix)
5. [Permanent Fix (Install HWE Kernel)](#5-permanent-fix-install-hwe-kernel)
6. [TL;DR](#6-tldr)

---

## 1. Problem Description

After a successful installation of Ubuntu 22.04.5 ARM64 Server on VMware Fusion, every reboot hangs indefinitely at the following output:

```
EFI stub: Booting Linux Kernel...
EFI stub: EFI_RNG_PROTOCOL unavailable
EFI stub: Using DTB from configuration table
EFI stub: Exiting boot services...
```

Nothing appears after this. The screen stays black forever regardless of how long you wait.

---

## 2. Root Cause

The default kernel shipped with Ubuntu 22.04 — **`5.15.x`** — has a known boot failure issue in VMware Fusion on Apple Silicon.

- After the EFI stub exits boot services, the kernel should begin initialization — but it silently freezes
- `EFI_RNG_PROTOCOL unavailable` is just a warning and is **not** the cause
- Recovery mode uses the same kernel (5.15), so it fails identically
- The **HWE (Hardware Enablement) kernel (`5.19+`)** boots successfully

---

## 3. Things That Did NOT Work

All of the following were attempted and failed with the same result.

### 3-1. Adding `acpi=force` in GRUB editor

Pressed `e` in GRUB menu and appended to the `linux` line:
```
acpi=force
```
→ **Failed**: Same hang at EFI stub

### 3-2. Adding `earlycon` + `console` parameters

```
acpi=force earlycon=pl011,0x9000000 console=ttyAMA0
```
→ **Failed**: No kernel output at all, still hangs

### 3-3. Adding software MMU options to `.vmx`

```
monitor.virtual_mmu = "software"
monitor.virtual_exec = "software"
```
→ **Failed**: Same symptom

### 3-4. Deleting `.nvram` file (EFI settings reset)

```bash
rm "Ubuntu 64-bit Arm Server 22.04.5 4.nvram"
```
→ **Failed**: Same symptom

### 3-5. Recovery mode boot

GRUB → Advanced options → recovery mode  
→ **Failed**: Uses the same 5.15 kernel, hangs identically

### 3-6. Manual boot from GRUB shell with `acpi=off`

```
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-5.15.0-179-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro acpi=off
grub> initrd /initrd.img-5.15.0-179-generic
grub> boot
```
→ **Failed**: The issue is the kernel itself, not the boot parameters

---

## 4. The Fix

### Key Insight

The Ubuntu 22.04 installation ISO includes an **HWE kernel** (`hwe-vmlinuz`, `hwe-initrd`) inside its `/casper/` directory. By booting directly from that HWE kernel via the GRUB shell, the system boots successfully.

### Step-by-Step

#### Step 1. Enter GRUB Shell

As soon as the VM starts, click inside the VMware window to capture keyboard input and press **`c`** to enter the GRUB command line.

> **Tip**: If the GRUB menu doesn't appear, spam **ESC** right after the VM starts to get to the Boot Manager, then navigate: Boot Manager → ubuntu → GRUB menu → press `c`

```
GNU GRUB  version 2.06

grub> _
```

#### Step 2. List Available Devices

```
grub> ls
```

Example output:
```
(proc) (memdisk) (hd0) (hd0,gpt3) (hd0,gpt2) (hd0,gpt1) (cd0) (cd0,msdos2) (cd0,msdos1) (lvm/ubuntu--vg-ubuntu--lv)
```

`(cd0)` is the CDROM drive where the Ubuntu ISO is mounted.

#### Step 3. Verify HWE Kernel Exists in ISO

```
grub> ls (cd0)/casper/
```

Confirm that `hwe-vmlinuz` and `hwe-initrd` are present in the output:

```
hwe-initrd  hwe-vmlinuz  initrd  vmlinuz  ...
```

#### Step 4. Boot Using the HWE Kernel from ISO

```
grub> linux (cd0)/casper/hwe-vmlinuz
grub> initrd (cd0)/casper/hwe-initrd
grub> boot
```

The system will boot successfully into the Ubuntu environment.

---

## 5. Permanent Fix (Install HWE Kernel)

Once booted, install the HWE kernel so the VM boots normally without needing the ISO.

### Option A: chroot from ISO environment

```bash
# Activate LVM volume
vgchange -ay

# Mount root partition
mount /dev/mapper/ubuntu--vg-ubuntu--lv /mnt

# Bind mount essential directories
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

# Enter chroot
chroot /mnt

# Install HWE kernel
apt update
apt install linux-generic-hwe-22.04 -y

# Reboot
exit
reboot
```

### Option B: Install directly after booting (if you got in via Option A above)

After logging in:

```bash
sudo apt update
sudo apt install linux-generic-hwe-22.04 -y
sudo reboot
```

After rebooting, the VM should boot normally without the ISO attached.

---

## 6. TL;DR

| Item | Detail |
|------|--------|
| **Broken kernel** | `linux-generic` (5.15.x) |
| **Working kernel** | `linux-generic-hwe-22.04` (5.19+) |
| **Quick fix** | Boot from ISO's HWE kernel via GRUB shell: `linux (cd0)/casper/hwe-vmlinuz` |
| **Permanent fix** | `sudo apt install linux-generic-hwe-22.04` |
| **Affected environments** | VMware Fusion + Apple Silicon (M1/M2/M3) + Ubuntu 22.04 |

---

## Notes

- This issue affects VMware Fusion 13.x through 26H1.
- Ubuntu 22.04.2 LTS and later work correctly on VMware Fusion ARM when using the HWE kernel.
- If you're doing a fresh install, consider **Ubuntu 24.04 LTS** instead — its default kernel (6.x) has no such issue.

---

*This document is based on firsthand troubleshooting experience.*
