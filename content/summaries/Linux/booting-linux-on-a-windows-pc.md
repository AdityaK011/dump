---
title: "Summary: Booting Linux on a Windows PC"
---

> **Full notes:** [[notes/Linux/booting-linux-on-a-windows-pc|Booting Linux on a Windows PC -->]]

## Key Concepts

### Migration Overview
- Clone Ubuntu data with `rsync` (dry-run + checksums first), delete Windows NTFS partition, expand Ubuntu ext4 into freed space
- Convert swap partition to swapfile on root filesystem; update `/etc/fstab`
- Keep single GPT ESP (FAT32, ~512MB) with UEFI binaries unchanged
- Verify GRUB config (`update-grub`) and UEFI boot entries (`efibootmgr`) after partition number changes

### Before / After Layout
- **Before:** ESP (512MB) + NTFS (200GB) + ext4 root (50GB) + swap partition (8GB)
- **After:** ESP (512MB) + ext4 root (258GB) + `/swapfile` (8GB on root)
- Partition numbers shift when NTFS is deleted — UUIDs usually stay the same if only resizing/moving

### Swap Partition → Swapfile
- `swapoff -a` → `fallocate -l 8G /swapfile` → `chmod 600` → `mkswap` → `swapon`
- Add `/swapfile none swap sw 0 0` to `/etc/fstab`; remove old swap partition entry
- Verify with `swapon --show` and `free -h`

### UEFI NVRAM Boot Entries
- UEFI stores boot config in NVRAM as `Boot####` variables + `BootOrder` list
- Each `Boot####` points to an EFI binary on the ESP (FAT32) with a label
- `efibootmgr -v` shows current boot order; firmware tries entries in `BootOrder` sequentially
- Ubuntu entry typically: `Boot0002* ubuntu` → `\EFI\ubuntu\shimx64.efi`
- Remove stale Windows entry: `sudo efibootmgr -b 0001 -B`
- Entries `2001`/`2002`/`2003` are firmware fallback for USB/CD/network ("RC" = removable media)

### Secure Boot: Shim vs GRUB
- **Secure Boot ON:** firmware validates EFI binary signatures against trusted DB
- **Boot chain:** firmware → shim (Microsoft-signed) → GRUB (Canonical-signed via shim) → kernel (Canonical-signed)
- **Never point boot entry to `grubx64.efi` directly** under Secure Boot — firmware lacks Canonical's key
- Pointing to `grubx64.efi` does NOT disable Secure Boot; boot will fail with signature rejection
- Correct entry must use `shimx64.efi`

### Recovery (Live USB + chroot)
- Boot live USB in **UEFI mode** → mount root + ESP → bind-mount `/dev`, `/proc`, `/sys`, `/run`
- `chroot /mnt` → mount `efivarfs` at `/sys/firmware/efi/efivars` (required for NVRAM access)
- `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu`
- `update-grub` → exit chroot → unmount → reboot
- "cannot find efivars" = forgot to mount efivarfs inside chroot

## Quick Reference

```
Migration sequence:
  rsync (dry-run) -> delete NTFS -> GParted move/expand ext4 -> swapfile -> fstab -> update-grub -> efibootmgr

Swapfile:
  sudo swapoff -a
  sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile
  sudo mkswap /swapfile && sudo swapon /swapfile
  # /etc/fstab: /swapfile none swap sw 0 0

UEFI boot management:
  efibootmgr -v                          # inspect entries
  efibootmgr -b 0001 -B                  # delete Boot0001 (Windows)
  efibootmgr -o 0002,2001,2002,2003      # set boot order

ESP check:
  ls /boot/efi/EFI/ubuntu/               # shimx64.efi, grubx64.efi
  ls /boot/efi/EFI/boot/                 # bootx64.efi (fallback)

Secure Boot boot chain:
  firmware -> shimx64.efi (MS-signed) -> grubx64.efi (Canonical) -> vmlinuz

Recovery chroot:
  mount root -> mount ESP at /mnt/boot/efi -> bind-mount dev/proc/sys/run
  chroot /mnt
  mount -t efivarfs none /sys/firmware/efi/efivars
  grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
  update-grub

Safety:
  rsync --dry-run first          # verify before copy
  sudo sgdisk --backup=table.bin /dev/sda   # GPT backup
  sudo mount -a                  # test fstab before reboot
  Keep UEFI live USB ready

Troubleshooting:
  GRUB missing OS     -> os-prober + update-grub
  No efivars in chroot -> mount -t efivarfs inside chroot
  Secure Boot fail    -> apt install shim-signed grub-efi-amd64-signed; re-run grub-install
```
