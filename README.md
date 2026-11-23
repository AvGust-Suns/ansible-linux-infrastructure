# Ansible Linux Infrastructure Automation

Ansible-based automation project for configuring and managing a small Linux server infrastructure running Debian 12.

The project demonstrates infrastructure configuration through reusable Ansible roles, SSH key-based authentication, SSH hardening, firewall management, and service deployment.

## Project Overview

The infrastructure consists of one Ansible control node and three managed Debian servers.

### Infrastructure Architecture

| Host | Management IP | Internal IP | Purpose |
|---|---|---|---|
| ansible-control | 192.168.56.12 | 10.10.10.11 | Ansible Control Node |
| ansible-node1 | 192.168.56.13 | 10.10.10.12 | Web Server |
| ansible-node2 | 192.168.56.14 | 10.10.10.13 | Docker Host |
| ansible-node3 | 192.168.56.15 | 10.10.10.14 | Infrastructure Node |

The control node connects to managed servers through SSH using public key authentication.

### Technologies

- Debian 12
- Ansible Core 2.14
- OpenSSH
- Nginx
- Docker
- Docker Compose
- UFW Firewall
- YAML and Jinja2

## Project Structure

```text
ansible-linux-automation/
├── ansible.cfg
├── .gitignore
├── inventory/
│   └── hosts.yml
├── playbooks/
│   ├── bootstrap-user.yml
│   └── site.yml
└── roles/
    ├── common/
    │   └── tasks/
    │       └── main.yml
    ├── nginx/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   └── main.yml
    │   └── templates/
    │       └── index.html.j2
    ├── docker/
    │   └── tasks/
    │       └── main.yml
    ├── ssh_hardening/
    │   ├── handlers/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    └── firewall/
        └── tasks/
            └── main.yml
```

## Implemented Features

### 1. Base System Configuration

The `common` role automates:

- APT package cache updates.
- Safe package upgrades.
- Installation of common administration utilities.
- SSH service activation.

### 2. Ansible Management User

The `bootstrap-user.yml` playbook prepares managed servers for Ansible administration:

- Creates a dedicated `ansible` user.
- Configures SSH public key authentication.
- Grants sudo privileges.
- Configures passwordless sudo for automation.

### 3. Nginx Deployment

The `nginx` role:

- Installs Nginx.
- Deploys a Jinja2-based index page.
- Uses Ansible facts to display the server hostname.
- Ensures the service is enabled and running.
- Uses a handler to restart Nginx when its configuration content changes.

Assigned to `ansible-node1`.

### 4. Docker Deployment

The `docker` role:

- Installs Docker Engine from the Debian package repository.
- Installs Docker Compose.
- Ensures the Docker service is enabled and running.

Assigned to `ansible-node2`.

### 5. SSH Hardening

The `ssh_hardening` role configures:

- Root SSH login disabled.
- Password authentication disabled.
- Public key authentication enabled.
- Keyboard-interactive authentication disabled.
- SSH configuration validation before applying a restart.

### 6. Firewall Configuration

The `firewall` role automates UFW configuration:

- Default incoming policy: deny.
- Default outgoing policy: allow.
- SSH access allowed on all managed servers.
- HTTP access allowed only on the web server.
- Firewall enabled automatically.

The project uses the `community.general.ufw` module.

**Note:** Docker manages its own iptables rules. UFW alone does not guarantee filtering of published Docker container ports.

## Prerequisites

- Debian 12 control node.
- Three Debian 12 managed servers.
- Ansible Core installed on the control node.
- SSH connectivity between the control node and managed servers.
- Python 3 on managed servers.
- Initial administrative SSH access for bootstrapping.

Install Ansible and the required collection:

```bash
apt update
apt install ansible sshpass -y

ansible-galaxy collection install community.general
```

## Deployment

### 1. Configure the inventory

Update `inventory/hosts.yml` with the IP addresses and host groups of your servers.

### 2. Generate an SSH key

On the control node:

```bash
ssh-keygen -t ed25519 -f /root/.ssh/ansible_ed25519
```

### 3. Bootstrap the management user

Use initial administrative SSH access:

```bash
ansible-playbook playbooks/bootstrap-user.yml -u root -k
```

### 4. Verify connectivity

```bash
ansible managed_nodes -m ping
```

### 5. Check the playbook

```bash
ansible-playbook playbooks/site.yml --syntax-check
```

### 6. Apply the configuration

```bash
ansible-playbook playbooks/site.yml
```

### 7. Verify the result

```bash
ansible managed_nodes -m ping

ansible managed_nodes -b -m command -a "ufw status verbose"

ansible web_servers -m command -a "curl -I http://127.0.0.1"
```

## Verification Results

The current implementation has been tested on three Debian 12 virtual machines.

| Test | Result |
|---|---|
| Ansible connectivity | Passed on all managed nodes |
| Nginx deployment | Passed |
| HTTP response | HTTP 200 OK |
| Docker installation | Passed |
| SSH hardening | Passed |
| UFW activation | Passed on all managed nodes |
| SSH connectivity after firewall activation | Passed |
| Playbook syntax validation | Passed |
| Idempotency test | Passed |

A repeated execution of the complete playbook resulted in:

| Host | OK | Changed | Failed |
|---|---:|---:|---:|
| ansible-node1 | 17 | 0 | 0 |
| ansible-node2 | 15 | 0 | 0 |
| ansible-node3 | 12 | 0 | 0 |

The zero `changed` count confirms that the playbook does not make unnecessary configuration changes when the desired state is already achieved.

## Development Roadmap

Planned improvements:

- [ ] Prometheus monitoring integration.
- [ ] Node Exporter deployment across managed servers.
- [ ] Grafana dashboards and provisioning.
- [ ] Alertmanager integration.
- [ ] Additional infrastructure automation.
- [ ] Improved security and configuration validation.

## Notes

This project is developed and tested in a VirtualBox-based laboratory environment. Network addresses are examples from the lab configuration and should be adjusted when deploying in another environment.
