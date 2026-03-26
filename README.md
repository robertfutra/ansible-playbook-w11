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
| Ansible Controller | ansible-server | `<ANSIBLE_SERVER_IP>` | Linux | `<ANSIBLE_USER>` |
| Target Machine | windows-target | `<WINDOWS_TARGET_IP>` | Windows 11 | `<WINDOWS_USER>` |

---

## Prerequisites

### On the Ansible Server

```bash
# Install Python and Ansible
sudo apt update && sudo apt install -y python3 python3-pip
pip3 install ansible

# Install the Windows module collection
ansible-galaxy collection install ansible.windows
```

### On the Windows 11 Machine

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

> The Windows user (`<WINDOWS_USER>`) must be a member of the local **Administrators** group.

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
| SSH password | `group_vars/windows/vault.yml` | Password for the `<WINDOWS_USER>` user on the Windows target |
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

### Setting up the self-hosted runner

GitHub-hosted runners are on the public internet and cannot reach `<WINDOWS_TARGET_IP>`. The runner must live on a machine that has **LAN access to both GitHub (outbound HTTPS/443) and the Windows target (port 22)** — that machine is `ansible-server` (`<ANSIBLE_SERVER_IP>`).

**1. Install the runner on ansible-server (as `<ANSIBLE_USER>`)**

```bash
mkdir ~/actions-runner && cd ~/actions-runner

# Get the exact download URL + registration token from:
#   repo → Settings → Actions → Runners → New self-hosted runner → Linux x64
curl -o actions-runner-linux-x64-<version>.tar.gz -L <url-from-github>
tar xzf actions-runner-linux-x64-<version>.tar.gz
```

**2. Register the runner to the repo**

```bash
# The token is single-use and expires after 1 hour — generate it from the GitHub UI above
./config.sh --url https://github.com/<YOUR_ORG>/<YOUR_REPO> \
            --token <REGISTRATION_TOKEN>
# Accept all defaults (runner name, work folder, labels)
# This assigns the label "self-hosted", which matches runs-on: self-hosted in deploy.yml
```

**3. Start the runner**

```bash
./run.sh
# Keep this alive in a tmux or screen session.
# It must be running for GitHub Actions to pick up jobs.
# Restart it manually after ansible-server reboots.
```

**4. Install Ansible on the runner**

```bash
sudo apt update && sudo apt install -y python3 python3-pip
pip3 install ansible
ansible-galaxy collection install ansible.windows
```

**5. SSH access to the Windows target**

The runner executes playbooks as `<ANSIBLE_USER>`. Ansible connects to `<WINDOWS_TARGET_IP>` over SSH using the password stored in `group_vars/windows/vault.yml`. No SSH key setup is needed — `ansible.cfg` sets `host_key_checking = False`, so the first connection goes through without a known-hosts prompt.

**6. Add the GitHub secret**

In the repo: **Settings → Secrets and variables → Actions → New repository secret**

| Name | Value |
|------|-------|
| `VAULT_PASSWORD` | The ansible-vault password used to encrypt `vault.yml` and `wg0.conf` |

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
