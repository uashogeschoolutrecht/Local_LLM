# Local LLM Deployment with SSO Authentication

This Ansible playbook automates the deployment of a self-hosted Large Language Model (LLM) infrastructure with Single Sign-On (SSO) authentication using OIDC/SAML integration.

## 🚀 Overview

This solution deploys a complete LLM stack that includes:

- **Ollama**: Backend LLM engine for running open-source language models
- **Open WebUI**: Modern web interface for interacting with the LLM
- **OAuth2 Proxy**: SSO authentication layer using OIDC
- **Nginx**: Reverse proxy for routing and request management
- **Docker Compose**: Container orchestration for all services

All services run in Docker containers and are secured behind SSO authentication, ensuring only authorized users can access the LLM.

## 📋 Prerequisites

- **Target System**: Ubuntu/Debian-based Linux server
- **Ansible**: Version 2.9 or higher installed on your control machine
- **System Requirements**:
  - 4GB+ RAM (8GB+ recommended for better model performance)
  - 20GB+ disk space for models
  - CPU with AVX/AVX2 support (recommended for faster inference)
- **Network Access**: Target server must be able to:
  - Pull Docker images from public registries
  - Reach your OIDC provider endpoint
  - Be accessible from your network (for web access)

## 🔑 Required Variables

The playbook requires the following variables to be provided:

| Variable | Description | Example |
|----------|-------------|---------|
| `PUBLIC_FQDN` | Fully qualified domain name for accessing the service | `llm.example.com` |
| `OIDC_ISSUER_URL` | Your OIDC provider's issuer URL | `https://auth.example.com/realms/myrealm` |
| `OIDC_CLIENT_ID` | OAuth2 client ID registered with your OIDC provider | `local-llm-client` |
| `OIDC_CLIENT_SECRET` | OAuth2 client secret | `super-secret-key-here` |
| `COOKIE_SECRET` | (Optional) Secret for cookie encryption | Auto-generated if not provided |

### Setting Up Variables

You can provide these variables in several ways:

#### Option 1: Ansible Extra Vars
```bash
ansible-playbook playbook.yml \
  -e PUBLIC_FQDN=llm.example.com \
  -e OIDC_ISSUER_URL=https://auth.example.com/... \
  -e OIDC_CLIENT_ID=your-client-id \
  -e OIDC_CLIENT_SECRET=your-secret
```

#### Option 2: Inventory File
Create an inventory file with your variables:
```ini
[llm-servers]
192.168.1.100 ansible_user=ubuntu

[llm-servers:vars]
PUBLIC_FQDN=llm.example.com
OIDC_ISSUER_URL=https://auth.example.com/...
OIDC_CLIENT_ID=your-client-id
OIDC_CLIENT_SECRET=your-secret
```

#### Option 3: Group Vars File
Create `group_vars/all.yml`:
```yaml
PUBLIC_FQDN: llm.example.com
OIDC_ISSUER_URL: https://auth.example.com/...
OIDC_CLIENT_ID: your-client-id
OIDC_CLIENT_SECRET: your-secret
```

## 🎯 Usage

### Basic Deployment

1. **Prepare your inventory**:
```bash
echo "your-server-ip ansible_user=ubuntu" > inventory.ini
```

2. **Run the playbook**:
```bash
ansible-playbook -i inventory.ini playbook.yml \
  -e PUBLIC_FQDN=llm.example.com \
  -e OIDC_ISSUER_URL=https://auth.example.com/... \
  -e OIDC_CLIENT_ID=your-client-id \
  -e OIDC_CLIENT_SECRET=your-secret
```

3. **Access your LLM**:
   - Navigate to `http://your-fqdn/` in your browser
   - You'll be redirected to your SSO provider
   - After authentication, you'll access the Open WebUI interface

### What the Playbook Does

The playbook executes the following steps automatically:

1. **System Preparation**:
   - Updates package cache
   - Installs Docker and dependencies
   - Sets up Docker Compose plugin

2. **Application Setup**:
   - Creates application directory at `/opt/local-llm`
   - Generates configuration files (docker-compose.yml, nginx.conf, .env)
   - Configures OAuth2 proxy with your OIDC settings

3. **Service Deployment**:
   - Pulls latest Docker images
   - Starts all containers
   - Downloads the `gemma3:4b` model automatically
   - Configures health checks and service dependencies

4. **Model Download**:
   - A dedicated container pulls the Gemma 3 (4B parameter) model
   - This happens automatically on first deployment
   - The model is stored in a persistent Docker volume

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                             │
└────────────────────────────┬────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   Nginx :80     │
                    │  (Reverse Proxy)│
                    └────────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
        ┌───────▼────────┐       ┌───────▼──────────┐
        │ OAuth2 Proxy   │       │   Open WebUI     │
        │   (SSO/OIDC)   │◄──────┤   :8080          │
        └────────────────┘       └──────────┬───────┘
                                            │
                                    ┌───────▼────────┐
                                    │   Ollama       │
                                    │   :11434       │
                                    │ (LLM Engine)   │
                                    └────────────────┘
```

### Service Details

- **Nginx (Port 80)**: Entry point for all requests
  - Routes `/` to Open WebUI (with auth)
  - Routes `/oauth2/` to OAuth2 Proxy
  - Routes `/ollama/` to Ollama API (with auth)
  - Handles WebSocket upgrades for real-time streaming

- **OAuth2 Proxy (Port 4180)**: Authentication gateway
  - Validates all requests via OIDC
  - Manages authentication cookies
  - Redirects unauthenticated users to SSO provider

- **Open WebUI (Port 8080)**: Web interface
  - Chat interface for LLM interaction
  - Model management
  - Conversation history
  - User settings

- **Ollama (Port 11434)**: LLM runtime
  - Runs language models (default: Gemma 3 4B)
  - Provides REST API for inference
  - Manages model storage and loading

## 🔧 Configuration

### Changing the Model

To use a different model, edit the `docker-compose.yml` generation in the playbook (line 105):

```yaml
command: ["pull", "your-model-name"]
```

Available models: [Ollama Library](https://ollama.com/library)

Popular options:
- `llama3.2:3b` - Meta's Llama 3.2 (3B)
- `phi4:3.5b` - Microsoft's Phi-4 (3.5B)
- `gemma3:4b` - Google's Gemma 3 (4B, default)
- `mistral:7b` - Mistral 7B
- `qwen2.5:7b` - Alibaba's Qwen 2.5 (7B)

### Custom Application Directory

Change the `app_dir` variable in the playbook:

```yaml
vars:
  app_dir: /your/custom/path
```

### Adjusting Timeout Settings

For large model responses, you may need to adjust timeout values in the nginx configuration (lines 176-179).

## 🔒 Security Considerations

1. **HTTPS Required for Production**: The current setup uses HTTP. For production:
   - Configure TLS certificates (Let's Encrypt recommended)
   - Update nginx to listen on port 443
   - Enable HTTPS redirects

2. **Cookie Security**: Cookies are marked secure in OAuth2 Proxy configuration

3. **Access Control**: All endpoints are protected by SSO authentication

4. **Secrets Management**: The `.env` file contains sensitive data and has restricted permissions (0600)

5. **Network Isolation**: All services communicate over a private Docker network

## 📊 Monitoring and Management

### Check Service Status
```bash
cd /opt/local-llm
docker compose ps
```

### View Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f ollama
docker compose logs -f open-webui
docker compose logs -f oauth2-proxy
```

### Restart Services
```bash
docker compose restart
```

### Update Services
```bash
docker compose pull
docker compose up -d
```

### Download Additional Models
```bash
docker exec -it ollama ollama pull llama3.2:3b
```

### List Available Models
```bash
docker exec -it ollama ollama list
```

## 🐛 Troubleshooting

### Authentication Loop
- Verify `PUBLIC_FQDN` matches your actual domain
- Check `OAUTH2_PROXY_REDIRECT_URL` is registered in your OIDC provider
- Ensure your OIDC provider is accessible from the server

### Model Not Loading
- Check available disk space: `df -h`
- View model-puller logs: `docker compose logs model-puller`
- Manually pull model: `docker exec -it ollama ollama pull gemma3:4b`

### Slow Responses
- Increase server resources (RAM/CPU)
- Use a smaller model
- Check CPU supports AVX instructions: `grep avx /proc/cpuinfo`

### Container Won't Start
- Check logs: `docker compose logs [service-name]`
- Verify port availability: `sudo netstat -tulpn | grep :80`
- Ensure Docker service is running: `sudo systemctl status docker`

## 📝 Requirements File

This playbook requires the `community.docker` collection. Install it with:

```bash
ansible-galaxy collection install community.docker
```

Or create a `requirements.yml`:
```yaml
collections:
  - name: community.docker
    version: ">=3.0.0"
```

Then install:
```bash
ansible-galaxy collection install -r requirements.yml
```

## 📚 Additional Resources

- [Ollama Documentation](https://github.com/ollama/ollama/blob/main/docs/README.md)
- [Open WebUI Documentation](https://docs.openwebui.com/)
- [OAuth2 Proxy Documentation](https://oauth2-proxy.github.io/oauth2-proxy/)
- [Ansible Docker Module](https://docs.ansible.com/ansible/latest/collections/community/docker/)

## 📄 License

See the LICENSE file in the repository root.

## 🤝 Support

For issues or questions:
1. Check the troubleshooting section above
2. Review service logs
3. Verify OIDC configuration with your SSO provider

---

**Note**: This setup is designed for internal/development use. For production deployments, implement proper TLS/SSL, monitoring, backups, and security hardening.

