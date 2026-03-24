# Ansible Playbook — WireGuard on Windows 11

Ansible playbook to automatically install and configure the [WireGuard](https://www.wireguard.com/) VPN client on a Windows 11 machine using an SSH connection.

---

## What this does

This playbook connects from a Linux-based Ansible server to a Windows 11 machine and performs the following steps automatically:

1. Downloads the official WireGuard installer from `wireguard.com`
2. Installs WireGuard silently (no user interaction needed)
3. Deploys a VPN tunnel configuration file (`.conf`)
4. Registers the tunnel as a Windows service
5. Starts the service and sets it to launch automatically on boot

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

## Configuration

Before running the playbook, open `roles/wireguard/vars/main.yml` and fill in the WireGuard VPN details:

```yaml
tunnel_name: "wg0"

# Your VPN client address
wireguard_address: "10.0.0.2/24"

# Keys — generate with: wg genkey | tee privatekey | wg pubkey > publickey
wireguard_private_key: "REPLACE_WITH_BASE64_PRIVATE_KEY"

# Optional DNS server
wireguard_dns: "1.1.1.1"

# VPN server details
wireguard_peer_public_key: "REPLACE_WITH_BASE64_PUBLIC_KEY"
wireguard_peer_endpoint: "SERVER_IP:51820"

# Routes — "0.0.0.0/0" = all traffic through VPN (full tunnel)
wireguard_peer_allowed_ips: "0.0.0.0/0, ::/0"
```

> **Security note:** The `wireguard_private_key` and `ansible_password` are stored in plain text for lab use. In a production environment, encrypt them using `ansible-vault`.

---

## Running the Playbook

```bash
# 1. Verify connectivity to the Windows machine
ansible -m win_ping windows

# 2. Run the playbook
ansible-playbook site.yml

# Optional: run in check mode (no changes applied)
ansible-playbook site.yml --check

# Optional: verbose output for troubleshooting
ansible-playbook site.yml -v
```

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
├── ansible.cfg                          # Ansible configuration (SSH, inventory path)
├── site.yml                             # Main playbook entry point
├── inventory/
│   └── hosts.yml                        # Target hosts and connection settings
├── group_vars/
│   └── windows.yml                      # Shared variables (installer URL, paths)
└── roles/
    └── wireguard/
        ├── tasks/main.yml               # Installation and configuration tasks
        ├── handlers/main.yml            # Service restart and post-install pause
        ├── vars/main.yml                # VPN tunnel variables (edit before running)
        └── templates/tunnel.conf.j2     # WireGuard config file template
```

---

## Generating WireGuard Keys

If you don't have keys yet, generate them on any Linux machine with WireGuard installed:

```bash
# Generate a private key
wg genkey > privatekey

# Derive the public key from it
wg pubkey < privatekey > publickey

cat privatekey   # paste into wireguard_private_key
cat publickey    # share with the VPN server administrator
```
