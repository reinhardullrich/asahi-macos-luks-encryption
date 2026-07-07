# Asahi macOS LUKS Encryption

A safety-first runbook for encrypting an existing Fedora Asahi Remix installation on Apple Silicon with LUKS2, using the recommended from-macOS offline helper-VM path.

## Operating Assumption

This repo assumes an AI agent/model is running in macOS during most of the work. Examples are Codex, Cloud Code, or another local AI agent that can run commands, inspect outputs, prepare tools, start the helper VM, and monitor progress with explicit human authorization.

The human still handles the parts that should not be delegated: backup confirmation, administrator approvals, destructive confirmation, LUKS passphrase entry, physical boot selection, first unlock, Fedora Asahi finalization, and cleanup authorization.

## Documents

- [`asahi-luks-encryption-runbook.md`](asahi-luks-encryption-runbook.md): full technical runbook and source cross-checks
- [`human-interaction-guide.md`](human-interaction-guide.md): only the human actions required when an AI agent handles the macOS-side work

## Workflow Improvements In This Playbook

This playbook is intentionally a little stricter than simply following the upstream helper's printed instructions. The small improvements are:

- The AI stages the pinned `asahi-luks-setup` script into the encrypted Fedora Asahi root as `/usr/local/sbin/asahi-luks-setup` before the helper VM powers off. This makes the later `finalize` command available inside Fedora Asahi.
- The guide explicitly tells the user to ignore the helper VM's upstream `When finished, type: poweroff` message until the finalization tool has been copied and verified.
- The guide uses a full shutdown before holding the power button for Startup Options. It does not rely on pressing Restart and then trying to catch the boot picker during reboot.
- The guide treats macOS partition numbers such as `disk0sN` as temporary. If the Mac reboots, partition discovery must be repeated before any write-capable command.
- The guide separates the human-only steps from the AI-handled macOS work, so the user knows exactly when to approve, type a passphrase, choose a boot option, or stop.
- The guide pins the helper project to release `v0.1.1` and uses the current Alpine aarch64 UEFI cloud-init helper image path instead of stale Fedora or old Alpine VM images.
- The guide explicitly protects macOS, System Recovery, the iBoot/System container, and the Asahi APFS stub from cleanup or write operations.
- Cleanup is limited to VM/helper/download artifacts and only happens after Fedora Asahi and macOS both boot successfully.

## Scope

This repository is documentation only. It does not contain an installer or automation script.

The runbook focuses on:

- using `pjordanandrsn/asahi-luks-setup` release `v0.1.1`
- running the encryption from macOS through `asahi-luks-mac` and QEMU
- avoiding writes to macOS, System Recovery, the iBoot/System container, and the Asahi APFS stub
- verifying the correct Fedora Asahi root partition before any destructive step
- recovery notes for interrupted LUKS reencryption

## Warning

Encrypting an existing root partition rewrites the partition in place. Have a reliable backup, stay on power, prevent sleep, and do not run the encryption step until the target partition has been freshly identified and explicitly confirmed.
