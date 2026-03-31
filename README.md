# mtproto-bootstrap

Ansible role for deploying a [Telegram MTProto proxy](https://github.com/TelegramMessenger/MTProxy) with Fake TLS support on any Linux VPS.

## What It Does

- Installs Docker (optional)
- Runs the official `telegrammessenger/proxy` container
- Generates a Fake TLS secret that disguises traffic as regular HTTPS
- Saves the configuration and outputs a ready-to-use `tg://proxy?...` link

## Requirements

- VPS running any Linux distribution (Ubuntu, Debian, CentOS, etc.)
- Minimum 512 MB RAM, 5 GB disk
- SSH access with root or sudo privileges
- Ansible 2.15+ on the control machine

## Quick Start

### 1. Clone the repository

```bash
git clone git@github.com:alilxxey/mtproto-bootstrap.git
cd mtproto-bootstrap
```

### 2. Install dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Configure inventory

Edit `inventory/hosts.yml` — set the IP address and SSH user of your VPS:

```yaml
---
all:
  hosts:
    mtproto:
      ansible_host: "203.0.113.10"    # Your VPS IP
      ansible_user: root               # SSH user
```

If using a non-default SSH key:

```yaml
      ansible_ssh_private_key_file: "~/.ssh/my_key"
```

### 4. Run the playbook

```bash
ansible-playbook playbooks/site.yml
```

On completion, the output will display connection details:

```
====================================
MTProto Proxy deployed successfully!
====================================
Server: 203.0.113.10
Port:   443
Secret: ee79612e727500000000000000000000
Domain: ya.ru
====================================
Link: tg://proxy?server=203.0.113.10&port=443&secret=ee79612e727500000000000000000000
====================================
```

### 5. Connect in Telegram

**Option 1 — via link:**
Open the `tg://proxy?...` link from the playbook output directly in Telegram — the proxy will be added automatically.

**Option 2 — manually:**

| Platform | Path |
|----------|------|
| Android / iOS | Settings → Data and Storage → Proxy Settings → Add Proxy → MTProto |
| Desktop | Settings → Advanced → Connection type → Use custom proxy |

Enter the Server, Port, and Secret from the playbook output.

## Variables

All variables are defined in `roles/mtproto_proxy/defaults/main.yml` and can be overridden in `inventory/hosts.yml`, `group_vars/`, `host_vars/`, or via `-e` at runtime.

| Variable | Default | Description |
|----------|---------|-------------|
| `mtproto_container_name` | `mtproto-proxy` | Docker container name |
| `mtproto_image` | `telegrammessenger/proxy:latest` | Docker image |
| `mtproto_port` | `443` | Port the proxy listens on |
| `mtproto_restart_policy` | `unless-stopped` | Container restart policy |
| `mtproto_fake_tls_domain` | `ya.ru` | Domain used for Fake TLS masquerading |
| `mtproto_secret` | `""` (auto-generated) | Secret key. Empty value triggers auto-generation |
| `mtproto_install_docker` | `true` | Install Docker if not found |
| `mtproto_config_path` | `/etc/mtproto-proxy` | Directory for storing config and secret on the host |

### Override Examples

**Via `inventory/hosts.yml`:**

```yaml
---
all:
  hosts:
    mtproto:
      ansible_host: "203.0.113.10"
      ansible_user: root
      mtproto_port: 8443
      mtproto_fake_tls_domain: "google.com"
```

**Via command line:**

```bash
ansible-playbook playbooks/site.yml -e "mtproto_port=8443 mtproto_fake_tls_domain=google.com"
```

**With an explicit secret (skip auto-generation):**

```bash
ansible-playbook playbooks/site.yml -e "mtproto_secret=ee79612e7275abcdef1234567890abcd"
```

## How Fake TLS Works

Fake TLS is an MTProto proxy mode where the connection is disguised as regular HTTPS traffic to a specified domain. This makes proxy detection via DPI (Deep Packet Inspection) significantly harder.

The secret is constructed as follows:

1. The domain (e.g. `ya.ru`) is converted to hex: `7961 2e72 75`
2. Padded with random bytes to reach 30 hex characters
3. Prefixed with `ee` — the Fake TLS marker

The resulting secret is 32 hex characters long and starts with `ee`.

When connecting, the Telegram client performs a TLS handshake that appears to an outside observer as a normal HTTPS connection to the domain encoded in the secret.

## Project Structure

```
mtproto-bootstrap/
├── ansible.cfg                 # Ansible configuration
├── requirements.yml            # Dependency: community.docker
├── inventory/
│   └── hosts.yml               # Host inventory and variables
├── playbooks/
│   └── site.yml                # Main playbook entrypoint
└── roles/
    └── mtproto_proxy/
        ├── defaults/main.yml   # Default variables
        ├── handlers/main.yml   # Container restart handler
        ├── tasks/
        │   ├── main.yml        # Role entrypoint
        │   ├── docker.yml      # Docker installation
        │   └── proxy.yml       # Secret generation, container launch
        └── templates/
            └── mtproto_config.txt.j2  # Config file template
```

## What Happens on Run

1. **docker.yml** (when `mtproto_install_docker: true`):
   - Checks if Docker is installed on the host
   - If not found — downloads and runs `get-docker.sh`
   - Ensures `docker.service` is running and enabled at boot

2. **proxy.yml**:
   - Creates the `/etc/mtproto-proxy` directory
   - Checks if a saved secret already exists
   - If no secret — generates a new Fake TLS secret
   - If secret exists — reads it from file (keeps the connection link stable across runs)
   - Pulls the `telegrammessenger/proxy` Docker image
   - Starts the container with the mapped port and secret
   - Detects the server's public IP
   - Saves the config to `/etc/mtproto-proxy/mtproto_config.txt`
   - Displays connection details

## Idempotency

The role is fully idempotent:

- Docker is not reinstalled if already present
- The secret is not regenerated — it is read from file
- The container is not recreated if the configuration hasn't changed
- The connection link remains stable across runs

## Container Management

After deployment, the container is managed with standard Docker commands:

```bash
# Status
docker ps | grep mtproto-proxy

# Logs
docker logs mtproto-proxy
docker logs -f mtproto-proxy  # follow

# Restart
docker restart mtproto-proxy

# Stop
docker stop mtproto-proxy

# Remove
docker rm -f mtproto-proxy
```

Configuration is saved at `/etc/mtproto-proxy/mtproto_config.txt` on the host.

## Firewall

This role does **not** manage the firewall. Make sure the proxy port (default 443) is open.

## Changing the Port

If port 443 is already in use (e.g. by a web server), specify an alternative:

```bash
ansible-playbook playbooks/site.yml -e "mtproto_port=8443"
```

Don't forget to open the new port in your firewall.
