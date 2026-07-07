# Asahi macOS LUKS Encryption

A safety-first runbook for encrypting an existing Fedora Asahi Remix installation on Apple Silicon with LUKS2, using the recommended from-macOS offline helper-VM path.

The main guide is [`asahi-luks-encryption-runbook.md`](asahi-luks-encryption-runbook.md).

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
