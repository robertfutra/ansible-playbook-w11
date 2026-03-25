# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Ansible playbook to install and configure the WireGuard VPN client on a Windows 11 host over SSH.

- **Ansible controller**: Linux VM at `10.70.0.246` (user: `fadmin`)
- **Target host**: Windows 11 VM at `10.88.0.47` (user: `robert`)
- **Connection method**: SSH + PowerShell shell (not WinRM)
- **Required collection**: `ansible.windows` (`ansible-galaxy collection install ansible.windows`)

## Running the playbook

```bash
# Test connectivity
ansible -m win_ping windows

# Run full deployment
ansible-playbook site.yml

# Run with verbose output
ansible-playbook site.yml -v

# Dry run (check mode)
ansible-playbook site.yml --check
```

## Architecture

The project uses a single role (`roles/wireguard`) with this flow:

1. **`site.yml`** — entry point; asserts Windows target, applies `wireguard` role, reports tunnel service state
2. **`inventory/hosts.yml`** — defines the `windows` group with SSH connection vars
3. **`group_vars/windows/vars.yml`** — non-secret path variables (installer URL, exe path, config dir)
4. **`group_vars/windows/vault.yml`** — ansible-vault encrypted secrets (SSH password)
5. **`roles/wireguard/vars/main.yml`** — tunnel name (used as `.conf` filename and service suffix)
6. **`roles/wireguard/tasks/main.yml`** — 8 idempotent tasks: check install → download → install silently → flush handlers → ensure config dir → deploy config → register tunnel service → start service
7. **`roles/wireguard/files/wg0.conf`** — vault-encrypted WireGuard `.conf` file; auto-decrypted by Ansible on copy
8. **`roles/wireguard/handlers/main.yml`** — post-install pause (5s) and service restart on config change
9. **`.github/workflows/deploy.yml`** — GitHub Actions CI/CD; runs on self-hosted runner, decrypts vault via `VAULT_PASSWORD` secret, deploys on every push to `main`

## Providing the WireGuard config file

The tunnel config is a pre-generated `.conf` file stored encrypted in the repo at `roles/wireguard/files/wg0.conf`. To add or replace it:

```bash
# Encrypt your .conf file and place it where the role expects it
ansible-vault encrypt wg0.conf --output roles/wireguard/files/wg0.conf
```

Ansible auto-decrypts the file when copying it to the target, as long as the vault password is provided at runtime. The `tunnel_name` variable in `roles/wireguard/vars/main.yml` must match the filename (without `.conf`).

## Windows 11 prerequisites

The target must have OpenSSH Server configured with PowerShell as the default shell:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
Restart-Service sshd
```

## Secrets handling

Secrets are stored encrypted in `group_vars/windows/vault.yml` using ansible-vault (AES256). The vault password is provided at runtime via:

- **Local runs:** `--vault-password-file ~/.vault_pass` or prompted interactively
- **GitHub Actions:** `VAULT_PASSWORD` repository secret, written to a temp file and deleted after the playbook run

Vault variables used:
- `vault_robert-w11_password` — SSH password for the Windows target

The WireGuard config is stored as a vault-encrypted **file** (`roles/wireguard/files/wg0.conf`), not as individual variables.

To re-encrypt or edit the vault:

```bash
ansible-vault edit group_vars/windows/vault.yml
ansible-vault rekey group_vars/windows/vault.yml
```
