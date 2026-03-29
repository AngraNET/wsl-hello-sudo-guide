# WSL Hello Sudo — Install from Source

A step-by-step guide to building and installing [lzlrd/wsl-hello-sudo](https://github.com/lzlrd/wsl-hello-sudo) from source on WSL2.

Replaces the sudo password prompt with **Windows Hello** (fingerprint, face, or PIN).

## What this does

When you run `sudo` in WSL, instead of typing a password you get a native Windows Hello dialog — fingerprint, face recognition, or PIN. Authentication is backed by the Windows TPM/secure enclave via the `KeyCredentialManager` API, so the private key never leaves the hardware.

The setup has two compiled components:
- A **PAM module** (`pam_wsl_hello.so`) that intercepts sudo auth on the Linux side, generates a challenge, and verifies a cryptographic signature
- A **Windows companion exe** (`WindowsHelloBridge.exe`) that invokes Windows Hello, signs the challenge with your TPM-bound key, and returns the signature

They talk to each other over a temp file + stdout — no network, no sockets.

## About the guide

`wsl-hello-sudo-agent-install-guide.md` is written to be followed by an **AI coding agent** (Claude, Copilot, Cursor, or any agent with shell access), not manually by the user. Hand it to your agent and say "read this and guide me through the installation." The agent will:

1. Run all commands autonomously via its shell tool
2. Only ask for your input at interactive steps (Rust installer, Windows Hello enrollment, installer prompts)
3. **Perform a full security audit of the entire source tree** before building — reading every source file, shell script, and Cargo manifest to check for network calls, shell injection, unexpected file writes, or credential exfiltration. The agent will not proceed if anything suspicious is found.
4. On upgrades, diff what changed, re-audit only the changed files, and rebuild only what's necessary

The security audit is re-run on every install and every update — not skipped because "it was clean last time."

## What's in this repo

- `wsl-hello-sudo-agent-install-guide.md` — Full guide covering:
  - Building the PAM module and Windows companion exe from source
  - Security audit checklist (run after every clone or update)
  - Installation walkthrough with interactive prompt answers
  - Upgrading to new versions (selective rebuild based on what changed)
  - Known issues and fixes

## Credits

- [lzlrd/wsl-hello-sudo](https://github.com/lzlrd/wsl-hello-sudo) — maintained fork
- [nullpo-head/WSL-Hello-sudo](https://github.com/nullpo-head/WSL-Hello-sudo) — original project
