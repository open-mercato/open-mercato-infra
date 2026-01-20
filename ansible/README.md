# Dokploy Infrastructure as Code

This repository contains Ansible playbooks and GitHub Actions workflows to deploy and manage [Dokploy](https://dokploy.com) on a Hetzner VPS.

## Overview

- **Automated deployment** via GitHub Actions on push to `main`
- **Idempotent playbooks** - safe to run multiple times
- **Security hardening** - separate playbook for post-domain configuration
- **Version controlled** - all infrastructure changes tracked in Git

## Prerequisites

### Server Requirements

- Hetzner VPS (or any Ubuntu server)
- Ubuntu 24.04 LTS
- SSH access configured with key-based authentication
- Minimum 1GB RAM, 20GB storage recommended

### Local Requirements (for manual runs)

- Python 3.11+ (not 3.14 - Ansible doesn't support it yet)
- Ansible 2.14+
- SSH private key for server access

```bash
# Using uv (recommended)
uv venv --python 3.11
source .venv/bin/activate
uv pip install ansible

# Or using pip
pip install ansible

# Install required collections
ansible-galaxy collection install -r requirements.yml
```

### SSH Key Setup for Local Development

If your SSH key has a passphrase, add it to the ssh-agent:

```bash
# Start ssh-agent (if not running)
eval "$(ssh-agent -s)"

# Add your key (enter passphrase once)
ssh-add ~/.ssh/id_ed25519

# Verify key is loaded
ssh-add -l
```

## GitHub Actions Setup

### Required Secrets

Configure these secrets in your GitHub repository settings (`Settings -> Secrets and variables -> Actions`):

| Secret | Description | Required |
|--------|-------------|----------|
| `SSH_PRIVATE_KEY` | Private SSH key content | Yes |
| `SERVER_IP` | Hetzner VPS public IP address | Yes |
| `SERVER_USER` | SSH user (typically `root`) | Yes |
| `SSH_KEY_PASSPHRASE` | Passphrase for SSH key (if protected) | No |

> **Note:** The workflows use `ssh-agent` to handle SSH keys. If your key has a passphrase, add the `SSH_KEY_PASSPHRASE` secret and the workflow will unlock it automatically.

### Available Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `deploy.yml` | Push to `main`, manual | Applies changes to the server |
| `preview.yml` | Pull requests, manual | Dry-run showing what would change |
| `lint.yml` | Push, PR | Validates Ansible syntax |
| `test.yml` | Push, PR | Runs Ansible tests |

### Creating a Dedicated SSH Key

It's recommended to use a dedicated SSH key for deployments:

```bash
# Generate a new key pair
ssh-keygen -t ed25519 -C "dokploy-deploy" -f ~/.ssh/dokploy_deploy

# Copy the public key to your server
ssh-copy-id -i ~/.ssh/dokploy_deploy.pub root@YOUR_SERVER_IP

# Add the private key content to GitHub Secrets as SSH_PRIVATE_KEY
cat ~/.ssh/dokploy_deploy
```

## Usage

### Automatic Deployment

Push changes to the `main` branch to trigger automatic deployment:

```bash
git add .
git commit -m "Update Dokploy configuration"
git push origin main
```

The workflow runs when:
- Pushing to `main` branch (changes in `ansible/` or workflow file)
- Manual trigger via GitHub Actions UI

### Manual Ansible Execution

For testing or debugging, run Ansible locally:

```bash
cd ansible

# Deploy Dokploy
ansible-playbook -i inventory/production.ini playbook.yml \
  -e "ansible_host=YOUR_SERVER_IP" \
  -e "ansible_user=root" \
  -e "ansible_ssh_private_key_file=~/.ssh/your_key"

# Security hardening (after domain is configured)
ansible-playbook -i inventory/production.ini harden.yml \
  -e "ansible_host=YOUR_SERVER_IP" \
  -e "ansible_user=root" \
  -e "ansible_ssh_private_key_file=~/.ssh/your_key"
```

### Using Workflow Dispatch

You can manually trigger the workflow from GitHub Actions:

1. Go to `Actions` tab in your repository
2. Select `Deploy Dokploy` workflow
3. Click `Run workflow`
4. Choose which playbook to run (`playbook.yml` or `harden.yml`)
5. Click `Run workflow`

## Post-Deployment Steps

### 1. Initial Setup

After the first deployment:

1. Open `http://YOUR_SERVER_IP:3000` in your browser
2. Create your admin account
3. Complete the initial setup wizard

### 2. Domain Configuration

Configure your domain in Dokploy:

1. Log in to Dokploy dashboard
2. Go to `Settings -> Web Server`
3. Enter your domain name
4. Save and wait for SSL certificate provisioning
5. Verify HTTPS access works: `https://your-domain.com`

### 3. Security Hardening

**Important:** Only run this AFTER your domain is working!

Via GitHub Actions:
1. Go to `Actions -> Deploy Dokploy`
2. Click `Run workflow`
3. Select `harden.yml`
4. Run the workflow

Via CLI:
```bash
ansible-playbook -i inventory/production.ini harden.yml \
  -e "ansible_host=YOUR_SERVER_IP" \
  -e "ansible_user=root"
```

This will:
- Install and configure fail2ban
- Remove port 3000 from the firewall
- Harden SSH configuration

After hardening, Dokploy is only accessible via your domain (not `IP:3000`).

## Project Structure

```
ansible/
├── inventory/
│   └── production.ini       # Server inventory
├── group_vars/
│   └── all.yml              # Shared variables
├── roles/
│   ├── dokploy/             # Main Dokploy installation role
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   └── defaults/
│   │       └── main.yml
│   └── security_hardening/  # Post-domain security role
│       ├── tasks/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── defaults/
│       │   └── main.yml
│       └── templates/
│           └── jail.local.j2
├── playbook.yml             # Main deployment playbook
├── harden.yml               # Security hardening playbook
└── README.md                # This file
```

## Configuration

### Customizing Variables

Edit `ansible/group_vars/all.yml` to customize:

```yaml
# System packages
system_packages:
  - curl
  - wget
  # ... add more as needed

# Firewall ports
ufw_allowed_ports_initial:
  - { port: 22, proto: tcp, comment: "SSH" }
  - { port: 80, proto: tcp, comment: "HTTP" }
  # ... customize ports

# Fail2ban settings
fail2ban_bantime: 3600    # Ban duration in seconds
fail2ban_maxretry: 5      # Failed attempts before ban
```

## Troubleshooting

### SSH Connection Failed

```bash
# Test SSH connection manually
ssh -i ~/.ssh/your_key root@YOUR_SERVER_IP

# Ensure the key is added to the server
ssh-copy-id -i ~/.ssh/your_key.pub root@YOUR_SERVER_IP
```

### Dokploy Not Starting

```bash
# Check Docker status on the server
ssh root@YOUR_SERVER_IP "docker ps -a"

# Check Dokploy logs
ssh root@YOUR_SERVER_IP "docker logs dokploy"
```

### Firewall Issues

```bash
# Check UFW status
ssh root@YOUR_SERVER_IP "ufw status verbose"

# Temporarily allow a port
ssh root@YOUR_SERVER_IP "ufw allow 3000/tcp"
```

### Re-running Installation

The playbook is idempotent. If Dokploy is already installed, it will skip the installation step. To force reinstall:

```bash
# On the server, remove Dokploy container
docker stop dokploy && docker rm dokploy

# Then re-run the playbook
```

## Security Notes

- SSH private keys are never committed to the repository
- Server IP is stored as a GitHub secret
- Use dedicated SSH keys for CI/CD (not personal keys)
- After domain setup, port 3000 should be closed
- fail2ban protects against SSH brute force attacks

## License

MIT
