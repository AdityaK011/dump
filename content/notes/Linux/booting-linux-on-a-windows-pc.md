---
title: "Booting Linux on a Windows PC"
---

This note documents migrating a Windows PC to run Ubuntu: cloning data with `rsync`, removing the Windows NTFS partition, expanding the Ubuntu root into freed space, converting a swap partition to a swapfile, and verifying UEFI boot entries. It covers UEFI NVRAM boot variables, Secure Boot with shim vs GRUB, and recovery via a live USB and chroot.

## Problem Statement and Goals

The goal was to remove the old Windows partition, reuse the freed space for Ubuntu, and convert the swap partition to a swapfile. Key requirements:

- **Data integrity:** Copy all files from the old disk to the new one safely (including permissions) and verify the copy.
- **GPT/UEFI compliance:** Maintain a GPT partition table with a single EFI System Partition (ESP) containing UEFI binaries.
- **Bootloader continuity:** Ensure UEFI firmware loads the Ubuntu bootloader (shim/GRUB) instead of the Windows loader.
- **Swapfile transition:** Remove the swap partition, create a swapfile on the Ubuntu root, and update system config.

Before changes, the disk looked roughly like this:

| Partition | Type         | Size   | Contents              |
|-----------|--------------|--------|-----------------------|
| sda1      | FAT32 (EFI)  | 512MB  | ESP with UEFI binaries|
| sda2      | NTFS         | 200GB  | Windows system & data |
| sda3      | ext4         |  50GB  | Ubuntu root (old)     |
| sda4      | linux-swap   |   8GB  | Swap area (old)       |

After migration:

| Partition | Type         | Size    | Contents        |
|-----------|--------------|---------|-----------------|
| sda1      | FAT32 (EFI)  | 512MB   | ESP (unchanged) |
| sda2      | ext4         | 258GB   | Ubuntu root (resized) |
| (no sda3) | (deleted NTFS)|         |                 |
| (no swap partition) |     |         |                 |

All data is now on `sda2` (former sda3 + free space). Swap is provided by `/swapfile`.

## Migration Steps

1. **Data cloning and verification:** Use `rsync` to copy all files from the old Ubuntu partitions to the new disk. Run in dry-run mode with checksums first:

   ```bash
   sudo rsync -avh --progress --dry-run /mnt/olddisk/ /mnt/newdisk/
   ```

   After verifying output matches expectations, run without `--dry-run`. Check that file counts and checksums match.

2. **Deleting the NTFS partition:** Using GParted on a UEFI-booted live USB, delete the old Windows NTFS partition (sda2). This frees up ~200 GB. GParted aligns partitions on 1 MiB boundaries by default.

3. **Resizing/moving Ubuntu partition:** In GParted, move the Ubuntu ext4 partition (old sda3) to start at the now-earlier boundary, and expand it to fill the freed space. Ensure the start is aligned to 1 MiB for performance.

4. **Swap partition conversion:** Turn off the swap partition and remove it. On the resized Ubuntu root (`/`), create a swapfile:

   ```bash
   sudo swapoff -a
   sudo fallocate -l 8G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

   Add `/swapfile none swap sw 0 0` to `/etc/fstab` to make it permanent.

5. **Updating `/etc/fstab`:** Remove or comment out the old swap partition entry and ensure only the new swapfile entry remains.

6. **Bootloader check:** Because partition numbers changed, verify GRUB still points to the correct root partition. Run `sudo update-grub` on the installed system. If UUIDs haven't changed, GRUB detects kernels normally.

7. **UEFI boot entry:** Run `efibootmgr` to verify firmware boot entries. Before changes, `efibootmgr -v` might show:

   ```
   BootCurrent: 0002
   BootOrder: 0002,0001,2001,2002,2003
   Boot0001* Windows Boot Manager  HD(1,...)/\EFI\Microsoft\Boot\bootmgfw.efi...
   Boot0002* ubuntu        HD(1,...)/\EFI\ubuntu\shimx64.efi
   Boot2001* EFI USB Device  RC
   Boot2002* EFI DVD/CDROM RC
   Boot2003* EFI Network   RC
   ```

   **Boot0002** (`ubuntu`) was first, so the firmware loads Ubuntu (shim) by default. Entries `2001`, `2002`, `2003` are fallback boot options for USB, CD, and network ("RC" = removable media) provided by the firmware. Remove the Windows entry to avoid confusion: `sudo efibootmgr -b 0001 -B`.

   After removal:

   ```
   BootOrder: 0002,2001,2002,2003
   Boot0002* ubuntu (shimx64.efi)
   Boot2001* EFI USB Device
   Boot2002* EFI DVD/CDROM
   Boot2003* EFI Network
   ```

8. **Final reboot and test:** Reboot into Ubuntu. Confirm swap is active (`swapon --show`) and that the ESP mounts at `/boot/efi` (`ls /boot/efi/EFI/boot` should show `bootx64.efi`, `fbx64.efi`, `mmx64.efi`).

## UEFI, EFI Files, and NVRAM Boot Entries

Unlike legacy BIOS, **UEFI** stores boot configuration in NVRAM. The firmware uses *Boot####* variables to know what to load. Each `Boot0001`, `Boot0002`, etc. points to a file on the EFI System Partition (ESP) and an optional name. The `BootOrder` variable tells the firmware which Boot#### entries to try first.

The Ubuntu shim loader lives at `/EFI/ubuntu/shimx64.efi` on the ESP (a FAT32 partition). NVRAM variables might say:

- `Boot0002`: label "ubuntu", path `\EFI\ubuntu\shimx64.efi`.
- `BootOrder: 0002,0001,...` means try Boot0002 first, then Boot0001, etc.

The Linux tool `efibootmgr` reads/writes these NVRAM vars:

> *BootOrder – the boot order as would appear in the boot manager. The boot manager tries to boot the first active entry in this list. If unsuccessful, it tries the next entry, and so on.*

The firmware reads the `BootOrder` NVRAM variable (an array of Boot#### identifiers) and loads the EFI binary pointed to by the first Boot#### in that list. Change BootOrder with `efibootmgr -o ...` or remove entries with `efibootmgr -b NNNN -B`.

## Secure Boot, Shim vs GRUB

With Secure Boot enabled, the firmware **validates signatures** of the EFI binaries it loads. Ubuntu uses a two-stage strategy:

- **Shim (signed by Microsoft):** loaded by firmware first. Firmware checks shim's signature against the DB of trusted keys in firmware.
- **GRUB (signed by Canonical):** shim then loads GRUB (`grubx64.efi`). Shim verifies GRUB's signature using its embedded Canonical certificate.
- **Kernel:** GRUB then loads the Linux kernel (signed by Canonical) and verifies it.

If the firmware tries to load GRUB directly (e.g. boot entry points to `grubx64.efi` on the ESP), the firmware does not have Canonical's key in its trusted DB and will reject GRUB's signature under Secure Boot. **Do not** point to `grubx64.efi` unless Secure Boot is disabled. Under Secure Boot, the entry *must* point to `shimx64.efi`.

Updating NVRAM to `\EFI\ubuntu\grubx64.efi` does not disable Secure Boot — it stays "Enabled" in firmware and will likely refuse to boot GRUB. The correct approach: firmware loads shim (trusted by Microsoft cert), shim loads GRUB (trusted by Canonical's key).

## Recovery (Live USB, chroot, grub-install)

If the system fails to boot after such changes, repair GRUB/UEFI by booting a live USB in UEFI mode:

- **Boot live USB in UEFI mode.** Choose "Try Ubuntu" on UEFI boot.
- **Identify partitions.** Use `lsblk` or `fdisk -l` to find the root (`/`) partition (ext4) and the ESP (FAT32, ~512MB, EFI flag).
- **Mount the installed system:**

  ```bash
  sudo mount /dev/sdaX /mnt            # sdaX = Ubuntu root partition
  sudo mount /dev/sdaY /mnt/boot/efi  # sdaY = EFI System Partition
  ```

- **Bind-mount system dirs:**

  ```bash
  for i in /dev /dev/pts /proc /sys /run; do
      sudo mount --bind $i /mnt$i
  done
  ```

- **Enter chroot:**

  ```bash
  sudo chroot /mnt
  ```

- **Mount efivars (if needed):**

  ```bash
  mount -t efivarfs none /sys/firmware/efi/efivars
  ```

  This ensures the chrooted environment can access UEFI NVRAM.

- **Install GRUB:** For UEFI systems, install to the disk (not partition):

  ```bash
  grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
  update-grub
  ```

  This reinstalls shim+GRUB (if `shim-signed` and `grub-efi-amd64` are present). The device (`/dev/sda`) is the entire disk.

- **Exit and reboot:**

  ```bash
  exit   # leave chroot
  sudo umount /mnt/sys/firmware/efi/efivars
  sudo umount /mnt/dev/pts /mnt/dev /mnt/proc /mnt/sys
  sudo umount /mnt/boot/efi /mnt
  sudo reboot
  ```

  Remove the USB when rebooting.

After reboot, `efibootmgr -v` should again show a valid `ubuntu` entry. If Windows is missing from GRUB afterward, install `os-prober` and ensure `GRUB_DISABLE_OS_PROBER=false` in `/etc/default/grub`, then run `update-grub`.

## Commands and Sample Outputs

**efibootmgr output (before Windows entry removal):**

```bash
$ efibootmgr -v
BootCurrent: 0002
BootOrder: 0002,0001,2001,2002,2003
Boot0001* Windows Boot Manager  HD(1,GPT,...)…\EFI\Microsoft\Boot\bootmgfw.efi
Boot0002* ubuntu        HD(1,GPT,...)…\EFI\ubuntu\shimx64.efi
Boot2001* EFI USB Device        RC
Boot2002* EFI DVD/CDROM       RC
Boot2003* EFI Network         RC
```

**Listing the EFI directory:** The ESP is typically mounted at `/boot/efi`:

```bash
$ ls /boot/efi/EFI/boot
bootx64.efi  fbx64.efi  mmx64.efi
```

**Sample rsync command (dry-run):**

```bash
$ sudo rsync -avh --progress --dry-run /mnt/old/ /mnt/new/
sending incremental file list
.bashrc
Documents/
Documents/report.docx
...
sent 1,234,567 bytes  received 21 bytes  2,345,678.90 bytes/sec
total size is 1,234,567  speedup is 1.00 (DRY-RUN)
```

**Swapfile creation:**

```bash
$ sudo fallocate -l 8G /swapfile
$ sudo chmod 600 /swapfile
$ sudo mkswap /swapfile
Setting up swapspace version 1, size = 8 GiB (8589934592 bytes)
$ sudo swapon /swapfile
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           7.8G        1.1G        4.3G        112M        2.4G        6.4G
Swap:          8.0G          0B        8.0G
```

Add `"/swapfile none swap sw 0 0"` to `/etc/fstab`.

## Safety Checklist and Troubleshooting

- **Back up and verify:** Before any destructive operation, back up important data. Clone the entire drive and verify with `rsync --dry-run` and checksums.
- **Boot media ready:** Keep a working Ubuntu live USB (UEFI boot) at hand in case the system becomes unbootable.
- **Power stability:** Use a UPS or ensure stable power during partitioning.
- **Partition table backup:** Optionally dump the GPT (`sudo sgdisk --backup=table.bin /dev/sda`) so you can restore if something goes wrong.
- **Check fstab edits:** Typing errors in `/etc/fstab` can prevent boot. After editing, test with `sudo mount -a` to catch errors.
- **If boot fails:** Use the recovery steps above. Errors like "no EFI variables" mean you might have skipped mounting `efivars`.

Common troubleshooting:

- If GRUB menu lacks an OS, run `os-prober` and `update-grub`.
- If reinstalling GRUB says "cannot find efivars", ensure you ran `mount -t efivarfs` inside chroot.
- If Secure Boot issues occur, run `apt install shim-signed grub-efi-amd64-signed` and rerun `grub-install`.

## Related notes

- [[notes/Networking/container-networking-internals|Container Networking Internals]] — Linux network namespaces, bridges, and iptables
- [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping & TTYs]] — Unix TTY abstraction and container I/O
