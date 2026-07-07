# Asahi macOS LUKS Encryption

A safety-first runbook for encrypting an existing Fedora Asahi Remix installation on Apple Silicon with LUKS2, using the recommended from-macOS offline helper-VM path.

## Operating Assumption

This repo assumes an AI agent/model is running in macOS during most of the work. Examples are Codex, Cloud Code, or another local AI agent that can run commands, inspect outputs, prepare tools, start the helper VM, and monitor progress with explicit human authorization.

The human still handles the parts that should not be delegated: backup confirmation, administrator approvals, destructive confirmation, LUKS passphrase entry, physical boot selection, first unlock, Fedora Asahi finalization, and cleanup authorization.

## Documents

- [`asahi-luks-encryption-runbook.md`](asahi-luks-encryption-runbook.md): full technical runbook and source cross-checks
- [`human-interaction-guide.md`](human-interaction-guide.md): only the human actions required when an AI agent handles the macOS-side work

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
