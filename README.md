# Ansible AI & Automation Deployment

This project provides a comprehensive automated deployment system for containerized AI services (Ollama, OpenWebUI, n8n) using Ansible and Docker. It features role-based service deployment with Traefik reverse proxy integration, requiring external Docker networks to be pre-configured by complementary infrastructure roles. Some of the configurations in this project have been targetted to AMD GPUs using
Vulkan, but can be easily changed to fit your hardware.

## Project Structure

```
deploy_ai_and_automation_playbook.yml  # Main deployment playbook
group_vars/
├── main.yml                           # Default configuration variables (paths, flags, domains)
├── vault.yml                          # Sensitive data (credentials, API keys) — encrypted
└── vault.yml.template                 # Template for creating vault.yml
inventory/
└── hosts                              # Target machine definitions (localhost by default)
roles/
├── llms/                              # LLMs (Ollama + OpenWebUI)
│   ├── tasks/
│   │   ├── main.yml                   # Network validation and conditional deployment
│   │   ├── ollama.yml                 # Ollama container with GPU support
│   │   └── openwebui.yml              # OpenWebUI container with ollama integration
│   └── templates/
│       ├── ollama.yml.j2              # Ollama Docker Compose template (AMD Vulkan GPU)
│       └── openwebui.yml.j2           # OpenWebUI Docker Compose template
└── automation/                         # Automation services (n8n)
  ├── tasks/
  │   ├── main.yml                   # Network validation and conditional deployment
  │   └── n8n.yml                    # n8n container deployment and setup
  └── templates/
    └── n8n.yml.j2                 # n8n Docker Compose template
```

## Key Features

### AI Services
- **Ollama**: LLM inference server with GPU acceleration (AMD Vulkan)
- **OpenWebUI**: Chat interface integrated with Ollama backend
- **n8n**: Workflow automation and integration platform

### Integration Points
- **Traefik**: Reverse proxy with automatic routing (requires pre-existing `proxy_network`)
- **Docker Compose**: Service orchestration via Jinja2-templated compose files
- **Docker Service Discovery**: Internal container communication (e.g., OpenWebUI → Ollama via DNS)

## Prerequisites

Before you begin, ensure the following:
- Ubuntu/Debian-based Linux system with Docker Engine (20.x+) and Docker Compose v2
- Python 3 and Ansible (2.10+ recommended)
- `community.docker` collection installed:
  ```bash
  ansible-galaxy collection install community.docker
  ```
- **External Docker Networks** (created by separate infrastructure roles):
  - `proxy_network` (default: `proxy`) — for Traefik reverse proxy integration
  - `home_lab_network` (default: `home_lab`) — for internal service communication
- A Traefik instance already deployed and monitoring the proxy network
- Domain configuration for service subdomains (e.g., `ollama.example.com`, `openwebui.example.com`)

## Getting Started

### 1. Clone and Configure

```bash
git clone <repository-url>
cd ai_and_automation

# Copy vault template
cp group_vars/vault.yml.template group_vars/vault.yml

# Edit vault.yml with your domain and device configuration
vim group_vars/vault.yml
```

### 2. Review Configuration

Edit `group_vars/main.yml` to customize:
- `mount_point_docker_volumes`: Path for Ollama models storage
- `openwebui_data_volume`: OpenWebUI persistent data location
- `n8n_data_file` / `n8n_files_path`: n8n configuration locations
- `deploy_llms` / `deploy_automation`: Enable/disable services
- Network names if your infrastructure uses different names

### 3. Run the Deployment

```bash
# Full deployment (all services) with vault and become password prompts
ansible-playbook -i inventory/hosts \
  --ask-vault-pass \
  --ask-become \
  deploy_ai_and_automation_playbook.yml

# Deploy only LLMS (skip automation)
ansible-playbook -i inventory/hosts \
  --ask-vault-pass \
  --ask-become \
  -e "deploy_llms=true deploy_automation=false" \
  deploy_ai_and_automation_playbook.yml

# Deploy only automation (n8n)
ansible-playbook -i inventory/hosts \
  --ask-vault-pass \
  --ask-become \
  -e "deploy_llms=false deploy_automation=true" \
  deploy_ai_and_automation_playbook.yml

# With debug mode to preserve rendered compose files in /tmp/
# (useful for troubleshooting variable rendering)
ansible-playbook -i inventory/hosts \
  --ask-vault-pass \
  --ask-become \
  -e debug_mode=true \
  deploy_ai_and_automation_playbook.yml

# Alternative: using vault password file (if you've created one securely)
ansible-playbook -i inventory/hosts \
  --vault-password-file ~/.vault_pass.txt \
  --ask-become \
  deploy_ai_and_automation_playbook.yml
```

## Architecture Details

### Network Dependencies
The deployment **requires pre-existing Docker networks**. Before running this playbook, ensure:
1. `{{ proxy_network_name }}` exists and is connected to Traefik
2. Services are validated in `roles/*/tasks/main.yml` before deployment

```bash
# Verify networks exist
docker network ls | grep -E "proxy|home_lab"
```

### Container Initialization Guards
All roles use the idempotent pattern of checking if containers exist before deployment:
```yaml
- community.docker.docker_container_info:
    name: ollama
  register: ollama_info

- ansible.builtin.template:
    src: ollama.yml.j2
  when: not ollama_info.exists
```

**Why?** Prevents accidental overwriting of running container configurations.

### Service Communication
- **OpenWebUI ↔ Ollama**: Uses Docker Compose service discovery
  - OpenWebUI environment: `OLLAMA_API_BASE_URL=http://ollama:11434`
  - Both services must be in the same Compose project
- **All Services ↔ Traefik**: Via shared `proxy_network` and Traefik labels

### GPU Support (AMD)
The Ollama template is configured for AMD GPUs via Vulkan:
```yaml
environment:
  - RENDER_DEVICE=/dev/dri/renderD128
devices:
  - /dev/dri:/dev/dri
```

**Porting to NVIDIA?** Replace Vulkan environment variables with CUDA settings and adjust device mappings.

## Configuration Reference

| Variable | File | Purpose |
|----------|------|---------|
| `mount_point_docker_volumes` | `group_vars/main.yml` | Root path for persistent container data |
| `models_path` | `group_vars/main.yml` | Ollama model storage location |
| `openwebui_data_volume` | `group_vars/main.yml` | OpenWebUI persistent data directory |
| `n8n_data_file` | `group_vars/main.yml` | n8n SQLite database and encryption key storage |
| `proxy_network_name` | `group_vars/main.yml` | Traefik network name (must exist) |
| `vault_domain` | `group_vars/vault.yml` | Base domain for service subdomains |
| `deploy_*` flags | `group_vars/main.yml` | Enable/disable service deployment (e.g. `deploy_llms`, `deploy_automation`) |
| `debug_mode` | `group_vars/main.yml` | Preserve rendered compose files in `/tmp/` for inspection |

## Troubleshooting

### Network Validation Failures
```
Required Docker network 'proxy' does not exist
```
**Solution**: Create the network via complementary infrastructure roles or manually:
```bash
docker network create proxy
```

### Service Won't Connect to Ollama
**Issue**: OpenWebUI can't reach Ollama
- Verify `OLLAMA_API_BASE_URL=http://ollama:11434` in OpenWebUI environment
- Confirm both services are in same Docker Compose project (check `docker ps`)
- Check logs: `docker logs openwebui`

### GPU Not Detected
**Issue**: Ollama container not using GPU
```bash

# Check rendering
ansible-playbook -i inventory/hosts deploy_ai_and_automation_playbook.yml -e debug_mode=true
cat /tmp/llms/ollama.yml

# Verify device access
docker exec ollama ls -la /dev/dri/
```

### Traefik Certificate Issues
**Issue**: Traefik can't obtain SSL certificates
- Verify `vault_domain` and domain prefix variables are set
- Confirm Traefik is running and connected to `proxy_network`
- Check Traefik logs for DNS/ACME errors

### Compose File Rendering Issues
Enable debug mode to inspect rendered templates:
```bash
ansible-playbook -i inventory/hosts deploy_ai_and_automation_playbook.yml -e debug_mode=true
# Inspect rendered templates in `/tmp/llms/` and `/tmp/automation/` (e.g. `/tmp/llms/ollama.yml`)
```

## Common Workflows

### Update Ollama Models Path
1. Edit `group_vars/main.yml` → `models_path`
2. Stop and remove container: `docker container rm ollama`
3. Re-run playbook (container check will pass and the role will render the new compose file under `/tmp/llms/`)

### Enable n8n Deployment
1. Ensure `deploy_automation: true` in `group_vars/main.yml` (or run with `-e "deploy_automation=true"`)
2. Verify `n8n_domain_prefix`, `n8n_data_file` and `n8n_files_path` are set in `group_vars/main.yml`
3. Run playbook

### Debug Variable Rendering
1. Set `debug_mode: true` in `group_vars/main.yml`
2. Run playbook
3. Inspect rendered files in `/tmp/llms/` and `/tmp/automation/`

## Dependencies

This playbook assumes:
- External Docker networks exist (created by `base_network_watchtower` and `core_infrastructure` roles)
- Traefik instance is running and monitoring the proxy network
- Host has Docker with compose v2 support

## License

This project is licensed under the MIT License. See the LICENSE file for more details.
