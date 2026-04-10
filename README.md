# Standalone Kali Linux — GNOME (No Chroot)

## Prerequisites & Disclaimer

> **WARNING:** This procedure requires the prior removal of the host machine's internal drive and its replacement with a dedicated SSD before proceeding. Failure to do so will result in the overwrite of the existing EFI/BIOS partition, rendering the primary operating system unbootable. Proceed only once physical drive substitution has been confirmed.

When executed correctly, this methodology resolves the following issues endemic to alternative approaches currently documented elsewhere:

- Inadvertent overwrite of critical data, directories, and partitions during Kali Linux installation
- Corruption of the host machine's EFI partition resulting in an unbootable system
- Exposure of an unprotected bootloader to evil-maid and evil-bootloader attack vectors
- Time loss attributable to incomplete or unverified third-party documentation

This guide was developed and validated on a **MacBook Air (13-inch, Early 2015)**. It is expected to function on any EFI-bootable machine, including late-2015 iMac models, provided the internal drive is removed prior to installation. Hardware used during development included a 500 GB PCIe 3.2 installation medium, a Physon 256 GB NVMe 4.0×4 Blade SSD as the target drive in place of the original Apple M.2, and a 2 TB WD_Black NVMe.

This document was compiled by cross-referencing numerous guides, blog posts, and community threads, as well as G0TM1LK's official documentation. The original references consulted were:

- https://www.kali.org/docs/installation/hard-disk-install/
- https://www.kali.org/docs/installation/btrfs/
- https://www.kali.org/docs/usb/usb-standalone-encrypted/
- https://www.gnu.org/software/grub/manual/grub/grub.html#Introduction-1
- https://superuser.com/questions/1705973/add-efi-partition-to-bios-menu-after-nvram-reset
- https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html
- https://bbs.archlinux.org/viewtopic.php?id=268460

> It is worth noting that the official Kali Linux documentation referenced above was subsequently revised to align with the methodology presented here, rather than the reverse.

> **Optional — Extra Flavor & Security:** Adjust this guide accordingly so that your EFI partition resides on a separate external thumb-drive. You can then remove the drive to mitigate retaliation on your bootloader mid-attack/post-install.

---

## Device Naming Convention

Throughout this guide, the target drive is referred to as `nvme0n1`, with partitions designated:

```
nvme0n1p1
nvme0n1p2
nvme0n1p3 / nvme0n1p3_crypt / luks-aaaa-aa-aa-aa-aaaaaa
nvme0n1p4_crypt
nvme0n1p5_crypt
```

**Verify all device identifiers against your own hardware configuration via `lsblk` before executing any commands.** Note that some machines will enumerate NVMe drives as `sdX` rather than `nvme0n1`, or vice versa depending on adapter usage. It is strongly recommended to transcribe and adjust this guide to reflect your specific device identifiers before beginning.

> **Optional Security Enhancement:** For additional protection against bootloader tampering, consider housing the EFI partition on a separate removable USB drive. Physical possession of the boot medium then becomes a prerequisite for system access, providing meaningful mitigation against post-install bootloader attacks.

---

## Physical Preparation

1. Power down the Mac.
2. Carefully remove the internal drive. This step ensures that `/target/boot/efi` is not inadvertently mounted to an incorrect location during installation.
3. Install the target SSD in the internal slot.
4. Reconnect power, attach the Kali installer USB via SATA 3.2 adapter, and boot from the installation image.

---

## Installation

At the boot menu, navigate to:

```
Advanced Options → Graphical Expert Install
```

Allow the installer to load its component list, then proceed through locale, language, and keyboard configuration according to preference.

When prompted to select additional components, choose:

```
cryptodisks udeb
parted udeb
low-mem install
mbr udeb
fdisk udeb
rescue mode
```

Proceed through network configuration. If the hardware uses an unsupported network driver — as is the case with this MacBook model — dismiss any network warnings and continue. Complete the following steps using system defaults unless otherwise noted:

- Network name
- System name
- Username & passwords
- Install base system
- Configure package manager

When prompted again for additional packages, select:

```
low-mem
```

Select **Detect disks**, then proceed to **Manual partitioning**.

The installer's built-in partition editor is sufficient for this procedure; `gdisk` is not required.

### Partition Layout

Partition the target drive as follows:

| Size | Type | Mount | Notes |
|------|------|-------|-------|
| ~570 MB | EFI | — | EFI system partition |
| 2 MB | Reserved | — | BIOS GRUB bootloader space |
| ≥5 GB (6.9 GB rec.) | ext4 | `/boot` | Mount option: `noatime` |
| ≥2× boot size | — | (do not use) | Reserved for boot backup; note size |
| ≥16.9 GB | Physical volume for encryption | — | Swap partition |
| All remaining (minus 16.9 GB) | Physical volume for encryption | — | Root partition |

> **Note:** Plan the root partition such that a minimum of **18 GB** remains available for use as a boot partition backup later in the process.

From the top of the partition menu:

1. Select **Configure encrypted volumes → Create encrypted volume**
2. Select the volumes to encrypt (those labeled with `crypt`), select **Finish**, confirm writing changes to disk, and set encryption passphrases.
3. From the two newly visible encrypted volumes:
   - Select the **root volume** → assign `btrfs` as the filesystem → set `/` as the mount point → apply `noatime` → format and finish.
   - Select the **second volume** → assign as swap space → format and finish.
4. Select **Finish partitioning and write changes to disk**, then continue.

### Software Selection

When prompted, deselect all options **except**:

```
GNOME
Default Software
```

### GRUB Installation

Proceed to **Install GRUB bootloader**.

- When prompted, select **Yes** to force EFI installation to the removable path.
- **Critical:** Do not permit the installer to modify NVRAM settings under any circumstances.
- Decline installation of `os-prober` — this is the only OS present on the drive at this stage and will be addressed later if required.

Complete the remaining installation steps using defaults. Remove the installation media and reboot into the newly installed system.

---

## Configuring BTRFS Filesystems

```bash
sudo apt dist-upgrade -y && sudo apt auto-remove -y && sudo apt clean
```

```bash
sudo passwd
```

Install essential tools:

```bash
sudo apt update && sudo apt install btrfs-progs snapper snapper-gui grub-btrfs -y
```

Create the Snapper configuration for the root filesystem `/`:

```bash
sudo cp /usr/share/snapper/config-templates/default /etc/snapper/configs/root
sudo sed -i 's/^SNAPPER_CONFIGS=\"\"/SNAPPER_CONFIGS=\"root\"/' /etc/default/snapper
```

Prevent `updatedb` from indexing snapshots (which would slow down the system):

```bash
sudo sed -i '/# PRUNENAMES=/ a PRUNENAMES = ".snapshots"' /etc/updatedb.conf
```

```bash
sudo mount /dev/mapper/nvme0n1p4_crypt /mnt

sudo btrfs subvolume create /mnt/@var@lib@gdm3
sudo btrfs subvolume create /mnt/@var@lib@AccountsService

sudo find /var/lib/gdm3/ -mindepth 1 -exec mv -t /mnt/@var@lib@gdm3/ {} +
sudo find /var/lib/AccountsService/ -mindepth 1 -exec mv -t /mnt/@var@lib@AccountsService/ {} +
```

Verify with:

```bash
ls -la /mnt/@var@lib@gdm3
ls -la /mnt/@var@lib@AccountsService
```

---

## Finalising the BTRFS Configuration

```bash
sudo blkid /dev/mapper/nvme0n1p4_crypt
# : UUID="<uuid>"
```

```bash
sudo nano /etc/fstab
```

Add the following (substitute `<uuid>` with the value from the previous command):

```fstab
# /var/lib/gdm3 was on /dev/mapper/nvme0n1p4_crypt during installation
UUID=<uuid> /var/lib/gdm3   btrfs   defaults,subvol=@var@lib@gdm3 0       0
# /var/lib/AccountsService was on /dev/mapper/nvme0n1p4_crypt during installation
UUID=<uuid> /var/lib/AccountsService   btrfs   defaults,subvol=@var@lib@AccountsService 0       0
```

Save, exit, and reload:

```bash
sudo systemctl daemon-reload
sudo reboot now
```

Archive the `/boot` directory elsewhere (on another device), then unmount:

```bash
sudo mount -oremount,ro /boot
sudo mount -oremount,ro /boot/efi
sudo chown <user> /tmp
sudo install -m0600 /dev/null /tmp/boot.tar
sudo tar -C /boot --acls --xattrs --one-file-system -cf /tmp/boot.tar .
sudo umount /boot/efi
sudo umount /boot
```

Wipe the underlying block device (assumed to be `/dev/nvme0n1p2`):

```bash
sudo dd if=/dev/urandom of=/dev/nvme0n1p2 bs=1M status=none
# dd: error writing '/dev/nvme0n1p2': No space left on device
```

Format the underlying block device to **LUKS1** (note `--type luks1` — Buster's `cryptsetup` defaults to LUKS2):

```bash
sudo cryptsetup luksFormat --type=luks1 /dev/nvme0n1p2

# WARNING!
# ========
# This will overwrite data on /dev/nvme0n1p2 irrevocably.
#
# Are you sure? (Type uppercase yes): YES
# Enter passphrase for /dev/nvme0n1p2:
# Verify passphrase:
```

Add a corresponding entry to `crypttab`, open it, and verify:

```bash
echo "nvme0n1p2_crypt UUID=$(sudo blkid -o value -s UUID /dev/nvme0n1p2) none luks" | sudo tee -a /etc/crypttab
sudo cryptdisks_start nvme0n1p2_crypt
# Starting crypto disk...nvme0n1p2_crypt (starting)...
# Please unlock disk nvme0n1p2_crypt:  ********
# nvme0n1p2_crypt (started)...done.
```

Reuse the old UUID from `fstab` when creating the filesystem to avoid editing the file:

```bash
sudo grep /boot /etc/fstab
# /boot was on /dev/nvme0n1p2 during installation
# UUID=<uuid> /boot           ext4    defaults        0       2

sudo mkfs.ext4 -m0 -U <uuid> /dev/mapper/nvme0n1p2_crypt
# abcdef 1.23.4 (15-Dec-2018)
# Creating filesystem with 246784 1k blocks and 61752 inodes
# Filesystem UUID: x-x-x-x-x
# […]
```

Mount `/boot` again from fstab and restore the archive:

```bash
sudo mount -v /boot
# mount: /dev/mapper/nvme0n1p2_crypt mounted on /boot.

sudo tar -C /boot --acls --xattrs -xf /tmp/boot.tar
sudo mount -v /boot/efi
```

---

## Enabling Cryptomount in GRUB2 + Removable Mode, Keyslot Setup & Auto-Unlock

Enable the feature, update the GRUB image, and reinstall in removable mode:

```bash
echo "GRUB_ENABLE_CRYPTODISK=y" | sudo tee -a /etc/default/grub
sudo update-grub

sudo grub-install /dev/nvme0n1 --force-extra-removable --no-nvram --uefi-secure-boot
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --boot-directory=/boot --bootloader-id=GRUB --force-extra-removable --no-nvram --uefi-secure-boot
```

Check current PBKDF iterations and reduce for faster unlock:

```bash
sudo cryptsetup luksDump /dev/nvme0n1p2 | grep -B1 "Iterations:"
# Key Slot 0: ENABLED
# Iterations:             1000000

sudo cryptsetup luksChangeKey --pbkdf-force-iterations 420690 /dev/nvme0n1p2
# Enter passphrase to be changed:
# Enter new passphrase:
# Verify passphrase:
```

> You can reuse the existing passphrase in the above prompts.

```bash
sudo cryptsetup luksOpen --test-passphrase --verbose /dev/nvme0n1p2
# Enter passphrase for /dev/nvme0n1p2:
# Key slot 1 unlocked.
# Command successful.
```

### Avoiding the Extra Password Prompt

```bash
sudo mkdir -m0700 /etc/keys
bash
( umask 0077 && sudo dd if=/dev/urandom bs=1 count=64 of=/etc/keys/boot.key conv=excl,fsync )
( umask 0077 && sudo dd if=/dev/urandom bs=1 count=64 of=/etc/keys/swap.key conv=excl,fsync )
( umask 0077 && sudo dd if=/dev/urandom bs=1 count=64 of=/etc/keys/root.key conv=excl,fsync )
zsh
sudo chmod u=rx,go-rwx /etc/keys
sudo chmod u=r,go-rwx /etc/keys/boot.key
sudo chmod u=r,go-rwx /etc/keys/swap.key
sudo chmod u=r,go-rwx /etc/keys/root.key
```

Create new key slots with the new key files:

```bash
sudo cryptsetup luksAddKey /dev/nvme0n1p2 /etc/keys/boot.key
sudo cryptsetup luksAddKey /dev/nvme0n1p3 /etc/keys/swap.key
sudo cryptsetup luksAddKey /dev/nvme0n1p4 /etc/keys/root.key
```

Add the entries to `crypttab`:

```bash
sudo sed -i "/^nvme0n1p2_crypt/c\nvme0n1p2_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p2) /etc/keys/boot.key luks,key-slot=1" /etc/crypttab
sudo sed -i "/^nvme0n1p3_crypt/c\nvme0n1p3_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p3) /etc/keys/swap.key luks,swap,discard,key-slot=1" /etc/crypttab
sudo sed -i "/^nvme0n1p4_crypt/c\nvme0n1p4_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p4) /etc/keys/root.key luks,discard,key-slot=1" /etc/crypttab
```

> Verify everything is correct and comment out (`#`) the original entries:
> ```bash
> sudo nano /etc/crypttab
> ```
> The mapper name for boot may change — run `lsblk` to confirm before editing.

---

## Finalising the BTRFS fstab Configuration

```bash
sudo nano /etc/fstab
```

Add/update the following entries (substituting your actual UUIDs):

```fstab
/dev/mapper/nvme0n1p4_crypt	/		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@		0	0
/dev/mapper/nvme0n1p4_crypt	/.snapshots	btrfs	defaults,noatime,ssd,compress=lzo,subvol=@.snapshots	0	4
# /boot was on /dev/nvme0n1p2 during installation
UUID=<uuid>	/boot		ext4	defaults,noatime					0	1
# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=<uuid>	/boot/efi	vfat	umask=0077						0	1
/dev/mapper/nvme0n1p4_crypt	/home		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@home		0	2
/dev/mapper/nvme0n1p4_crypt	/root		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@root		0	3
/dev/mapper/nvme0n1p4_crypt	/srv		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@srv		0	0
/dev/mapper/nvme0n1p4_crypt	/tmp		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@tmp		0	0
/dev/mapper/nvme0n1p4_crypt	/usr/local	btrfs	defaults,noatime,ssd,compress=lzo,subvol=@usr@local	0	0
/dev/mapper/nvme0n1p4_crypt	/var/log	btrfs	defaults,noatime,ssd,compress=lzo,subvol=@var@log	0	0
/dev/mapper/nvme0n1p3_crypt	none		swap	sw							0	0
# /var/lib/gdm3*lightdm was on /dev/mapper/nvme0n1p4_crypt during installation
UUID=<uuid>	/var/lib/gdm3*lightdm		btrfs	defaults,subvol=@var@lib@gdm3			0	0
# /var/lib/AccountsService was on /dev/mapper/nvme0n1p4_crypt during installation
UUID=<uuid>	/var/lib/AccountsService	btrfs	defaults,subvol=@var@lib@AccountsService		0	0
```

Save and exit, then:

```bash
sudo systemctl daemon-reload
```

Tell `initramfs` to use the keyfile:

```bash
echo "KEYFILE_PATTERN=/etc/keys/*.key" | sudo tee -a /etc/cryptsetup-initramfs/conf-hook
```

Include restrictive permissions to avoid leaking key material:

```bash
echo "UMASK=0077" | sudo tee -a /etc/initramfs-tools/initramfs.conf
```

Regenerate `initramfs`:

```bash
sudo update-initramfs -u -k all
```

Update GRUB:

```bash
sudo update-grub
```

Reboot for changes to take effect:

```bash
sudo reboot now
```

> **we're in....**

---

## Post-Installation

Once the system has rebooted successfully:

1. Open the **Disks** utility via the application launcher.
2. Select the unlocked boot partition, click the gear icon, and choose **Change Passphrase**. Cycle the passphrase to a temporary value and then back to the intended secure passphrase — this step initialises the keyring entry correctly.
3. Click the gear icon again and select **Edit Encryption Options**. Enable **User Session Defaults** and confirm.
4. Reopen the same menu, disable **User Session Defaults**, remove the `nofail` option from the options field, enter your passphrase, and confirm.

Then open **Snapper GUI**:

1. Click the disk icon in the top-left corner to configure snapshot limits to a manageable level.
2. Create a new pre-boot configuration for the root filesystem.
3. Create an initial pre-boot root (`/`) snapshot with `number` as the cleanup algorithm.
4. Set `NUMBER_LIMIT` in the name field with a value of `3`.
5. Confirm and proceed.

The system is now ready for a full update including kernel upgrades. The additional space reserved during partitioning ensures that future updates will not exhaust the boot partition, maintaining long-term system stability.

---

## Optional: Migrating to USB

If relocation of the installation to a USB device is desired after the fact:

1. Remove the internal NVMe and place it in an external NVMe-to-USB adapter.
2. Boot a separate Kali live image and identify source and target devices.
3. Execute:

```bash
sudo ddrescue /dev/sdc1 /dev/sdd1 rescue-1.log
sudo ddrescue /dev/sdc2 /dev/sdd2 rescue-2.log
sudo ddrescue /dev/sdc3 /dev/sdd3 rescue-3.log
```

Where `sdc` is the source and `sdd` is the target. The `--force` flag may be required.

> **Note:** If the original NVMe is subsequently reinstalled in the internal slot while the portable adapter is also connected, boot failures may occur. Both disks will share identical UUIDs, and GRUB2 will default to resolving those found earliest in the device stack — which will not correspond to the parameters specified in `initramfs.conf`.

---

*~h1-t3k~ (n0|l1f3)*
