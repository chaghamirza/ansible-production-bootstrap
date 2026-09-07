# Ansible Production Bootstrap

A modular Ansible automation project for bootstrapping Ubuntu-based servers, applying baseline system configuration and security hardening, and provisioning either Docker with Docker Compose or a Kubernetes cluster using containerd.

The project is designed to provide a repeatable and structured server initialization workflow with separate Ansible roles for common system configuration, security hardening, Docker, and Kubernetes.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Requirements](#requirements)
- [Ansible Collections](#ansible-collections)
- [Inventory Configuration](#inventory-configuration)
- [Authentication and SSH Configuration](#authentication-and-ssh-configuration)
- [Configurable Variables](#configurable-variables)
- [Execution Modes](#execution-modes)
  - [Docker and Docker Compose](#docker-and-docker-compose)
  - [Kubernetes Cluster](#kubernetes-cluster)
- [Playbook Reference](#playbook-reference)
- [Role Reference](#role-reference)
- [Common Role](#common-role)
- [Hardening Role](#hardening-role)
- [Docker Role](#docker-role)
- [Kubernetes Role](#kubernetes-role)
- [Kubernetes Cluster Workflow](#kubernetes-cluster-workflow)
- [Network and Internet Requirements](#network-and-internet-requirements)
- [Security Considerations](#security-considerations)
- [Idempotency and Re-running Playbooks](#idempotency-and-re-running-playbooks)
- [Troubleshooting](#troubleshooting)
- [Customization](#customization)
- [Operational Notes](#operational-notes)
- [License](#license)

---

## Overview

This repository provides an Ansible-based server bootstrap framework for preparing Linux servers for containerized workloads.

The automation is divided into independent layers:

1. **Common system configuration**
2. **Security hardening**
3. **Docker and Docker Compose provisioning**
4. **Kubernetes cluster provisioning**

The project supports two primary deployment scenarios:

### Standard Docker Server

For servers intended to run Docker workloads, the complete bootstrap can be executed through:

```bash
ansible-playbook site.yml
```

This applies:

```text
Common Configuration
        |
        v
Security Hardening
        |
        v
Docker Engine + Docker Compose
```

### Kubernetes Cluster

For Kubernetes deployments, Docker must not be installed through this project.

The Kubernetes role installs and configures **containerd** as the container runtime and provisions the Kubernetes components required by `kubeadm`.

The recommended workflow is:

```text
Common Configuration
        |
        v
Security Hardening
        |
        v
containerd
        |
        v
kubelet + kubeadm + kubectl
        |
        v
Kubernetes Control Plane
        |
        v
Calico CNI
        |
        v
Worker Nodes
```

---

# Architecture

The repository follows the standard Ansible separation between inventories, playbooks, and reusable roles.

```text
ansible-production-bootstrap/
|
+-- ansible.cfg
|
+-- site.yml
|
+-- inventories/
|   +-- labs/
|   |   +-- hosts.ini
|   |   +-- group_vars/
|   |   |   +-- all/
|   |   |       +-- vars.yml
|   |   |       +-- vault.yml
|   |   +-- host_vars/
|   |
|   +-- company_A/
|   |   +-- hosts.ini
|   |   +-- group_vars/
|   |   |   +-- all/
|   |   |       +-- vars.yml
|   |   |       +-- vault.yml
|   |   +-- host_vars/
|   |
|   +-- company_B/
|
+-- playbooks/
|   +-- common.yml
|   +-- hardening.yml
|   +-- docker.yml
|   +-- kubernetes.yml
|
+-- roles/
    +-- common/
    +-- docker/
    +-- hardening/
    +-- kubernetes/
```

---

# Repository Structure

## `ansible.cfg`

The central Ansible configuration file.

It defines:

- Default inventory
- Role search path
- SSH host-key behavior
- Privilege escalation
- Output format
- Parallel execution count

Current configuration:

```ini
[defaults]
inventory = inventories/labs/hosts.ini
roles_path = roles
host_key_checking = false
retry_files_enabled = false
force_color = True
stdout_callback = yaml
forks = 10
```

Privilege escalation is configured to use `sudo` and execute tasks as `root`.

### Changing the Inventory

By default, Ansible uses:

```text
inventories/labs/hosts.ini
```

If another environment is required, either modify `ansible.cfg` or explicitly specify another inventory:

```bash
ansible-playbook -i inventories/company_A/hosts.ini site.yml
```

---

## `site.yml`

`site.yml` is the main orchestration playbook for standard Docker-based server provisioning.

It imports the following playbooks in order:

```text
common.yml
hardening.yml
docker.yml
```

Therefore:

```bash
ansible-playbook site.yml
```

is sufficient when the target system is intended to be a Docker server.

`site.yml` does **not** install Kubernetes.

---

# Requirements

## Control Node

The Ansible control machine requires:

- Linux or another supported Ansible control environment
- Ansible
- SSH client
- Python
- Required Ansible Collections

Ansible should be installed and available as:

```bash
ansible --version
```

Verify the installed version before using the project.

---

## Managed Nodes

The automation is primarily designed for Ubuntu-based systems using:

- APT
- systemd
- OpenSSH
- Python
- sudo

The Docker role explicitly uses the official Docker Ubuntu repository.

The Kubernetes role uses the Kubernetes APT repository for Kubernetes v1.33 packages.

The target servers should therefore be compatible with the Ubuntu-based package and service configuration implemented by these roles.

---

## Initial SSH Access

Before running the automation, Ansible must be able to connect to every target host using the credentials currently defined in the inventory.

For example:

```bash
ansible all -m ping
```

A successful response should be obtained before running any provisioning playbook.

The initial connection normally uses:

```ini
ansible_user=vagrant
ansible_port=22
```

These values must match the actual initial SSH configuration of the target servers.

---

# Ansible Collections

The project uses modules from the following Ansible Collections:

- `community.general`
- `ansible.posix`

Install them on the Ansible control node with:

```bash
ansible-galaxy collection install community.general ansible.posix
```

The project uses these collections for functionality including:

- Kernel module management
- Timezone configuration
- UFW configuration
- SSH authorized keys
- sysctl configuration

---

# Inventory Configuration

The inventory determines which servers are managed and how they are grouped.

The default inventory is:

```text
inventories/labs/hosts.ini
```

Example:

```ini
[master]
172.16.208.143

[worker]
172.16.208.144
172.16.208.145

[servers:children]
master
worker

[all:vars]
ansible_user=vagrant
ansible_port=22
# ansible_ssh_private_key_file=~/.ssh/id_rsa
```

## Kubernetes Groups

Kubernetes nodes must be divided into:

```ini
[master]
```

and:

```ini
[worker]
```

The first host in the `master` group is used by the Kubernetes role as the control-plane node.

Workers are automatically joined to the cluster using the join command generated by the master.

---

# Authentication and SSH Configuration

There are two different SSH stages in the deployment.

## Initial Connection

Before hardening, Ansible uses the values defined in the inventory.

For example:

```ini
[all:vars]
ansible_user=vagrant
ansible_port=22
```

## Post-Hardening Connection

The hardening role changes SSH to the configured port and user.

The default values are:

```yaml
system_user: "vagrant"
ssh_port: 2222
```

It also disables:

```text
Root SSH login
Password authentication
```

and configures SSH key authentication.

---

## Changing the Administrative User

To change the system user created by the hardening role, edit:

```text
roles/hardening/defaults/main.yml
```

For example:

```yaml
system_user: "admin"
```

The user will be:

- Created if it does not exist
- Added to the `sudo` group
- Configured for passwordless sudo
- Given the configured SSH public key

---

## Changing the SSH Port

The default SSH port is:

```yaml
ssh_port: 2222
```

Change it in:

```text
roles/hardening/defaults/main.yml
```

For example:

```yaml
ssh_port: 22022
```

The hardening role performs the transition in the following order:

1. Allows the new SSH port through UFW.
2. Changes `sshd_config`.
3. Restarts SSH.
4. Updates the active Ansible connection parameters.
5. Verifies connectivity on the new port.
6. Denies TCP port 22.
7. Continues with Fail2Ban configuration.

This ordering is intentional to reduce the risk of locking out the automation during the SSH migration.

---

## Changing the SSH Public Key

The hardening role loads the public key from:

```text
roles/hardening/files/id_rsa.pub
```

Replace its contents with the public SSH key that should be authorized for the configured system user.

Example:

```text
ssh-ed25519 AAAA... user@example.com
```

Do not place private SSH keys inside this repository.

---

## Important: Password Authentication

This project does not rely on SSH passwords after hardening.

The hardening role explicitly configures:

```text
PasswordAuthentication no
```

Therefore, authentication should be performed using the configured SSH public/private key pair.

The `system_user` receives passwordless sudo through:

```text
/etc/sudoers.d/<system_user>
```

This is separate from SSH authentication.

---

# Configurable Variables

The most important configurable values are located in the `defaults/main.yml` files of each role.

---

## Common Variables

File:

```text
roles/common/defaults/main.yml
```

### Timezone

```yaml
system_timezone: "Asia/Tehran"
```

### System Packages

The default administration packages include:

```text
curl
wget
git
vim
htop
unzip
tar
jq
apt-transport-https
ca-certificates
gnupg
lsb-release
chrony
tree
```

### Network Utilities

The role installs:

```text
net-tools
dnsutils
iputils-ping
traceroute
mtr-tiny
tcpdump
netcat-openbsd
socat
bridge-utils
iperf3
```

---

## Hardening Variables

File:

```text
roles/hardening/defaults/main.yml
```

Main variables:

```yaml
ssh_port: 2222
system_user: "vagrant"
ssh_permit_root_login: "prohibit-password"
ssh_password_authentication: "no"
ssh_max_auth_tries: "3"

ufw_default_incoming_policy: deny
ufw_default_outgoing_policy: allow
```

The actual SSH hardening tasks enforce:

```text
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries <configured-value>
```

---

## Docker Variables

File:

```text
roles/docker/defaults/main.yml
```

### Docker Users

```yaml
docker_users:
  - "vagrant"
```

Users listed here are added to the `docker` group.

### Docker Logging

```yaml
docker_log_max_size: "100m"
docker_log_max_file: "3"
```

These values are applied through:

```text
roles/docker/templates/daemon.json.j2
```

### Cgroup Driver

```yaml
docker_cgroup_driver: "systemd"
```

The Docker daemon is configured to use the systemd cgroup driver.

---

## Kubernetes Variables

File:

```text
roles/kubernetes/defaults/main.yml
```

### Kubernetes Version

```yaml
k8s_version: "1.33.2-1.1"
```

The specified versions of:

```text
kubelet
kubeadm
kubectl
```

are installed and subsequently held to prevent unintended package upgrades.

### Pod Network CIDR

```yaml
pod_network_cidr: "192.168.0.0/16"
```

This value is passed to:

```bash
kubeadm init
```

### Container Runtime Socket

```yaml
cri_socket: "unix:///run/containerd/containerd.sock"
```

Kubernetes is configured to use containerd as its CRI endpoint.

---

# Execution Modes

# Docker and Docker Compose

Use this mode when the target server is intended to run Docker workloads.

## Step 1: Configure Inventory

Edit:

```text
inventories/labs/hosts.ini
```

Add the target servers to the appropriate groups.

For a normal Docker-only environment, the servers can be placed in the `servers` group.

Example:

```ini
[servers]
192.168.1.101
192.168.1.102

[all:vars]
ansible_user=vagrant
ansible_port=22
```

## Step 2: Verify Connectivity

```bash
ansible all -m ping
```

## Step 3: Run the Complete Bootstrap

```bash
ansible-playbook site.yml
```

The master playbook automatically executes:

```text
common.yml
hardening.yml
docker.yml
```

in that order.

No additional Docker playbook execution is required.

---

# Kubernetes Cluster

When provisioning Kubernetes, **do not run `site.yml`** and do not run:

```bash
ansible-playbook playbooks/docker.yml
```

The Kubernetes role configures **containerd** directly and does not require Docker Engine.

Installing Docker unnecessarily on Kubernetes nodes is therefore intentionally avoided.

---

## Step 1: Configure the Master and Workers

Edit:

```text
inventories/labs/hosts.ini
```

Example:

```ini
[master]
192.168.1.101

[worker]
192.168.1.102
192.168.1.103

[servers:children]
master
worker

[all:vars]
ansible_user=vagrant
ansible_port=22
```

The inventory should contain exactly the intended Kubernetes topology.

---

## Step 2: Run Common Configuration

```bash
ansible-playbook playbooks/common.yml
```

This prepares the operating system and applies the common baseline configuration.

---

## Step 3: Run Security Hardening

```bash
ansible-playbook playbooks/hardening.yml
```

This secures SSH, configures UFW, creates the administrative user, installs the SSH key, and enables Fail2Ban.

### Important SSH Inventory Update

The hardening role changes the SSH port from the initial inventory value to the configured `ssh_port`.

For example:

```text
22 -> 2222
```

If the Kubernetes playbook is executed as a **separate Ansible command**, the inventory must reflect the new SSH connection parameters.

For example:

```ini
[all:vars]
ansible_user=vagrant
ansible_port=2222
```

If `system_user` was changed, `ansible_user` must also be changed accordingly.

This is important because `set_fact` changes the active connection parameters during the hardening play, but those runtime facts should not be relied upon across separate `ansible-playbook` process executions.

---

## Step 4: Run Kubernetes Provisioning

After the common and hardening stages are complete:

```bash
ansible-playbook playbooks/kubernetes.yml
```

The Kubernetes playbook performs the remaining cluster provisioning.

---

# Playbook Reference

## `playbooks/common.yml`

Purpose:

```text
Base operating-system preparation
```

The playbook executes the `common` role against:

```text
servers
```

It performs:

- System timezone configuration
- Base administration package installation
- Network troubleshooting package installation
- Chrony installation
- Chrony service enablement
- Current swap deactivation
- Persistent swap deactivation
- Custom MOTD deployment
- Disabling Ubuntu dynamic MOTD components

---

## `playbooks/hardening.yml`

Purpose:

```text
Server security hardening
```

The playbook executes the `hardening` role against:

```text
servers
```

It performs:

- Administrative user creation
- Sudo group membership
- Passwordless sudo configuration
- SSH authorized-key deployment
- UFW installation
- UFW configuration
- New SSH port authorization
- SSH port migration
- Root SSH login restriction
- Password authentication disabling
- X11 forwarding disabling
- Maximum SSH authentication attempt configuration
- Runtime Ansible connection migration
- Old SSH port denial
- Fail2Ban installation
- Fail2Ban service enablement

---

## `playbooks/docker.yml`

Purpose:

```text
Docker Engine and Docker Compose provisioning
```

The playbook executes the `docker` role against:

```text
servers
```

It performs:

- Docker kernel module configuration
- Container networking sysctl configuration
- Docker official GPG key installation
- Official Docker APT repository configuration
- Docker Engine installation
- Docker CLI installation
- containerd installation as part of the Docker package stack
- Docker Buildx plugin installation
- Docker Compose plugin installation
- Docker daemon configuration
- Docker service enablement
- Docker service startup
- Non-root Docker user configuration

---

## `playbooks/kubernetes.yml`

Purpose:

```text
Kubernetes cluster provisioning
```

Unlike the other service-specific playbooks, this playbook targets:

```text
all
```

and uses the Kubernetes role to determine whether each node is a master or worker based on its inventory group membership.

It performs:

- Kernel module configuration
- Kubernetes networking sysctl configuration
- containerd installation
- containerd configuration
- systemd cgroup configuration
- Kubernetes package repository configuration
- kubelet installation
- kubeadm installation
- kubectl installation
- Kubernetes package version pinning
- Control-plane initialization
- kubeconfig creation
- Calico CNI deployment
- Worker join command generation
- Worker node cluster joining

---

# Role Reference

The repository contains four independent roles.

```text
roles/
├── common
├── hardening
├── docker
└── kubernetes
```

Each role encapsulates a specific infrastructure responsibility.

---

# Common Role

Location:

```text
roles/common/
```

## Responsibilities

The common role establishes the operating-system baseline required by the rest of the automation.

### Timezone

The system timezone is configured using:

```yaml
system_timezone
```

### Base Packages

Administration tools such as:

```text
curl
wget
git
vim
htop
jq
tree
```

are installed.

### Network Diagnostics

Network troubleshooting tools include:

```text
ping
traceroute
mtr
tcpdump
netcat
socat
iperf3
dig
nslookup
```

### NTP

Chrony is installed, enabled, and started.

### Swap

Swap is disabled immediately and its configuration is modified to prevent it from being enabled after reboot.

This is particularly important for Kubernetes nodes.

### MOTD

The default MOTD is replaced with a custom template containing:

- Hostname
- Operating system
- Kernel
- Architecture
- Primary IP
- Primary interface
- MAC address
- CPU count
- RAM
- Deployment date

---

# Hardening Role

Location:

```text
roles/hardening/
```

The hardening role establishes the baseline security configuration for managed servers.

## Administrative User

The configured `system_user` is:

- Created
- Assigned `/bin/bash`
- Added to the `sudo` group
- Configured for passwordless sudo

## SSH Key Authentication

The public key stored at:

```text
roles/hardening/files/id_rsa.pub
```

is installed for the administrative user.

## SSH Hardening

The role configures:

```text
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
```

The SSH port is configurable.

## Firewall

UFW is configured with:

```text
Incoming: deny
Outgoing: allow
```

The new SSH port is allowed before SSH is restarted.

Port 22 is subsequently denied.

## Fail2Ban

Fail2Ban is installed and configured as an enabled system service.

---

# Docker Role

Location:

```text
roles/docker/
```

The Docker role installs Docker from the official Docker APT repository.

## Installed Components

The role installs:

```text
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
```

Docker Compose is therefore available through the Docker CLI:

```bash
docker compose
```

## Kernel Configuration

The following modules are loaded:

```text
overlay
br_netfilter
```

The following networking parameters are enabled:

```text
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
```

## Docker Daemon Configuration

The generated:

```text
/etc/docker/daemon.json
```

configures:

- systemd cgroup driver
- JSON-file logging
- Maximum log size
- Maximum number of retained log files

Example effective configuration:

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

## Docker Group

Users defined through:

```yaml
docker_users
```

are added to the `docker` group, allowing Docker CLI access without explicitly using `sudo`.

---

# Kubernetes Role

Location:

```text
roles/kubernetes/
```

The Kubernetes role uses:

```text
containerd
```

as the container runtime.

It does not require Docker Engine.

---

## Containerd Configuration

The role:

- Installs containerd
- Loads required kernel modules
- Configures Kubernetes networking sysctl parameters
- Generates the default containerd configuration
- Enables systemd cgroups
- Configures the Kubernetes pause image
- Enables and starts containerd

The configured CRI socket is:

```text
unix:///run/containerd/containerd.sock
```

---

## Kubernetes Packages

The role configures the Kubernetes v1.33 APT repository and installs:

```text
kubelet
kubeadm
kubectl
```

All three packages use the configured:

```yaml
k8s_version
```

and are subsequently held to prevent unintended upgrades.

---

# Kubernetes Cluster Workflow

The Kubernetes role determines node behavior from inventory group membership.

## Master Nodes

A host in:

```ini
[master]
```

executes the master tasks.

The role:

1. Checks whether the cluster has already been initialized.
2. Runs `kubeadm init` when required.
3. Creates the user's `.kube` directory.
4. Copies `/etc/kubernetes/admin.conf`.
5. Installs Calico.
6. Generates a worker join command.
7. Stores the join command as an Ansible fact.

The initialization uses:

```bash
kubeadm init \
  --pod-network-cidr=<configured CIDR> \
  --cri-socket=<configured CRI socket>
```

---

## Worker Nodes

Hosts in:

```ini
[worker]
```

execute the worker tasks.

Each worker:

1. Checks whether it has already joined the cluster.
2. Retrieves the generated join command from the first master.
3. Executes the join operation.
4. Explicitly specifies the containerd CRI socket.

This makes the worker provisioning repeatable and avoids rejoining an already configured worker.

---

# Network and Internet Requirements

Kubernetes provisioning requires external network connectivity.

The automation downloads resources from external repositories and endpoints, including:

```text
Docker official APT repository
Kubernetes APT repository
GitHub
registry.k8s.io
Calico manifests
```

In environments where these endpoints are inaccessible because of regional network restrictions or sanctions, the required network bypass mechanism must be enabled **before running the Kubernetes playbook**.

This may be a properly configured:

- VPN
- Proxy
- Network tunneling mechanism
- Other authorized connectivity solution

The connectivity mechanism must be functional from the target servers, not merely from the Ansible control node, because package downloads, container image pulls, and Kubernetes resource retrieval may occur directly from the managed nodes.

Before provisioning Kubernetes, verify that the target nodes can reach the required external endpoints.

---

# Security Considerations

## SSH Host Key Checking

The current `ansible.cfg` contains:

```ini
host_key_checking = false
```

This makes initial automation easier but reduces SSH host authenticity protection.

For higher-security environments, consider enabling host-key verification and maintaining trusted SSH host keys through an appropriate operational process.

---

## SSH Password Authentication

Password authentication is disabled by the hardening role.

Ensure that the configured SSH public key is valid and that the corresponding private key is available before applying the hardening role.

Otherwise, SSH access may be lost after the hardening operation.

---

## SSH Port Change

The firewall rule for the new SSH port is created before the SSH daemon is restarted.

This ordering is deliberate.

Do not manually remove the old SSH access path before confirming connectivity through the new port.

---

## Passwordless Sudo

The configured administrative user receives:

```text
NOPASSWD:ALL
```

sudo access.

This is convenient for automation but provides full root-level access to that account.

For production environments with stricter security requirements, review this behavior against the organization's privilege-management policy.

---

## Docker Group Privileges

Membership in the `docker` group effectively provides root-equivalent control over the host because Docker can create privileged containers and mount host resources.

Only trusted users should be added to:

```yaml
docker_users
```

---

## SSH Private Keys

Private SSH keys must never be committed to this repository.

Only public keys should be stored in:

```text
roles/hardening/files/
```

---

# Idempotency and Re-running Playbooks

The roles contain checks intended to prevent unnecessary repeated operations.

Examples include:

- Kubernetes initialization checks
- Worker join checks
- Existing containerd configuration checks
- Existing Kubernetes package keyring checks

For example, the master node checks:

```text
/etc/kubernetes/admin.conf
```

before running `kubeadm init`.

Worker nodes check:

```text
/etc/kubernetes/kubelet.conf
```

before attempting to join the cluster.

This allows playbooks to be safely re-run in normal operational circumstances.

However, changing fundamental configuration such as:

- SSH port
- Administrative username
- Kubernetes topology
- Kubernetes version
- Pod CIDR

should be treated as an infrastructure change rather than simply assuming that re-running the playbook will automatically migrate every dependent component.

---

# Recommended Deployment Procedures

## Docker Server

Use:

```bash
ansible all -m ping
ansible-playbook site.yml
```

The resulting system contains:

```text
Common tools
+
Security hardening
+
Docker Engine
+
Docker Compose
```

---

## Kubernetes Cluster

Use the following sequence:

```bash
ansible all -m ping

ansible-playbook playbooks/common.yml

ansible-playbook playbooks/hardening.yml
```

After hardening, make sure the inventory contains the new SSH connection parameters.

For the default configuration:

```ini
[all:vars]
ansible_user=vagrant
ansible_port=2222
```

Then run:

```bash
ansible-playbook playbooks/kubernetes.yml
```

Do **not** run:

```bash
ansible-playbook site.yml
```

or:

```bash
ansible-playbook playbooks/docker.yml
```

for Kubernetes nodes.

---

# Environment Separation

The repository supports multiple inventories.

Example:

```text
inventories/
├── labs/
├── company_A/
└── company_B/
```

Each environment can maintain its own:

```text
hosts.ini
group_vars/
host_vars/
```

This allows the same roles and playbooks to be reused across different environments.

For example:

```bash
ansible-playbook \
  -i inventories/company_A/hosts.ini \
  playbooks/common.yml
```

---

# Variables and Inventory Precedence

Role defaults are located under:

```text
roles/<role>/defaults/main.yml
```

These provide baseline values.

Environment-specific configuration should preferably be maintained under:

```text
inventories/<environment>/group_vars/
```

and host-specific configuration under:

```text
inventories/<environment>/host_vars/
```

The repository already provides:

```text
vars.yml
vault.yml
```

locations for environment-specific variables and sensitive data.

For sensitive values, Ansible Vault should be preferred over storing plaintext secrets in Git.

Example:

```bash
ansible-vault encrypt inventories/company_A/group_vars/all/vault.yml
```

---

# File and Directory Reference

## `inventories/`

Environment-specific inventory configuration.

### `hosts.ini`

Defines:

- Managed hosts
- Master nodes
- Worker nodes
- Initial SSH connection parameters

### `group_vars/`

Variables shared by groups of hosts.

### `host_vars/`

Variables specific to individual hosts.

### `vault.yml`

Reserved for encrypted sensitive variables.

---

## `playbooks/`

Execution entry points.

```text
common.yml
hardening.yml
docker.yml
kubernetes.yml
```

Each playbook invokes one corresponding role.

---

## `roles/common/`

Operating-system baseline configuration.

Contains:

```text
defaults/
tasks/
handlers/
templates/
files/
```

---

## `roles/hardening/`

Security and SSH hardening.

Contains:

```text
defaults/
tasks/
handlers/
templates/
files/
```

The SSH public key is located at:

```text
roles/hardening/files/id_rsa.pub
```

---

## `roles/docker/`

Docker Engine and Docker Compose provisioning.

The Docker daemon configuration template is:

```text
roles/docker/templates/daemon.json.j2
```

---

## `roles/kubernetes/`

Kubernetes cluster provisioning.

The role separates Kubernetes provisioning into:

```text
containerd.yml
kube_tools.yml
master.yml
worker.yml
```

This separation keeps container-runtime preparation, Kubernetes package installation, control-plane initialization, and worker joining logically independent.

---

# Troubleshooting

## Ansible Cannot Connect to a Host

Test connectivity:

```bash
ansible all -m ping
```

Verify:

```text
IP address
SSH port
SSH username
SSH private key
Sudo access
```

---

## SSH Connection Fails After Hardening

Verify the configured port:

```yaml
ssh_port: 2222
```

and ensure the inventory reflects the new port for subsequent Ansible executions:

```ini
ansible_port=2222
```

Also verify that the corresponding private key matches the public key deployed by the hardening role.

---

## Docker Installation Fails

Verify that the target system can access:

```text
download.docker.com
```

and that the system is compatible with the Ubuntu APT repository used by the role.

---

## Kubernetes Package Installation Fails

Verify connectivity to:

```text
pkgs.k8s.io
```

and confirm that the configured Kubernetes version exists in the configured repository.

---

## Kubernetes Initialization Fails

Check:

```text
containerd
swap
kernel modules
sysctl parameters
CRI socket
external network connectivity
```

The configured CRI endpoint is:

```text
/run/containerd/containerd.sock
```

---

## Worker Nodes Fail to Join

Verify:

1. The master node was initialized successfully.
2. The master generated a valid join command.
3. The worker can reach the master.
4. containerd is running on the worker.
5. The worker has the correct CRI socket.
6. The worker is present in the `[worker]` inventory group.
7. The first master is present in the `[master]` group.

---

# Customization

The project is intentionally structured so that most environment-specific changes can be made without modifying the role tasks themselves.

Recommended customization points are:

```text
inventories/<environment>/hosts.ini
inventories/<environment>/group_vars/
inventories/<environment>/host_vars/
roles/<role>/defaults/main.yml
```

Typical changes include:

- Target hosts
- SSH username
- SSH port
- SSH public key
- System timezone
- Package lists
- Docker users
- Docker log limits
- Kubernetes version
- Kubernetes Pod CIDR
- Container runtime socket

---

# Operational Notes

## Docker and Kubernetes Are Separate Deployment Paths

The project intentionally treats Docker and Kubernetes as separate provisioning targets.

### Docker Path

```text
common
  |
hardening
  |
docker
```

### Kubernetes Path

```text
common
  |
hardening
  |
kubernetes
```

The Kubernetes path does not use the Docker role.

---

## Kubernetes Runtime

Kubernetes is configured to use:

```text
containerd
```

through:

```text
/run/containerd/containerd.sock
```

Docker Engine is not required for the Kubernetes deployment.

---

## Kubernetes Networking

The cluster is configured with:

```yaml
pod_network_cidr: "192.168.0.0/16"
```

and Calico is used as the CNI.

The Calico manifest is applied from the upstream Calico project.

---

## Kubernetes Version Pinning

The project explicitly installs the configured Kubernetes package version and places:

```text
kubelet
kubeadm
kubectl
```

on package hold.

This prevents unattended package upgrades from unexpectedly changing the Kubernetes component versions.

Kubernetes version upgrades should therefore be treated as deliberate infrastructure operations.

---

# Quick Reference

## Test Ansible Connectivity

```bash
ansible all -m ping
```

## Docker Server

```bash
ansible-playbook site.yml
```

## Individual Common Setup

```bash
ansible-playbook playbooks/common.yml
```

## Individual Hardening

```bash
ansible-playbook playbooks/hardening.yml
```

## Docker Only

```bash
ansible-playbook playbooks/docker.yml
```

## Kubernetes

```bash
ansible-playbook playbooks/common.yml
ansible-playbook playbooks/hardening.yml
ansible-playbook playbooks/kubernetes.yml
```

For separate executions after hardening, ensure that `ansible_user` and `ansible_port` in the inventory match the hardened SSH configuration.

---

# Deployment Decision Matrix

| Requirement | `site.yml` | `docker.yml` | `kubernetes.yml` |
|---|---:|---:|---:|
| Common system setup | Yes | No | No |
| Security hardening | Yes | No | No |
| Docker Engine | Yes | Yes | No |
| Docker Compose | Yes | Yes | No |
| containerd for Kubernetes | No | No | Yes |
| kubelet | No | No | Yes |
| kubeadm | No | No | Yes |
| kubectl | No | No | Yes |
| Kubernetes control plane | No | No | Yes |
| Kubernetes workers | No | No | Yes |
| Calico CNI | No | No | Yes |

---

# Design Principles

The project follows several operational principles:

1. **Separation of concerns**  
   System configuration, hardening, Docker, and Kubernetes are implemented as separate roles.

2. **Reusable roles**  
   Roles can be invoked independently through their corresponding playbooks.

3. **Environment-specific inventories**  
   Different environments can use the same automation with independent inventories and variables.

4. **Repeatable provisioning**  
   Existing Kubernetes initialization and worker membership are detected before destructive or duplicate operations are performed.

5. **Explicit runtime configuration**  
   Docker and Kubernetes use explicitly configured container runtime and cgroup settings.

6. **Security-first bootstrap**  
   SSH hardening, firewall configuration, key-based authentication, and Fail2Ban are applied before service provisioning in the standard bootstrap workflow.

7. **Controlled Kubernetes versions**  
   Kubernetes packages are explicitly versioned and held to prevent unplanned upgrades.

---

# License

This project is distributed under the terms of the license included in:

```text
LICENSE
```

Review the license file for the complete terms and conditions.
