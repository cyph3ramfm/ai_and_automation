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
├── llms/                              # LLMs + Document Processing (Ollama, OpenWebUI, SearXNG, Tika, Qdrant)
│   ├── tasks/
│   │   ├── main.yml                   # Network validation and conditional deployment
│   │   ├── ollama.yml                 # Ollama container with GPU support
│   │   ├── openwebui.yml              # OpenWebUI container with ollama integration
│   │   ├── searxng.yml                # SearXNG container for web search
│   │   ├── tika.yml                   # Tika container for OCR/document processing
│   │   └── qdrant.yml                 # Qdrant vector database for RAG
│   └── templates/
│       ├── ollama.yml.j2              # Ollama Docker Compose template (AMD Vulkan GPU)
│       ├── openwebui.yml.j2           # OpenWebUI Docker Compose template
│       ├── searxng.yml.j2             # SearXNG Docker Compose template with Redis
│       ├── tika.yml.j2                # Tika Docker Compose template with Traefik routing
│       └── qdrant.yml.j2              # Qdrant Docker Compose template
└── automation/                         # Automation services (n8n)
    ├── tasks/
    │   ├── main.yml                   # Network validation and conditional deployment
    │   └── n8n.yml                    # n8n container deployment and setup
    └── templates/
        └── n8n.yml.j2                 # n8n Docker Compose template
```

## Key Features

### AI & Automation Services
- **Ollama**: LLM inference server with GPU acceleration (AMD Vulkan)
- **OpenWebUI**: Chat interface integrated with Ollama backend, web search, document RAG, and vector embeddings
- **SearXNG**: Privacy-respecting metasearch engine for web search integration
- **Tika**: OCR and document processing server for RAG (Retrieval-Augmented Generation)
- **Qdrant**: Vector database for storing and retrieving document embeddings in RAG workflows
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
- `searxng_config_volume` / `searxng_data_volume` / `searxng_redis_data_volume`: SearXNG storage paths
- `tika_cpu_limit` / `tika_memory_limit` / `tika_domain_prefix`: Tika resource limits and domain
- `qdrant_data_volume` / `qdrant_cpu_limit` / `qdrant_memory_limit`: Qdrant vector DB storage and resources
- `n8n_data_file` / `n8n_files_path`: n8n configuration locations
- `deploy_llms` / `deploy_automation`: Enable/disable service groups
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
- **OpenWebUI ↔ Tika**: OCR and document extraction
  - OpenWebUI environment: `DOCUMENT_EXTRACTOR_URL=http://tika:9998`
- **OpenWebUI ↔ Qdrant**: Vector embedding storage and retrieval
  - OpenWebUI environment: `QDRANT_URI=http://qdrant:6333`
- **OpenWebUI ↔ SearXNG**: Web search integration
  - OpenWebUI environment: `SEARXNG_BASE_URL=http://searxng:8080`
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

### RAG Workflow Architecture (Retrieval-Augmented Generation)
OpenWebUI provides a complete RAG pipeline integrating document processing, embedding, and retrieval:

1. **Document Upload** → User uploads document in OpenWebUI
2. **OCR & Extraction** (Tika) → Document text extracted via Tika OCR server (`DOCUMENT_EXTRACTOR_URL=http://tika:9998`)
3. **Embedding Generation** → Extracted text converted to embeddings using OpenWebUI's embedding model
4. **Vector Storage** (Qdrant) → Embeddings stored in Qdrant vector DB (`QDRANT_URI=http://qdrant:6333`) with collection prefix (`QDRANT_COLLECTION_PREFIX=openwebui`)
5. **Chat Query** → User asks question in OpenWebUI chat
6. **Semantic Search** → Query embedded and similar embeddings retrieved from Qdrant
7. **Context + LLM** → Retrieved document context combined with query and sent to Ollama LLM
8. **Response** → Ollama generates response grounded in document content

**Network Requirement**: Qdrant runs on `home_lab_network` (internal only) for secure inter-service communication.

## Configuration Reference

| Variable | File | Purpose |
|----------|------|---------|
| `mount_point_docker_volumes` | `group_vars/main.yml` | Root path for persistent container data |
| `models_path` | `group_vars/main.yml` | Ollama model storage location |
| `openwebui_data_volume` | `group_vars/main.yml` | OpenWebUI persistent data directory |
| `searxng_config_volume` | `group_vars/main.yml` | SearXNG configuration storage |
| `searxng_data_volume` | `group_vars/main.yml` | SearXNG cache storage |
| `searxng_redis_data_volume` | `group_vars/main.yml` | SearXNG Redis data storage |
| `searxng_domain_prefix` | `group_vars/main.yml` | Domain prefix for SearXNG web UI |
| `tika_cpu_limit` | `group_vars/main.yml` | CPU limit for Tika container (e.g., "2.0") |
| `tika_memory_limit` | `group_vars/main.yml` | Memory limit for Tika in MB (e.g., "2048M") |
| `tika_domain_prefix` | `group_vars/main.yml` | Domain prefix for Tika web UI |
| `qdrant_data_volume` | `group_vars/main.yml` | Qdrant vector database storage location |
| `qdrant_collection_prefix` | `group_vars/main.yml` | Collection prefix for Qdrant embeddings (e.g., "openwebui") |
| `qdrant_cpu_limit` | `group_vars/main.yml` | CPU limit for Qdrant container (e.g., "1.0") |
| `qdrant_memory_limit` | `group_vars/main.yml` | Memory limit for Qdrant in MB (e.g., "1024M") |
| `n8n_data_file` | `group_vars/main.yml` | n8n SQLite database and encryption key storage |
| `n8n_files_path` | `group_vars/main.yml` | n8n files storage location |
| `proxy_network_name` | `group_vars/main.yml` | Traefik network name (must exist) |
| `home_lab_network_name` | `group_vars/main.yml` | Internal home lab network name (must exist) |
| `vault_domain` | `group_vars/vault.yml` | Base domain for service subdomains |
| `deploy_llms` | `group_vars/main.yml` | Enable/disable LLM services (Ollama, OpenWebUI, SearXNG, Tika, Qdrant) |
| `deploy_automation` | `group_vars/main.yml` | Enable/disable n8n automation service |
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

### RAG Not Working (Qdrant/Tika/OpenWebUI)
**Issue**: Document embeddings not being stored or retrieved
1. **Verify Qdrant is running**:
   ```bash
   docker ps | grep qdrant
   curl http://localhost:6333/health
   ```
2. **Verify Tika is accessible from OpenWebUI**:
   ```bash
   docker exec openwebui curl -s http://tika:9998/v1/meta
   ```
3. **Check OpenWebUI logs for embedding errors**:
   ```bash
   docker logs openwebui | grep -i "qdrant\|embedding\|vector"
   ```
4. **Verify Qdrant collection was created**:
   ```bash
   docker exec qdrant curl -s http://localhost:6333/collections | jq '.result.collections[] | .name'
   ```
5. **Ensure `VECTOR_DB=qdrant` is set in OpenWebUI**:
   ```bash
   docker exec openwebui env | grep VECTOR_DB
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
