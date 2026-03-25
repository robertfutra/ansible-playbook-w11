# Ansible Playbook — WireGuard on Windows 11

Ansible playbook to automatically install and configure the [WireGuard](https://www.wireguard.com/) VPN client on a Windows 11 machine using an SSH connection.

---

## What this does

This playbook connects from a Linux-based Ansible server to a Windows 11 machine and performs the following steps automatically:

1. Downloads the official WireGuard installer from `wireguard.com`
2. Installs WireGuard silently (no user interaction needed)
3. Waits for the WireGuard manager service to be fully ready
4. Deploys a pre-generated VPN tunnel configuration file (`.conf`) stored encrypted in the repo
5. Validates the deployed config is a valid WireGuard file
6. Registers the tunnel as a Windows service
7. Starts the service and sets it to launch automatically on boot

The whole process is **idempotent** — running the playbook multiple times is safe and will only make changes when something is actually different.

---

## Infrastructure

| Role | Hostname | IP | OS | User |
|------|----------|----|----|------|
| Ansible Controller | ansible-server | 10.70.0.246 | Linux | fadmin |
| Target Machine | robert-w11 | 10.88.0.47 | Windows 11 | robert |

---

## Prerequisites

### On the Ansible Server (10.70.0.246)

```bash
# Install Python and Ansible
sudo apt update && sudo apt install -y python3 python3-pip
pip3 install ansible

# Install the Windows module collection
ansible-galaxy collection install ansible.windows
```

### On the Windows 11 Machine (10.88.0.47)

Run the following in PowerShell **as Administrator**:

```powershell
# 1. Install OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# 2. Start the SSH service and enable auto-start
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# 3. Set PowerShell as the default SSH shell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" `
  -Name DefaultShell `
  -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
  -PropertyType String -Force

# 4. Allow inbound SSH through Windows Firewall
#    Check if the rule already exists:
Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -ErrorAction SilentlyContinue

#    If it is missing, create it:
New-NetFirewallRule `
  -Name        "OpenSSH-Server-In-TCP" `
  -DisplayName "OpenSSH Server (sshd)" `
  -Description "Allow inbound SSH connections for Ansible management" `
  -Enabled     True `
  -Direction   Inbound `
  -Protocol    TCP `
  -Action      Allow `
  -LocalPort   22

#    Verify the rule is active:
Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP" | Select-Object Name, Enabled, Direction, Action

# 5. Restart SSH to apply all changes
Restart-Service sshd
```

> **Note:** The firewall rule is the most common reason SSH hangs when connecting from the Ansible server. Even if OpenSSH is installed and running, without this rule the connection will appear to hang indefinitely.

> The Windows user (`robert`) must be a member of the local **Administrators** group.

---

## Providing the WireGuard config file

The tunnel config is a pre-generated `.conf` file stored **encrypted** in the repo at `roles/wireguard/files/wg0.conf`.

To add or replace it:

```bash
# Encrypt your .conf file and place it where the role expects it
ansible-vault encrypt wg0.conf --output roles/wireguard/files/wg0.conf

# Commit the encrypted file
git add roles/wireguard/files/wg0.conf
git commit -m "Update WireGuard tunnel config"
```

The deploy task reads the file with `lookup('file', ...)` on the controller, which decrypts it automatically, then writes the content to the target via `win_copy content:`. The `tunnel_name` variable in `roles/wireguard/vars/main.yml` must match the filename (without `.conf`).

---

## Secrets handling

Secrets are stored encrypted using ansible-vault (AES256). The vault password is provided at runtime:

- **Local runs:** `--vault-password-file ~/.vault_pass` or prompted interactively
- **GitHub Actions:** `VAULT_PASSWORD` repository secret, written to a temp file and deleted after the run

| Secret | Location | Description |
|--------|----------|-------------|
| SSH password | `group_vars/windows/vault.yml` | Password for the `robert` user on the Windows target |
| WireGuard config | `roles/wireguard/files/wg0.conf` | Vault-encrypted tunnel `.conf` file |

```bash
# Edit or re-key the vault variables file
ansible-vault edit group_vars/windows/vault.yml
ansible-vault rekey group_vars/windows/vault.yml
```

---

## Running the Playbook

```bash
# 1. Verify connectivity to the Windows machine
ansible -m win_ping windows

# 2. Run the playbook
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Optional: run in check mode (no changes applied)
ansible-playbook site.yml --check --vault-password-file ~/.vault_pass

# Optional: verbose output for troubleshooting
ansible-playbook site.yml -v --vault-password-file ~/.vault_pass
```

---

## Automated Deployment (GitHub Actions)

The pipeline triggers automatically when a **pull request is merged into `main`**. It can also be triggered manually from the GitHub Actions UI.

**Workflow:**

1. Clones the repo to `~/ansible-playbook-w11` on the runner if it doesn't exist, or pulls the latest changes
2. Writes the vault password (from the `VAULT_PASSWORD` GitHub secret) to a temp file
3. Runs `ansible-playbook site.yml --vault-password-file .vault_pass`
4. Deletes the temp vault password file (always, even on failure)

**Setup required on the runner (ansible-server):**

- GitHub Actions self-hosted runner installed and registered to the repo
- `ansible` and the `ansible.windows` collection installed
- SSH access to the target Windows machine configured

**GitHub secret required:**

| Secret | Description |
|--------|-------------|
| `VAULT_PASSWORD` | Password used to decrypt vault files |

**Recommended: enable branch protection on `main`** (Settings → Branches) with "Require a pull request before merging" and at least 1 required approval.

---

## Verifying the Result

After the playbook finishes, check on the Windows machine:

```powershell
# The service should appear as "Running"
Get-Service "WireGuardTunnel*"

# View the WireGuard tunnel log
& "C:\Program Files\WireGuard\wireguard.exe" /dumplog
```

---

## Project Structure

```
ansible-playbook/
├── .github/
│   └── workflows/
│       └── deploy.yml                   # GitHub Actions CI/CD (triggers on merged PR)
├── ansible.cfg                          # Ansible configuration (SSH, inventory path)
├── site.yml                             # Main playbook entry point
├── inventory/
│   └── hosts.yml                        # Target hosts and connection settings
├── group_vars/
│   └── windows/
│       ├── vars.yml                     # Non-secret variables (installer URL, paths)
│       └── vault.yml                    # Ansible Vault encrypted secrets (SSH password)
└── roles/
    └── wireguard/
        ├── tasks/main.yml               # Installation and configuration tasks
        ├── handlers/main.yml            # Service restart and post-install pause
        ├── vars/main.yml                # Tunnel name
        └── files/wg0.conf              # Vault-encrypted WireGuard tunnel config
```
