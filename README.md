# WSL Hello Sudo — Install from Source

A step-by-step guide to building and installing [lzlrd/wsl-hello-sudo](https://github.com/lzlrd/wsl-hello-sudo) from source on WSL2.

Replaces the sudo password prompt with **Windows Hello** (fingerprint, face, or PIN).

## What's in this repo

- `how_to_enable_and_upgrade.md` — Full guide covering:
  - Building the PAM module and Windows companion exe from source
  - Security audit checklist (run after every clone or update)
  - Installation walkthrough with interactive prompt answers
  - Upgrading to new versions (selective rebuild based on what changed)
  - Known issues and fixes

## Credits

- [lzlrd/wsl-hello-sudo](https://github.com/lzlrd/wsl-hello-sudo) — maintained fork
- [nullpo-head/WSL-Hello-sudo](https://github.com/nullpo-head/WSL-Hello-sudo) — original project
