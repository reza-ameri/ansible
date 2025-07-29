# Ansible Docker and Server Hardening for Restricted Environments

This Ansible project automates the deployment of **Docker** and **Docker Compose** on Ubuntu servers and applies **security hardening** using `iptables`, `fail2ban`, and SSH configurations. It is tailored for environments with internet restrictions, such as in **Iran**, where sanctions require a proxy to access external resources. The project supports multiple servers and allows targeting a specific server using the `--limit` flag, making it ideal for both single-server and multi-server setups.

## Features
- **Docker Installation**:
  - Installs Docker and Docker Compose with proxy support for package downloads.
  - Adds the specified user to the `docker` group for non-root access.
- **Server Hardening**:
  - Configures `iptables` firewall with a `DROP` policy for incoming traffic, allowing specific ports (22 for SSH, 80 for HTTP, 443 for HTTPS by default).
  - Disables root login via SSH for enhanced security.
  - Installs and enables `fail2ban` to protect against brute-force attacks.
  - Disables unnecessary services (`apport`, `whoopsie`).
- **Proxy Support**:
  - Uses HTTP or SOCKS5 proxy to bypass internet restrictions in Iran for downloading packages.
- **Multi-Server Management**:
  - Defines servers in `inventory/hosts.yml` and supports execution on a single server with `--limit`.
- **Centralized Configuration**:
  - Manages proxy, SSH, and firewall settings in `inventory/group_vars/all.yml` for easy maintenance.

## Prerequisites
- **Control Node** (where Ansible is executed):
  - Ubuntu or any Linux distribution with **Ansible** installed (version 2.9+ recommended).
  - SSH key pair generated (`~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`).
  - Python 3 and `pip` for installing Ansible dependencies.
- **Target Servers**:
  - Ubuntu 22.04 LTS (other distributions may require adjustments to roles).
  - At least 1GB RAM (2GB swap recommended for low-memory servers).
  - SSH access enabled with the control node’s public key in `~/.ssh/authorized_keys`.
- **Proxy**:
  - A reliable HTTP or SOCKS5 proxy (e.g., `proxy.example.com:1080`) to bypass sanctions.
  - Proxy credentials (username and password, if required).
- **Network**:
  - Ports 22 (SSH), 80 (HTTP), and 443 (HTTPS) open on target servers for basic operation.
- **Git**:
  - Required to clone this repository.

## Project Structure
```bash
ansible-docker-hardening/
├── inventory/
│   ├── hosts.yml              # Defines target servers
│   └── group_vars/
│       └── all.yml            # Centralized variables (proxy, SSH, firewall ports)
├── playbook.yml               # Main playbook to execute Docker and hardening roles
├── roles/
│   ├── docker/
│   │   └── tasks/
│   │       └── main.yml       # Tasks for installing Docker and Docker Compose
│   ├── hardening/
│   │   ├── tasks/
│   │   │   └── main.yml       # Tasks for server hardening (iptables, fail2ban, SSH)
│   │   └── handlers/
│   │       └── main.yml       # Handlers for restarting services (e.g., SSH)
└── README.md                  # This documentation
```

## Setup Instructions

### 1. Clone the Repository
Clone the repository to your control node:
```bash
git clone https://github.com/your-username/ansible-docker-hardening.git
cd ansible-docker-hardening
```

### 2. Install Ansible and Dependencies
Install Ansible and required Python packages on the control node:
```bash
sudo apt update
sudo apt install -y python3-pip
pip3 install ansible
ansible-galaxy collection install ansible.posix
```

### 3. Configure SSH Keys
Ensure an SSH key pair exists on the control node:
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
- Press Enter to accept the default path (`~/.ssh/id_rsa`).

Copy the public key to each target server (replace `192.168.1.101` with your server’s IP):
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@192.168.1.101
```
If using a proxy, configure SSH to route through it:
```bash
nano ~/.ssh/config
```
Add the following, replacing `proxy.example.com:1080` with your proxy details:
```conf
Host 192.168.1.101 192.168.1.102 192.168.1.103
    User ubuntu
    ProxyCommand nc -X 5 -x proxy.example.com:1080 %h %p
```
Set appropriate permissions:
```bash
chmod 600 ~/.ssh/config
```

### 4. Configure Inventory
Edit `inventory/hosts.yml` to define your target servers:
```yaml
all:
  hosts:
    server1:
      ansible_host: 192.168.1.101
    server2:
      ansible_host: 192.168.1.102
    server3:
      ansible_host: 192.168.1.103
```
- Replace `192.168.1.101`, `192.168.1.102`, and `192.168.1.103` with your actual server IPs.

Edit `inventory/group_vars/all.yml` to configure proxy, SSH, and firewall settings:
```yaml
---
# SSH settings
ansible_ssh_user: ubuntu
ansible_ssh_private_key_file: /home/your_control_node_user/.ssh/id_rsa

# Proxy settings
proxy_host: proxy.example.com
proxy_port: 1080
proxy_user: your_proxy_user
proxy_pass: your_proxy_password
proxy_url: "http://{{ proxy_user }}:{{ proxy_pass }}@{{ proxy_host }}:{{ proxy_port }}"

# Firewall ports (for iptables)
allowed_ports:
  - 22
  - 80
  - 443

# Docker settings
docker_compose_version: v2.24.5
```
- Replace `your_control_node_user` with the username on your control node (e.g., `ubuntu`).
- Update `proxy_host`, `proxy_port`, `proxy_user`, and `proxy_pass` with your proxy credentials.
- Modify `allowed_ports` if additional ports are needed (e.g., `8080` for custom applications or `9200` for Elasticsearch).

### 5. (Optional) Configure SOCKS5 Proxy
If using a SOCKS5 proxy, install `proxychains` on the control node:
```bash
sudo apt install -y proxychains
```
Edit `/etc/proxychains.conf` to configure the proxy:
```conf
socks5 proxy.example.com 1080 your_proxy_user your_proxy_password
```
- Replace `proxy.example.com`, `1080`, `your_proxy_user`, and `your_proxy_password` with your proxy details.

### 6. Test Ansible Connectivity
Verify SSH connectivity to a target server (e.g., `server1`):
```bash
ansible -i inventory/hosts.yml server1 -m ping
```
Expected output:
```
server1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
If using a SOCKS5 proxy:
```bash
proxychains ansible -i inventory/hosts.yml server1 -m ping
```

## Execution Instructions
To install Docker and harden a specific server (e.g., `server1`):
```bash
ansible-playbook -i inventory/hosts.yml playbook.yml --limit server1
```
- **With SOCKS5 proxy**:
  ```bash
  proxychains ansible-playbook -i inventory/hosts.yml playbook.yml --limit server1
  ```
- **Run only Docker installation**:
  ```bash
  ansible-playbook -i inventory/hosts.yml playbook.yml --limit server1 --tags docker
  ```
- **Run only hardening**:
  ```bash
  ansible-playbook -i inventory/hosts.yml playbook.yml --limit server1 --tags hardening
  ```

## Verification
After running the playbook, connect to the target server (e.g., `server1`) to verify:
```bash
ssh ubuntu@192.168.1.101
```

- **Docker Installation**:
  ```bash
  docker --version
  docker-compose --version
  docker run hello-world
  ```
  - Expected: Docker and Docker Compose versions displayed, and `hello-world` container runs successfully.
- **Hardening**:
  - Check firewall rules:
    ```bash
    sudo iptables -L -v -n
    ```
    - Expected: `INPUT` chain with `DROP` policy and allowed ports (22, 80, 443).
  - Verify SSH configuration:
    ```bash
    sudo grep PermitRootLogin /etc/ssh/sshd_config
    ```
    - Expected: `PermitRootLogin no`.
  - Check `fail2ban` status:
    ```bash
    sudo systemctl status fail2ban
    ```
    - Expected: Service is active and running.

## Notes for Users in Iran
- **Sanctions and Proxy**: This project is designed to work in Iran, where internet access is restricted due to sanctions. Ensure your proxy (HTTP or SOCKS5) is reliable for downloading Docker packages and accessing external repositories (e.g., GitHub, Docker Hub).
- **Low-Memory Servers**: For servers with limited RAM (e.g., 1GB), enable a 2GB swap file to prevent memory issues:
  ```bash
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  ```
  Verify swap:
  ```bash
  free -h
  ```
- **Proxy Connectivity**: Test your proxy before running the playbook:
  ```bash
  curl --proxy http://your_proxy_user:your_proxy_password@proxy.example.com:1080 https://www.google.com
  ```

## Customization
- **Additional Ports**: To open more ports (e.g., `8080` for a web app or `9200` for Elasticsearch), update `allowed_ports` in `inventory/group_vars/all.yml`:
  ```yaml
  allowed_ports:
    - 22
    - 80
    - 443
    - 8080
    - 9200
  ```
- **Server-Specific Proxy**: For a server with a different proxy, create `inventory/host_vars/server1.yml`:
  ```yaml
  proxy_host: proxy2.example.com
  proxy_port: 1080
  proxy_user: user2
  proxy_pass: pass2
  ```
- **Other Distributions**: For CentOS or Red Hat, modify roles to use `yum`/`dnf` and `firewalld` instead of `apt` and `iptables`. Contact the maintainer for assistance.
- **Advanced Hardening**: To restrict SSH to specific IPs or add more security measures, edit `roles/hardening/tasks/main.yml`.

## Troubleshooting
- **SSH Connection Errors**:
  - Ensure SSH keys are correctly configured and the proxy is reachable.
  - Test SSH manually:
    ```bash
    ssh -o ProxyCommand="nc -X 5 -x proxy.example.com:1080 %h %p" ubuntu@192.168.1.101
    ```
  - Check `~/.ssh/config` and `ansible_ssh_private_key_file` in `inventory/group_vars/all.yml`.
- **Proxy Issues**:
  - Verify proxy credentials in `inventory/group_vars/all.yml`.
  - Test proxy connectivity (see above).
- **Playbook Failures**:
  - Run the playbook with verbose output for details:
    ```bash
    ansible-playbook -i inventory/hosts.yml playbook.yml --limit server1 -v
    ```
  - Check target server logs (e.g., `/var/log/syslog` or `/var/log/messages`).
- **Memory Issues**:
  - Ensure swap is enabled on low-memory servers (see above).

## Contributing
Contributions are welcome! Please submit issues or pull requests to enhance the project. Ideas for additional features (e.g., monitoring with Prometheus, integration with ELK Stack) are appreciated.

## License
This project is licensed under the [MIT License](LICENSE).

## Contact
For questions or support, open an issue on GitHub or contact the maintainer.
