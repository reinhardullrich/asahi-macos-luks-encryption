# Human Interaction Guide for Asahi LUKS Encryption

Date prepared: 2026-07-07

This document lists only the actions a human must perform during the Fedora Asahi LUKS encryption workflow.

Assumption: an AI agent such as Codex, Cloud Code, or another local AI agent is already running in macOS and has permission to do local macOS-side work. The AI handles downloads, source checks, QEMU preparation, partition discovery, preflight, helper VM startup, command monitoring, and cleanup only after authorization.

The human still has to perform several actions because they involve physical access, private secrets, irreversible consent, or rebooting away from macOS.

## Visible Terminal Rule

Any step that may require human typing must run in a visible macOS Terminal window that the human can see and interact with.

This includes:

- macOS administrator password prompts
- `sudo` prompts
- destructive `YES` confirmations
- LUKS passphrase entry
- helper VM login and console commands
- first-boot unlock or recovery commands

The AI must not run these interactive steps only inside a hidden tool terminal. If the AI starts a hidden command and it asks for a password or passphrase, the AI must cancel it and reopen the command in a visible external Terminal window.

The human should never type passwords or passphrases into chat. They belong only in the visible Terminal, macOS prompt, helper VM console, or Fedora boot prompt.

## Human Responsibilities at a Glance

The human must:

1. Confirm that a backup exists.
2. Keep the Mac on power and awake.
3. Approve macOS administrator prompts when appropriate.
4. Read and explicitly confirm the exact target partition.
5. Type the destructive `YES` confirmation.
6. Choose and type the LUKS passphrase.
7. Wait for the AI to stage the finalization tool inside the encrypted Fedora root.
8. Reboot into Fedora Asahi manually.
9. Unlock the encrypted Asahi installation at boot.
10. Allow SELinux relabeling to finish.
11. Run the finalization step inside Fedora Asahi.
12. Verify that Fedora Asahi and macOS both still boot.
13. Authorize cleanup of temporary VM files.

The human must not:

- type the LUKS passphrase into chat
- type a password or passphrase into a hidden AI/tool terminal
- approve encryption before seeing the exact target partition
- approve anything that targets `/dev/disk0` as a whole disk
- use Disk Utility for partition decisions
- close the Terminal window during encryption
- interrupt the encryption, relabeling, or finalization steps
- reboot before the AI confirms the finalization tool was staged
- delete or modify Asahi, macOS, iBoot/System, or System Recovery partitions

## Before the AI Starts

The AI asks whether a backup exists.

The human must answer clearly:

```text
Yes, I have a backup and I am ready to continue.
```

If there is no backup, the human must answer:

```text
Stop. I do not have a backup.
```

Do not continue without a backup.

The human must connect the Mac to power.

The human must choose a LUKS passphrase before the encryption starts. This passphrase must be memorable and strong. It will be needed every time Fedora Asahi boots.

Do not send the passphrase to the AI. Do not paste it into chat. The passphrase is typed only into the helper VM or Fedora Asahi boot prompt when the prompt asks for it.

## When macOS Asks for Administrator Permission

The AI may need administrator permission for read-only raw partition checks and later for QEMU raw partition access.

macOS may ask for:

- Touch ID
- the macOS user password
- a Terminal password prompt for `sudo`

The human should approve only if the AI has just explained what operation is being performed.

Before approving, the human must be able to see the actual prompt in a normal macOS Terminal window or macOS system prompt. If the AI says a hidden terminal is waiting for a password, stop and ask the AI to reopen the command in a visible Terminal window.

Safe examples:

```text
I am approving a preflight raw-byte check.
```

```text
I am approving the confirmed encryption run for the shown Fedora Asahi root partition.
```

Stop if the request appears before partition discovery or if the AI has not explained why administrator permission is needed.

Never type the macOS password into chat. Type it only into the macOS prompt or Terminal prompt.

## Partition Confirmation

Before encryption, the AI must show a summary like this:

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

The human must read this carefully.

The human should check:

- the root target is a partition such as `/dev/disk0s6`, not `/dev/disk0`
- the root target is described as btrfs
- the APFS stub is explicitly listed as not modified
- System Recovery is explicitly listed as not touched
- iBoot/System container is explicitly listed as not touched
- `/boot` and EFI are separate from the root target
- the partition list was captured after the most recent reboot

If anything looks wrong, the human must say:

```text
Stop. The partition summary does not look safe.
```

Only if everything looks correct, the human types this exact confirmation in chat:

```text
I confirm that /dev/disk0sX is the Fedora Asahi btrfs root partition and I authorize encryption.
```

Replace `X` with the actual number shown by the AI.

Do not give a vague confirmation like:

```text
Looks fine.
```

The confirmation must name the exact partition.

## Before the Irreversible Step

The AI starts sleep prevention and a durable Terminal session.

The human must:

1. Keep the Mac plugged into power.
2. Keep the lid open if this is a MacBook, unless external power and clamshell mode are known to be stable.
3. Leave the Terminal window open.
4. Do not quit Codex.
5. Do not quit Terminal.
6. Do not reboot.
7. Do not let anyone else use the Mac during encryption.

The human should be ready to stay nearby until the passphrase prompts have been completed.

## The `YES` Confirmation

When `asahi-luks-mac` reaches the destructive confirmation, it asks for a typed confirmation.

The human must verify one last time that the target displayed on screen is the same Fedora Asahi root partition previously confirmed.

If the target is correct, the human types:

```text
YES
```

Use uppercase letters.

If the target is not correct, the human types nothing and tells the AI:

```text
Stop. The final target confirmation is wrong.
```

The AI should not type `YES` on behalf of the human.

## LUKS Passphrase Entry

During encryption, the helper VM asks for the new LUKS passphrase.

The human must type the passphrase directly into the helper VM or Terminal prompt.

Usually the prompt asks twice:

```text
Enter passphrase:
Verify passphrase:
```

The human must type the same passphrase both times.

Expected behavior:

- the passphrase may not be visible
- there may be no dots or stars
- pressing Enter submits the passphrase
- if the two entries do not match, the tool may ask again

Do not paste the passphrase into chat.

Do not ask the AI to remember the passphrase.

If the human is not certain what was typed, stop before continuing and repeat the prompt if the tool allows it.

After the passphrase is accepted, the human should write it down or store it in a password manager before rebooting. Losing this passphrase can lock the human out of Fedora Asahi.

## During Encryption

Once encryption begins, the human must wait.

The human must not:

- close the Terminal window
- quit Codex
- put the Mac to sleep
- unplug power
- restart the Mac
- force quit QEMU
- press Control-C
- close the lid

The AI watches progress in macOS.

The human only needs to intervene if the AI asks for a specific confirmation or if macOS displays a permission prompt.

If the Mac loses power or the encryption is interrupted, do not reinstall and do not improvise. Tell the AI exactly what happened. The correct recovery path is usually resume or repair, not a new attempt from scratch.

## When the Helper VM Finishes

At the end of the helper VM step, the screen should indicate that encryption and boot wiring are complete.

Before the helper VM powers off, the AI must stage the finalization tool into the encrypted Fedora Asahi root.

The helper VM may show an upstream message saying something like:

```text
When finished, type: poweroff
```

Do not follow that message yet. In this playbook, encryption is not finished until the finalization tool has also been staged.

Reason: the offline encryption script sets up the encrypted boot, but the final command later is:

```sh
sudo /usr/local/sbin/asahi-luks-setup finalize
```

That file must already exist inside Fedora Asahi before the first boot.

The AI should reopen the encrypted root in the helper VM, copy the pinned `asahi-luks-setup` script into:

```text
/usr/local/sbin/asahi-luks-setup
```

The human may need to type the new LUKS passphrase one more time so the helper VM can reopen the encrypted root for this copy step.

The human should wait until the AI confirms:

```text
Finalization tool staged successfully.
```

The AI may then ask the helper VM to power off.

The human must wait for the AI to say that the helper VM is off and it is time to reboot.

Do not reboot before the AI confirms that the helper VM has finished.

Do not reboot if the AI says the finalization tool could not be staged.

## Rebooting Into Fedora Asahi

This is a human-only step because rebooting leaves macOS and the macOS AI session.

When the AI says it is time:

1. Shut down the Mac completely.
2. Wait until the screen is fully off and the Mac is no longer restarting.
3. From the powered-off state, press the power button and keep holding it.
4. Keep holding until the startup options screen appears.
5. Choose the Fedora/Asahi boot option.
6. Do not choose macOS for this first encrypted boot.

Do not use a normal Restart and then try to hold the power button during the reboot. On Apple Silicon, the safer path is a full shutdown first, then press and hold the power button while starting from off.

If the Fedora/Asahi option is missing, stop. Boot macOS again and tell the AI what appeared.

## First LUKS Unlock at Boot

Fedora Asahi should ask for the LUKS passphrase early in boot.

The prompt may be obvious, or it may be partly hidden in boot text.

If the screen looks paused or stuck, try this:

1. Press Enter once.
2. Wait a few seconds.
3. Type the LUKS passphrase.
4. Press Enter.

Expected behavior:

- the prompt may show `***` while typing
- the prompt may show nothing while typing
- boot may continue only after pressing Enter

If the passphrase is rejected:

1. Check keyboard layout mentally.
2. Try again slowly.
3. Make sure Caps Lock is not on.
4. If it fails repeatedly, stop and boot macOS for recovery help.

Do not run random recovery commands from the emergency shell.

## SELinux Relabel

After the first successful unlock, Fedora Asahi should perform SELinux relabeling.

The human must wait.

This can take a while.

Expected behavior:

- many text lines may appear
- the system may look busy
- it may reboot automatically when finished

The human must not interrupt this process.

If it reboots, choose Fedora/Asahi again if the boot picker appears, then enter the LUKS passphrase again.

## Logging Into Fedora Asahi

After relabeling completes, Fedora Asahi should reach the normal login screen or desktop.

The human logs in normally.

If the system reaches an emergency shell, stop and write down the exact message. Do not run disk repair commands unless the runbook or AI explicitly instructs it.

## Finalization Inside Fedora Asahi

Finalization is required.

The AI running in macOS cannot perform this step while the Mac is booted into Fedora Asahi.

The human must open a Fedora terminal.

First check whether the tool exists:

```sh
ls -l /usr/local/sbin/asahi-luks-setup
```

This file should exist because the AI staged it before the helper VM powered off.

If the file exists, run:

```sh
sudo /usr/local/sbin/asahi-luks-setup finalize
```

If Fedora asks for the user's password, type the Fedora user password into the terminal. Do not type it into chat.

Expected result:

- finalization removes temporary boot arguments
- finalization restores normal boot appearance
- finalization rebuilds initramfs on the real Apple Silicon hardware
- finalization ends without an error

If `/usr/local/sbin/asahi-luks-setup` does not exist, stop and do not improvise. This means the required staging step did not happen or failed. Boot back into macOS and tell the AI:

```text
Finalize tool is missing in Fedora Asahi.
```

The AI must then use the helper/recovery path to mount the encrypted root and copy the exact pinned script into `/usr/local/sbin/asahi-luks-setup`.

## Reboot After Finalization

After finalization completes, the human must reboot Fedora Asahi.

On the next boot:

1. Choose Fedora/Asahi if the boot picker appears.
2. Enter the LUKS passphrase.
3. Confirm Fedora reaches the normal login screen.
4. Log in.

The boot should be cleaner than the first post-encryption boot.

## Human Verification in Fedora Asahi

After the final reboot, the human should open a Fedora terminal and run:

```sh
lsblk -f
findmnt /
findmnt /home
cat /etc/crypttab
cat /proc/cmdline
getenforce
```

The human does not need to fully interpret every line alone, but should check the basics:

- root should involve `crypto_LUKS` or `/dev/mapper/luks-root`
- `/` should mount through `/dev/mapper/luks-root`
- `/home` should mount through `/dev/mapper/luks-root`
- `/etc/crypttab` should mention `luks-root`
- `/proc/cmdline` should contain `rd.luks.name`
- `getenforce` should print `Enforcing`

If anything looks wrong, copy the output to a note or photo and boot back to macOS to ask the AI.

Do not paste secrets into chat. These commands should not reveal the LUKS passphrase.

## Confirm macOS Still Boots

After Fedora Asahi has booted successfully at least twice, the human should confirm macOS still boots.

Steps:

1. If macOS is already the default boot option, a normal restart is fine.
2. If the startup options screen is needed, shut down the Mac completely.
3. Wait until the screen is fully off.
4. From the powered-off state, press the power button and keep holding it until startup options appear.
5. Choose macOS.
6. Log into macOS.
7. Confirm files and normal macOS behavior look correct.
8. Reopen Codex or the AI agent if needed.

Do not rely on pressing Restart and then holding the power button if you need the startup options screen. The safer Apple Silicon path is full shutdown first, then press and hold while starting from off.

If macOS does not boot, stop and do not delete anything.

## Cleanup Authorization

Only after:

- Fedora Asahi has booted successfully after encryption
- finalization has completed
- Fedora Asahi has booted successfully again
- macOS has booted successfully

the AI may ask to clean up temporary files.

Safe cleanup examples:

- Alpine helper image
- QEMU overlay files
- cloud-init seed ISO
- temporary downloaded tool archives
- temporary logs

The human should authorize cleanup only if the AI states that it is deleting VM/helper/download artifacts, not real Asahi or macOS partitions.

Safe cleanup approval:

```text
I authorize cleanup of temporary VM helper files only.
```

Do not authorize cleanup if the AI mentions:

- Asahi root partition
- Asahi `/boot`
- Asahi EFI
- Asahi APFS stub
- macOS APFS
- System Recovery
- iBoot/System container
- `/dev/disk0`

## Stop Conditions

The human must stop the process immediately if any of these happen:

- the target partition is unclear
- the displayed target is a whole disk such as `/dev/disk0`
- System Recovery appears as a target
- iBoot/System container appears as a target
- the Asahi APFS stub appears as a write target
- the partition IDs come from before a reboot
- the Mac is low on battery or loses power
- the LUKS passphrase was forgotten before first boot
- the LUKS prompt rejects the passphrase repeatedly
- Fedora drops into an emergency shell
- finalization fails
- macOS does not boot after the procedure

When stopping, the human should preserve the screen state if possible:

1. Do not close the terminal.
2. Take a photo or copy the exact error text.
3. Tell the AI what happened.
4. Do not run improvised disk commands.

## Final Human Success Criteria

The human is done only when all of these are true:

- Fedora Asahi asks for the LUKS passphrase at boot.
- The passphrase unlocks Fedora Asahi.
- Fedora Asahi reaches the normal login screen.
- `finalize` has completed.
- Fedora Asahi boots cleanly after finalization.
- macOS still boots normally.
- Temporary helper cleanup has been explicitly authorized or intentionally postponed.
