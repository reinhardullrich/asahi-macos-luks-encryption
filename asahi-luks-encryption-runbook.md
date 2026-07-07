# Fedora Asahi LUKS Encryption Runbook

Date prepared: 2026-07-07

Purpose: encrypt an existing Fedora Asahi Remix installation on this Apple Silicon Mac using the safer macOS/offline VM method, without touching macOS, System Recovery, or unrelated partitions.

This document is a planning and execution guide. It is intentionally conservative. The actual encryption step must not be run until the partition IDs have been shown and explicitly confirmed.

Operating assumption: an AI agent such as Codex, Cloud Code, or another local AI agent is running in macOS and can perform the macOS-side work with explicit human authorization. That agent can download tooling, check sources, prepare QEMU, discover partitions, run preflight, start and monitor the helper VM, and clean up temporary helper files. The human still performs physical actions, administrator approvals, destructive confirmations, passphrase entry, reboot selection, first unlock, and Fedora Asahi finalization.

## Short Summary

We will use the `pjordanandrsn/asahi-luks-setup` project, specifically the `asahi-luks-mac` launcher.

The existing UTM Fedora VM is useful as a general Linux VM, but it is not the preferred tool for the real encryption. The real method is:

1. Download a small Alpine aarch64 helper image.
2. Let `asahi-luks-mac` start that image directly with QEMU, not UTM.
3. Pass only the real Asahi partitions into that temporary helper VM.
4. Run `asahi-luks-setup --offline --device /dev/vdb encrypt` inside that helper VM.
5. Before leaving the helper VM, copy the pinned `asahi-luks-setup` script into the encrypted Fedora root as `/usr/local/sbin/asahi-luks-setup`.
6. Reboot into Fedora Asahi, unlock with the new LUKS passphrase, let SELinux relabel, then run `finalize`.

The key point: the helper VM does not magically see the Mac disk. We explicitly attach the correct Asahi partitions to it. That explicit attachment is the powerful and dangerous part.

## Sources Checked

Primary source:

- `pjordanandrsn/asahi-luks-setup`: https://github.com/pjordanandrsn/asahi-luks-setup
- Release `v0.1.1`: https://github.com/pjordanandrsn/asahi-luks-setup/releases/tag/v0.1.1
- Upstream Asahi installer feature request for LUKS: https://github.com/AsahiLinux/asahi-installer/issues/137

Background sources:

- Coffee Junk gist, "Asahi Full Disk Encryption": https://gist.github.com/coffeejunk/cc7c6d87fd7366bfe037d4be9e37ce4c
- David Alger, "Fedora Asahi Remix with LUKS Encryption": https://davidalger.com/posts/fedora-asahi-remix-on-apple-silicon-with-luks-encryption/
- `osx-tools/asahi-encrypt`: https://github.com/osx-tools/asahi-encrypt
- Asahi partitioning cheatsheet: https://github.com/AsahiLinux/docs/blob/main/docs/sw/partitioning-cheatsheet.md
- Asahi security notes: https://github.com/AsahiLinux/docs/blob/main/docs/platform/security.md
- Asahi boot process guide: https://github.com/AsahiLinux/docs/blob/main/docs/alt/boot-process-guide.md
- Asahi installer issue about duplicate UUIDs: https://github.com/AsahiLinux/asahi-installer/issues/265
- `cryptsetup-reencrypt(8)`: https://man7.org/linux/man-pages/man8/cryptsetup-reencrypt.8.html
- `dracut.cmdline(7)`: https://man7.org/linux/man-pages/man7/dracut.cmdline.7.html
- Alpine cloud image directory checked on 2026-07-07: https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/cloud/

Important cross-check results:

- The newer `asahi-luks-setup` repo explicitly says the from-macOS offline VM path is the recommended path in v0.1.1.
- That release says the from-macOS path was hardware-validated on Apple Silicon hardware with Fedora Asahi Remix 44.
- The older Coffee Junk and David Alger guides use the same core idea: shrink btrfs by 32 MB, run `cryptsetup reencrypt`, then wire up `crypttab`, GRUB, dracut, and SELinux relabeling.
- The newer repo adds important safeguards missing from the older guides: target magic checks, idempotent resume behavior, mapper-name consistency with `rd.luks.name`, SELinux handling, recovery mode, and better interruption handling.
- The official Asahi partitioning docs add two critical safety constraints: do not touch the first iBoot/System container or the last `Apple_APFS_Recovery` partition, and do not trust old `disk0sN` numbers after a reboot because macOS can renumber them.
- A Fedora Asahi install is not just root, `/boot`, and EFI. It also has a 2.5 GB APFS stub container tied to the install. We identify it so we do not touch it.
- Asahi issue 265 documents same-edition side-by-side installs with identical filesystem UUIDs. Do not select the target by UUID alone.
- `cryptsetup-reencrypt(8)` confirms that `--encrypt`, `--reduce-device-size`, resume, and repair are real supported primitives, but it also explicitly requires reliable backups.
- `dracut.cmdline(7)` confirms `rd.luks.name=<uuid>=<name>`, which is why this guide prefers the stable mapper name `luks-root`.
- The Alpine image filename in the repo's VM test README was stale when checked on 2026-07-07. The old `nocloud_alpine-3.23.4-aarch64-uefi-cloudinit-r0.qcow2` URL returned 404. The current mirror listed Alpine 3.24.1 generic aarch64 UEFI cloud-init qcow2 images.
- The upstream Asahi installer LUKS feature request is still open, so this is not a routine built-in installer checkbox yet.

## What This Will Encrypt

This encrypts only the Fedora Asahi btrfs root partition.

Expected standard Fedora Asahi logical group:

```text
Asahi APFS stub     APFS   about 2.5 GB          remains unmodified
Asahi EFI           vfat   mounted as /boot/efi  remains unencrypted
Asahi /boot         ext4   mounted as /boot      remains unencrypted
Asahi root          btrfs  mounted as / and /home encrypted with LUKS2
```

The APFS stub, EFI, `/boot`, and root partitions should be treated as one Asahi install group for identification purposes. The encryption target is only the btrfs root partition.

This does not encrypt:

- macOS APFS volumes
- System Recovery
- iBoot/System container
- Asahi APFS stub
- Asahi EFI
- Asahi `/boot`
- any UTM virtual disks
- any unrelated downloaded images

Important: official Asahi documentation says the last `Apple_APFS_Recovery` partition is critical. Do not delete, resize, mount for writes, or otherwise modify it during this procedure.

## High-Level Architecture

There are three separate things:

1. UTM Fedora VM
   - Already working.
   - Useful as a normal Linux VM.
   - Not the preferred tool for raw Asahi partition encryption.

2. Alpine helper image
   - Small, temporary Linux image.
   - Started directly by QEMU through `asahi-luks-mac`.
   - Prepared automatically with `cryptsetup`, `btrfs-progs`, and related tools.
   - Used only during the encryption/recovery procedure.

3. Real Fedora Asahi install
   - Lives on the Mac internal disk.
   - Its root partition is passed into the helper VM as `/dev/vdb`.
   - Its `/boot` and EFI partitions are passed in too so the script can update boot files.

Device order inside the helper VM:

```text
/dev/vda  helper VM overlay/root disk
/dev/vdb  real Asahi btrfs root partition, the encryption target
/dev/vdc  cloud-init seed
/dev/vdd  real Asahi /boot partition
/dev/vde  real Asahi EFI partition
```

The encryption command inside the helper VM is:

```sh
bash /mnt/alts/asahi-luks-setup --offline --device /dev/vdb encrypt
```

## Strict Safety Rules

Do not run the encryption until all of these are true:

- A backup exists.
- The Mac is connected to power.
- Sleep is disabled for the duration of the operation.
- A fresh `diskutil list internal physical` was captured after the most recent reboot.
- The exact Asahi APFS stub, root, `/boot`, and EFI partition IDs have been identified.
- The root partition has passed a btrfs magic check.
- The partition list has been shown to the user.
- The user explicitly confirms the exact target partition.
- Selection is based on partition role, size, physical grouping, and filesystem magic, not on UUID alone.

Never allow commands to write to:

- macOS APFS partitions
- System Recovery
- iBoot/System container
- Asahi APFS stub
- the wrong Linux partition
- a whole-disk device such as `/dev/disk0` instead of a partition such as `/dev/disk0s6`

The root target should be a partition device such as:

```text
/dev/disk0s6
```

It should not be:

```text
/dev/disk0
```

Do not reuse old partition IDs from notes or screenshots. macOS can renumber `disk0sN` identifiers after a reboot or partition table change. If the Mac has rebooted since partition discovery, redo discovery.

## Required Tools

On macOS:

- QEMU from Homebrew, providing `qemu-system-aarch64`
- QEMU AArch64 UEFI firmware, usually:

```text
/opt/homebrew/share/qemu/edk2-aarch64-code.fd
```

- `asahi-luks-mac`
- `asahi-luks-setup`
- Alpine aarch64 UEFI cloud-init helper image

Inside the helper VM:

- `cryptsetup`
- `btrfs-progs`
- `util-linux`
- modules for ext4 and vfat

The `asahi-luks-mac` script installs these inside Alpine using `apk`, which is why Alpine is the natural helper image. A Fedora helper could work in theory, but the current script is Alpine-shaped.

## Phase 0: Do Not Touch Disks Yet

Before starting:

1. Fully quit any unrelated UTM/VM activity unless intentionally using it.
2. Make sure the target Asahi root, `/boot`, and EFI partitions are not mounted by macOS.
3. Keep the existing UTM Fedora VM untouched unless it is needed for unrelated reference checks.
4. Do not delete any Asahi partitions.
5. Do not run any raw-device command with write access.
6. Do not use the graphical Disk Utility app for partition decisions. Use `diskutil` output.

## Phase 1: Download and Pin the Tooling

Use release `v0.1.1` unless there is a strong reason to inspect newer `main`.

Recommended local working directory:

```text
~/alts-vmtest/asahi-luks-setup
```

Example commands for later execution:

```sh
mkdir -p ~/alts-vmtest/asahi-luks-setup
cd ~/alts-vmtest/asahi-luks-setup
curl -L -O https://raw.githubusercontent.com/pjordanandrsn/asahi-luks-setup/v0.1.1/asahi-luks-mac
curl -L -O https://raw.githubusercontent.com/pjordanandrsn/asahi-luks-setup/v0.1.1/asahi-luks-setup
curl -L -O https://raw.githubusercontent.com/pjordanandrsn/asahi-luks-setup/v0.1.1/README.md
chmod +x asahi-luks-mac asahi-luks-setup
./asahi-luks-mac --version
```

The raw tagged downloads avoid relying on the GitHub API tarball redirect and avoid wildcard-renaming an extracted directory.

Alternative if using git:

```sh
cd ~/alts-vmtest
git clone https://github.com/pjordanandrsn/asahi-luks-setup.git
cd asahi-luks-setup
git checkout v0.1.1
```

Why pin:

- Disk encryption work should avoid surprise changes from a moving branch.
- Release `v0.1.1` documents hardware validation and includes the recovery subcommand.

## Phase 2: Install or Verify QEMU

Check:

```sh
command -v qemu-system-aarch64
qemu-system-aarch64 --version
ls /opt/homebrew/share/qemu/edk2-aarch64-code.fd
```

If QEMU is missing:

```sh
brew install qemu
```

This is a macOS host requirement. It is independent of UTM.

## Phase 3: Download the Alpine Helper Image

The repo's VM test README used an Alpine 3.23.4 filename, but that specific URL returned 404 when checked on 2026-07-07.

Use the current generic Alpine aarch64 UEFI cloud-init qcow2 listed in the Alpine mirror. On 2026-07-07, the relevant current filename was:

```text
generic_alpine-3.24.1-aarch64-uefi-cloudinit-r0.qcow2
```

Download example:

```sh
mkdir -p ~/alts-vmtest
cd ~/alts-vmtest
curl -L -O https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/cloud/generic_alpine-3.24.1-aarch64-uefi-cloudinit-r0.qcow2
curl -L -O https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/cloud/generic_alpine-3.24.1-aarch64-uefi-cloudinit-r0.qcow2.sha512
```

Verify checksum:

```sh
cd ~/alts-vmtest
shasum -a 512 -c generic_alpine-3.24.1-aarch64-uefi-cloudinit-r0.qcow2.sha512
cp generic_alpine-3.24.1-aarch64-uefi-cloudinit-r0.qcow2 alpine-aarch64.qcow2
```

The checksum command should print `OK`.

Expected helper image path for the script:

```text
~/alts-vmtest/alpine-aarch64.qcow2
```

`asahi-luks-mac` can auto-detect this path, but passing it explicitly is clearer.

## Phase 4: Read-Only macOS Partition Discovery

Goal: identify the exact Asahi partitions before any write-capable operation.

Useful read-only commands:

```sh
diskutil list internal physical
diskutil list
```

Save this output in the working notes for this exact run. If the Mac reboots before encryption, throw this output away and capture a new one. The `disk0sN` identifiers are not stable.

For each candidate partition:

```sh
diskutil info /dev/disk0sN
```

Expected candidates:

- Asahi APFS stub: APFS container, about 2.5 GB, logically tied to the Asahi install, not a target.
- EFI: shown as EFI or vfat/FAT, about 500 MB.
- `/boot`: Linux filesystem, ext4, usually around 1 GB.
- root: Linux filesystem, btrfs, usually the largest Linux partition.
- System Recovery: `Apple_APFS_Recovery`, usually last. Never touch it.
- iBoot/System container: `Apple_APFS_ISC`, usually first. Never touch it.

Important: `asahi-luks-mac` auto-detects the root as the largest internal "Linux Filesystem" partition. That is a useful guess, not an absolute truth. If there are leftover Linux partitions, manual override is safer.

If there are two same-edition Fedora Asahi installs, UUIDs may be duplicated. In that case, selection must rely on current partition position, size, physical grouping with the matching Asahi APFS stub and EFI, and the script's btrfs/LUKS magic checks.

Check whether macOS has mounted any candidate Asahi partitions:

```sh
mount
diskutil info /dev/disk0sX
diskutil info /dev/disk0sY
diskutil info /dev/disk0sZ
```

If a target partition is mounted, stop and unmount it before any write-capable run. Do not browse or modify the partition in Finder.

Manual override options:

```sh
--device /dev/disk0sX
--boot-device /dev/disk0sY
--efi-device /dev/disk0sZ
```

## Phase 5: Read-Only Preflight

If the Mac has rebooted since Phase 4, redo Phase 4 first. Do not carry old `/dev/disk0sN` identifiers into preflight.

First run a dry-run preflight:

```sh
cd ~/alts-vmtest/asahi-luks-setup
./asahi-luks-mac \
  --helper-image ~/alts-vmtest/alpine-aarch64.qcow2 \
  --dry-run \
  preflight
```

Then run the real preflight. This may need admin rights because the script checks raw partition bytes:

```sh
cd ~/alts-vmtest/asahi-luks-setup
sudo ./asahi-luks-mac \
  --helper-image ~/alts-vmtest/alpine-aarch64.qcow2 \
  preflight
```

If using manual partition IDs:

```sh
sudo ./asahi-luks-mac \
  --helper-image ~/alts-vmtest/alpine-aarch64.qcow2 \
  --device /dev/disk0sX \
  --boot-device /dev/disk0sY \
  --efi-device /dev/disk0sZ \
  preflight
```

Preflight should confirm:

- target root candidate
- btrfs superblock magic for a fresh encrypt, or LUKS magic for resume
- `/boot` partition
- `/boot/efi` partition
- QEMU exists
- firmware exists
- helper image exists
- `asahi-luks-setup` is next to `asahi-luks-mac`

Stop if:

- target is ambiguous
- btrfs magic check fails
- the root target is under 8 GB
- the target is a whole disk, not a partition
- `/boot` or EFI looks wrong
- QEMU or firmware is missing
- helper image cannot boot

Optional helper sanity check before attaching real partitions:

```sh
qemu-img info ~/alts-vmtest/alpine-aarch64.qcow2
```

If QEMU or the helper image is suspect, debug that with a throwaway image first. Do not debug QEMU while the real Asahi partitions are attached.

## Phase 6: Explicit Confirmation Gate

Before encryption, show the user a clear summary:

```text
About to encrypt:

Asahi root target: /dev/disk0sX
Size:              <size>
Detected as:       btrfs root candidate

Asahi APFS stub:   /dev/disk0sA
Detected as:       APFS stub, not modified

Asahi /boot:       /dev/disk0sY
Detected as:       ext4 or Linux boot partition

Asahi EFI:         /dev/disk0sZ
Detected as:       EFI/vfat partition

Will not touch:
macOS APFS partitions
System Recovery
iBoot/System container
Asahi APFS stub
UTM virtual disks
other downloads
```

Only continue if the user explicitly confirms.

Suggested confirmation wording:

```text
I confirm that /dev/disk0sX is the Fedora Asahi btrfs root partition and I want to encrypt it.
```

## Phase 7: Prevent Sleep and Use a Durable Session

The encryption can take 15 to 30 minutes or longer, depending on partition size.

Interruptions that can kill QEMU:

- Mac sleeping
- Terminal session closing
- SSH disconnect, if remote
- Terminal restore/restart
- helper files in `/tmp` being removed

Before encrypting:

- Plug in power.
- Disable sleep temporarily in System Settings.
- Keep helper files in `~/alts-vmtest`, not `/tmp`.
- Start `caffeinate` so macOS does not sleep during the long write.
- Prefer `tmux` if available.

Example:

```sh
caffeinate -dimsu
```

Leave that Terminal window open until encryption finishes.

In a second Terminal window, if `tmux` is available:

```sh
tmux new-session -s asahi-luks
```

If the terminal disconnects:

```sh
tmux attach -t asahi-luks
```

If `tmux` is unavailable, use a stable local Terminal session and do not close it.

## Phase 8: Run Encryption

Last gate before write access:

- Re-run `diskutil list internal physical` if anything has changed or the Mac rebooted.
- Confirm the target Asahi root partition is still the same role and size.
- Confirm root, `/boot`, and EFI are not mounted in macOS.
- Confirm the APFS stub, iBoot/System container, and System Recovery are not targets.

Use explicit devices if preflight showed any ambiguity.

Template:

```sh
cd ~/alts-vmtest/asahi-luks-setup
sudo ./asahi-luks-mac \
  --helper-image ~/alts-vmtest/alpine-aarch64.qcow2 \
  --device /dev/disk0sX \
  --boot-device /dev/disk0sY \
  --efi-device /dev/disk0sZ \
  encrypt
```

If auto-detection was unambiguous and confirmed:

```sh
cd ~/alts-vmtest/asahi-luks-setup
sudo ./asahi-luks-mac \
  --helper-image ~/alts-vmtest/alpine-aarch64.qcow2 \
  encrypt
```

What `asahi-luks-mac` does:

1. Confirms target again.
2. Requires a typed `YES`.
3. Builds a cloud-init seed ISO.
4. Creates a disposable helper overlay from the Alpine helper image.
5. Resizes the helper overlay to 8 GB for working space.
6. Launches QEMU/HVF with the real Asahi partitions attached.
7. Shows a helper VM console.
8. Prints a message to log in as `root` with password `asahi`.
9. Instructs the operator to run:

```sh
bash /mnt/alts/asahi-luks-setup --offline --device /dev/vdb encrypt
```

Inside the helper VM, the encryption script:

1. Checks tools.
2. Classifies the target:
   - raw btrfs: fresh encryption
   - LUKS2 in progress: resume
   - complete LUKS2: re-apply boot wire-up only
   - anything else: refuse
3. Performs a keyboard/passphrase prompt check.
4. Shrinks btrfs by 32 MB.
5. Runs:

```sh
cryptsetup reencrypt --encrypt --reduce-device-size 32M --type luks2 /dev/vdb
```

6. Opens the new LUKS container as `luks-root`.
7. Mounts root, home, `/boot`, and EFI.
8. Writes `/etc/crypttab`.
9. Rewrites `/etc/fstab` so `/` and `/home` use `/dev/mapper/luks-root`.
10. Updates GRUB/kernel arguments with:

```text
rd.luks.name=<LUKS_UUID>=luks-root
enforcing=0
```

11. Removes splash/quiet for the first boot so the passphrase prompt and relabel output are visible.
12. Rebuilds GRUB config.
13. Rebuilds initramfs with `dracut --no-hostonly --regenerate-all` because the helper VM hardware is not the target Apple Silicon hardware.
14. Touches:

```text
/.autorelabel
```

15. Runs consistency checks against `crypttab`, `fstab`, and GRUB.
16. Unmounts and closes the LUKS mapper.

Required finalizer staging before leaving the helper VM:

The upstream `v0.1.1` offline `encrypt` path installs the login reminder, but it does not call `self_install` into the target Fedora root. Therefore the helper VM must copy the same script into the encrypted Fedora Asahi system before poweroff. Otherwise the later `finalize` command may not exist.

After `encrypt` completes and returns to the helper VM prompt, run:

```sh
cryptsetup luksOpen /dev/vdb luks-root
mkdir -p /mnt/asahi-root
mount -o subvol=root /dev/mapper/luks-root /mnt/asahi-root
install -D -m 0755 /mnt/alts/asahi-luks-setup /mnt/asahi-root/usr/local/sbin/asahi-luks-setup
ls -l /mnt/asahi-root/usr/local/sbin/asahi-luks-setup
sync
umount /mnt/asahi-root
cryptsetup close luks-root
```

The human may need to type the LUKS passphrase again for `cryptsetup luksOpen`. Do not power off the helper VM until the `ls -l` check shows:

```text
/mnt/asahi-root/usr/local/sbin/asahi-luks-setup
```

When encryption and finalizer staging are both finished inside the helper VM:

```sh
poweroff
```

## Phase 9: First Boot Into Fedora Asahi

Reboot the Mac into Fedora Asahi:

1. Shut down the Mac completely.
2. Wait until the screen is fully off.
3. From the powered-off state, press the power button and keep holding it.
4. Keep holding until startup options appear.
5. Choose Fedora/Asahi.
6. Wait for the LUKS prompt.
7. Enter the passphrase.

Do not use a normal Restart and then try to hold the power button during the reboot. The safer Apple Silicon path is full shutdown first, then press and hold the power button while starting from off.

Prompt behavior note:

The passphrase prompt can be visually buried in boot output and look like a hang. If so:

- start typing the passphrase anyway, or
- press Enter to reprint the prompt.

After unlock:

- SELinux relabel should run.
- The first boot can be slow.
- It may reboot itself once.

## Phase 10: Finalize

After the relabel boot completes and the system reaches Fedora Asahi:

Important operational note: once the Mac is booted into Fedora Asahi, the AI agent running in macOS is not running the commands anymore. The required staging step in Phase 8 must already have copied `asahi-luks-setup` into the encrypted Fedora root.

The offline encryption step installs a login reminder in the target root, but it does not self-install the finalization tool. The guide therefore requires the helper VM copy step before first boot.

Check whether the tool exists:

```sh
ls -l /usr/local/sbin/asahi-luks-setup
```

If it exists:

```sh
sudo /usr/local/sbin/asahi-luks-setup finalize
```

If it does not exist, stop. Do not improvise a replacement command inside Fedora Asahi. Boot macOS again and use the recovery/helper path to mount the encrypted root and install the exact pinned `asahi-luks-setup` script:

```sh
cryptsetup luksOpen /dev/vdb luks-root
mkdir -p /mnt/asahi-root
mount -o subvol=root /dev/mapper/luks-root /mnt/asahi-root
install -D -m 0755 /mnt/alts/asahi-luks-setup /mnt/asahi-root/usr/local/sbin/asahi-luks-setup
sync
umount /mnt/asahi-root
cryptsetup close luks-root
```

Then boot Fedora Asahi again and run `sudo /usr/local/sbin/asahi-luks-setup finalize`.

Finalize should:

- remove `enforcing=0`
- restore `rhgb quiet`
- remove the login reminder
- trim the dracut config
- rebuild initramfs natively on the real Asahi hardware
- regenerate GRUB config

After finalizing, reboot once more and confirm a clean boot.

## Phase 11: Verification

In Fedora Asahi, verify:

```sh
lsblk -f
findmnt /
findmnt /home
cat /etc/crypttab
cat /etc/fstab
cat /proc/cmdline
```

Expected:

- Asahi root partition type is `crypto_LUKS`.
- `/` is mounted from `/dev/mapper/luks-root`.
- `/home` is mounted from `/dev/mapper/luks-root`.
- `/etc/crypttab` contains a `luks-root` mapping.
- kernel command line contains `rd.luks.name=<uuid>=luks-root`.
- `enforcing=0` is gone after finalize.
- `/boot` and EFI remain unencrypted.

Check SELinux:

```sh
getenforce
```

Expected after finalize:

```text
Enforcing
```

Check macOS safety:

- Boot macOS.
- Confirm macOS starts normally.
- Confirm APFS volumes and recovery are untouched.

## Phase 12: Cleanup

Only after two successful Asahi boots and one successful macOS boot:

Safe cleanup candidates:

- Alpine helper image
- QEMU overlay files
- cloud-init seed ISO
- temporary downloaded repo archive
- temporary test logs

Do not delete:

- Fedora Asahi partitions
- Asahi `/boot`
- Asahi EFI
- macOS partitions
- System Recovery
- any user data backups

The existing UTM Fedora VM can be kept or deleted later, but it is unrelated to the encrypted Asahi install.

## Recovery: If Encryption Is Interrupted

Do not reinstall immediately.

The newer script is designed to be idempotent. Re-running `encrypt` classifies the target and does the right thing:

- raw btrfs: fresh run
- LUKS2 with reencryption in progress: resume
- complete LUKS2: re-apply wire-up
- unexpected filesystem: stop

Recovery command from macOS:

```sh
cd ~/alts-vmtest/asahi-luks-setup
sudo ./asahi-luks-mac \
  --helper-image ~/alts-vmtest/alpine-aarch64.qcow2 \
  --device /dev/disk0sX \
  --boot-device /dev/disk0sY \
  --efi-device /dev/disk0sZ \
  recover
```

Inside recovery helper VM:

```sh
bash /mnt/alts/asahi-luks-setup --offline --device /dev/vdb encrypt
```

If cryptsetup says repair is required:

```sh
cryptsetup repair /dev/vdb
bash /mnt/alts/asahi-luks-setup --offline --device /dev/vdb encrypt
```

## Recovery: If First Boot Drops to Emergency Shell

Likely causes:

- LUKS UUID mismatch between GRUB command line and `/etc/crypttab`.
- Mapper name mismatch, for example root opened as `luks-<uuid>` instead of `luks-root`.
- initramfs did not include the needed boot/storage/crypto support.

From recovery VM, inspect:

```sh
cryptsetup luksOpen /dev/vdb luks-root
mkdir -p /mnt/r /mnt/boot
mount -o subvol=root /dev/mapper/luks-root /mnt/r
mount /dev/vdd /mnt/boot
cat /mnt/r/etc/crypttab
cat /mnt/r/etc/fstab
cat /mnt/r/etc/default/grub
ls /mnt/boot/loader/entries/
cat /mnt/boot/loader/entries/*.conf
```

Expected relationship:

```text
/etc/crypttab:
luks-root UUID=<LUKS_UUID> none luks

GRUB/kernel args:
rd.luks.name=<LUKS_UUID>=luks-root

/etc/fstab:
/ and /home should point to /dev/mapper/luks-root
```

If the UUIDs do not match, fix the files, rebuild GRUB/dracut in chroot, then retry boot.

## Recovery: If LUKS Prompt Appears Hidden

This is a known boot UX issue.

Try:

1. Press Enter once to reprint the prompt.
2. Type the passphrase even if the prompt is visually buried.
3. Watch for `***` echoing as you type.

## Recovery: If Keyboard Input Fails at Boot

This is one reason the manual in-Asahi path has a keyboard-check gate. The from-macOS path avoids the pre-mount shell but cannot fully prove the future boot prompt input path until first boot.

If the internal keyboard does not work at the LUKS prompt:

- try an external keyboard if available
- power cycle and retry
- use recovery path from macOS to inspect the encrypted root
- do not rerun random bootloader commands without checking `crypttab`, `fstab`, and GRUB first

## What Not To Use

Avoid the `autoencrypt` mode for this machine unless there is a strong new reason.

Reason:

- The v0.1.1 release marks `autoencrypt` experimental.
- The README documents a prior hardware run that left a Mac mini half-encrypted and unbootable.
- The macOS/offline VM path is the recommended path in the release notes.

Avoid blindly following Coffee Junk's scripts.

Reason:

- They are useful background, but simpler and older.
- Partition detection is less defensive.
- They do not encode the newer gotchas and recovery handling.

Avoid relying only on David Alger's older manual USB guide.

Reason:

- It is conceptually correct and valuable background.
- But it assumes a USB/recovery flow and more manual steps.
- The newer QEMU-from-macOS flow better matches this Mac and current goal.

## Operator Checklist

Preflight checklist:

- [ ] Backup exists.
- [ ] Power connected.
- [ ] Sleep disabled.
- [ ] QEMU installed.
- [ ] Firmware found.
- [ ] Alpine helper image downloaded.
- [ ] Alpine helper checksum verified.
- [ ] `asahi-luks-setup` release pinned.
- [ ] `asahi-luks-mac` executable.
- [ ] Fresh macOS partition list reviewed after the most recent reboot.
- [ ] iBoot/System container identified as not a target.
- [ ] System Recovery identified as not a target.
- [ ] Asahi APFS stub identified as not a target.
- [ ] Asahi root identified.
- [ ] Asahi `/boot` identified.
- [ ] Asahi EFI identified.
- [ ] Target root, `/boot`, and EFI are not mounted in macOS.
- [ ] Duplicate-UUID risk considered if multiple same-edition Fedora Asahi installs exist.
- [ ] btrfs magic confirmed.
- [ ] exact devices shown to user.
- [ ] user explicitly confirms target.

Encryption checklist:

- [ ] `caffeinate` or equivalent sleep prevention is active.
- [ ] Start durable session.
- [ ] Run `asahi-luks-mac encrypt`.
- [ ] Confirm `YES` only after seeing the correct target.
- [ ] Set LUKS passphrase.
- [ ] Wait for encryption completion.
- [ ] Reopen LUKS root in helper VM.
- [ ] Copy `/mnt/alts/asahi-luks-setup` to `/usr/local/sbin/asahi-luks-setup` inside the encrypted Fedora root.
- [ ] Verify the staged finalizer with `ls -l`.
- [ ] Power off helper VM.

First boot checklist:

- [ ] Boot Fedora Asahi.
- [ ] Enter LUKS passphrase.
- [ ] Let SELinux relabel.
- [ ] Let automatic reboot happen if it occurs.
- [ ] Reach Fedora Asahi system.
- [ ] Run `finalize`.
- [ ] Reboot again.
- [ ] Confirm clean encrypted boot.
- [ ] Confirm macOS still boots.

Cleanup checklist:

- [ ] Keep backups.
- [ ] Remove temporary helper files only after successful verification.
- [ ] Do not delete Asahi partitions.
- [ ] Do not delete macOS/System Recovery.

## Final Recommendation

Proceed with the from-macOS offline VM path using `asahi-luks-mac` v0.1.1, but only after a strict preflight and explicit partition confirmation.

Use the existing UTM Fedora VM as a convenience only. It is not the main encryption mechanism.

Use Alpine as the temporary helper image because the actual `asahi-luks-mac` launcher prepares the guest using Alpine package commands.

Treat the whole operation as reversible only until the encryption command begins. Once `cryptsetup reencrypt` starts, the correct recovery approach is to resume or repair the LUKS reencryption, not to improvise new disk writes.
