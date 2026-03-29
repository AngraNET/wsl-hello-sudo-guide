# WSL Hello Sudo — Installation Guide (for Claude)

This document guides Claude through installing `wsl-hello-sudo` from source on a new WSL instance.
The user will ask Claude to read this and walk them through it step by step.

Source: https://github.com/lzlrd/wsl-hello-sudo

---

## How to Use This Guide

Execute each step via Bash tool. Only ask the user for manual intervention when:
- A prompt requires interactive input (installer dialogs, Windows Hello enrollment)
- A WSL restart is needed
- A step fails unexpectedly

Proceed one step at a time and wait for confirmation before moving to the next.

---

## Step 1 — Install Linux build dependencies

```bash
sudo apt update && sudo apt install -y gcc make libpam0g-dev git libssl-dev pkg-config
```

---

## Step 2 — Install Rust

Ask the user to run this (it's interactive):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

They should choose option `1` (default install).

> **After Rust installs, WSL must be restarted.** Tell the user to run `wsl --shutdown` from Windows Terminal, then reopen WSL and come back. The conversation will be lost — that's expected. Tell them to say "continue the wsl-hello-sudo install" and read this file again.

---

## Step 3 — Resume after WSL restart

When the user returns after restart, check if cargo is available:

```bash
source ~/.cargo/env && cargo --version
```

If not found, check `~/.cargo/bin/cargo` exists and source manually.

---

## Step 4 — Add Windows cross-compile target

```bash
rustup target add x86_64-pc-windows-gnu && sudo apt install -y gcc-mingw-w64-x86-64
```

---

## Step 5 — Clone the repo

```bash
git clone https://github.com/lzlrd/wsl-hello-sudo.git ~/src/wsl-hello-sudo
```

If the directory already exists (resumed session), skip the clone.

---

## Step 6 — Security Audit (run after every clone or pull)

Read and audit the entire repository source before building. Do not rely on a file list — new files may have been added. Use the Glob tool to discover all source files, shell scripts, and Cargo manifests, then read and review every one of them. The repo may have changed since the last install.

Also run:

```bash
cd ~/src/wsl-hello-sudo && git log --oneline -20
```

Review recent commits for anything suspicious before continuing.

### What to check for

| Area | What to look for |
|------|-----------------|
| Network calls | Any `reqwest`, `hyper`, `TcpStream`, `UdpSocket`, outbound connections |
| Shell injection | Any `Command::new("sh")`, `cmd /c`, string interpolation into shell args |
| File writes outside expected paths | Writes outside `/etc/pam_wsl_hello/`, `/lib/security/`, Windows install dir |
| Credential exfiltration | Anything reading `/etc/pam_wsl_hello/` keys and sending them anywhere |
| New dependencies in Cargo.toml | Unfamiliar crates — check crates.io for legitimacy |
| install.sh changes | Any new curl/wget calls, new files being placed, changed install paths |

### Expected clean behaviour (baseline from 2026-03-28)

- **PAM module**: Gets PAM username → reads public key from `/etc/pam_wsl_hello/public_keys/` → generates UUID challenge → spawns `WindowsHelloBridge.exe authenticator` → verifies returned signature with OpenSSL SHA256. No network, no shell.
- **Windows bridge (authenticator mode)**: Calls `KeyCredentialManager` Windows Hello API → signs challenge with TPM-bound private key → returns signature on stdout. No network, no file writes.
- **Windows bridge (creator mode)**: Calls `KeyCredentialManager::RequestCreateAsync` → writes public key as `.pem` to current directory only.
- **install.sh**: Copies files, creates `/etc/pam_wsl_hello/config`, runs `pam-auth-update`. No remote calls.

### Verdict

If the audit passes, state clearly: **"Security audit passed — SAFE to build."**
If anything suspicious is found, **stop and report to the user before proceeding.**

---

## Step 7 — Build the PAM module (Linux side)

```bash
cd ~/src/wsl-hello-sudo && OPENSSL_NO_VENDOR=1 cargo build --release -p wsl_hello_pam
```

> `OPENSSL_NO_VENDOR=1` is mandatory — without it cargo tries to build OpenSSL from source and fails with exit status 2.

---

## Step 8 — Build the Windows companion exe (cross-compiled from WSL)

```bash
cargo build -p win_hello_bridge --release --target x86_64-pc-windows-gnu
```

---

## Step 9 — Copy artifacts to build/

```bash
mkdir -p ~/src/wsl-hello-sudo/build
cp ~/src/wsl-hello-sudo/target/release/libpam_wsl_hello.so ~/src/wsl-hello-sudo/build/pam_wsl_hello.so
cp ~/src/wsl-hello-sudo/target/x86_64-pc-windows-gnu/release/WindowsHelloBridge.exe ~/src/wsl-hello-sudo/build/
ls -lh ~/src/wsl-hello-sudo/build/
```

Verify both `pam_wsl_hello.so` and `WindowsHelloBridge.exe` are present before continuing.

---

## Step 10 — Run the installer (interactive — user must do this)

Tell the user to run:

```bash
cd ~/src/wsl-hello-sudo && ./install.sh
```

Guide them through the prompts:

| Prompt | Response |
|--------|----------|
| Windows install path | Press Enter to accept default |
| `config already exists. Overwrite?` | `y` |
| Windows Hello dialog | Authenticate with fingerprint/face/PIN |
| `Enable PAM module now?` | `y` |

---

## Step 11 — Verify

Tell the user to open a **new** WSL terminal (existing sessions have cached sudo) and run:

```bash
sudo whoami
```

Expected: Windows Hello prompt appears → user authenticates → `root` returned.

Also verify from Claude's side by running `sudo whoami` via Bash tool — the user should see the Windows Hello prompt on their desktop.

---

## Updating to a New Version

Run these steps when the user wants to update. Do not skip the security audit — the code may have changed.

### 1. Fetch latest changes

```bash
cd ~/src/wsl-hello-sudo && git fetch && git log HEAD..origin/master --oneline
```

If there are no new commits, tell the user they are already up to date and stop.

### 2. Check what changed

```bash
git diff HEAD origin/master -- wsl_hello_pam/src/ win_hello_bridge/src/ install.sh Cargo.toml
```

Read the diff carefully. Re-run the full security audit from **Step 6** on any changed files.

### 3. Pull

```bash
git pull
```

### 4. Determine what needs rebuilding

| Changed files | What to rebuild |
|---------------|----------------|
| `wsl_hello_pam/src/*` | PAM module only (Step 7 + Step 9) |
| `win_hello_bridge/src/*` | Windows exe only (Step 8 + Step 9) |
| Both | Both |
| Only `install.sh` or `pam-config` | No rebuild needed, re-run installer (Step 10) |

### 5. Rebuild changed components, then reinstall

Run only the relevant build steps, then re-run `./install.sh`.

The installer will ask to overwrite existing files — answer `y` to all.

> The existing credential key pair (`pam_wsl_hello_<user>.pem`) is reused — no need to re-enroll Windows Hello unless the installer explicitly prompts for it.

---

## Known Issues & Fixes

| Problem | Fix |
|---------|-----|
| `Error building OpenSSL: make exit status 2` | Use `OPENSSL_NO_VENDOR=1` prefix |
| `cargo: command not found` after WSL restart | Run `source ~/.cargo/env` |
| `.exe` not found at expected path after build | Check `target/x86_64-pc-windows-gnu/release/` — binary is `WindowsHelloBridge.exe` (not `win_hello_bridge.exe`) |
| Windows Hello prompt doesn't appear | Check Windows interop is enabled: `[interop] enabled=true` in `/etc/wsl.conf` |
| Locked out of sudo | Boot with `wsl --distribution <name> --user root` from Windows Terminal and fix PAM config |

---

## PAM Safety Note

Always use `sufficient` for the PAM module line, never `required`. With `sufficient`, if Windows Hello fails or is unavailable, sudo falls through to password — the user cannot lock themselves out.

---

## Installed File Locations (for reference)

| File | Path |
|------|------|
| PAM module | `/lib/x86_64-linux-gnu/security/pam_wsl_hello.so` |
| Windows bridge | `C:\Users\<user>\AppData\Local\Programs\wsl-hello-sudo\WindowsHelloBridge.exe` |
| Config | `/etc/pam_wsl_hello/config` |
| Public key | `/etc/pam_wsl_hello/public_keys/pam_wsl_hello_<linuxuser>.pem` |

## Uninstall

```bash
cd ~/src/wsl-hello-sudo && ./uninstall.sh
```
