# Local LLM Deployment with SSO Authentication & HTTPS

This Ansible playbook automates the deployment of a production-ready, self-hosted Large Language Model (LLM) infrastructure with Single Sign-On (SSO) authentication using OIDC/SAML integration and automatic HTTPS certificate management.

## 🚀 Overview

This solution deploys a complete, production-ready LLM stack that includes:

- **Ollama**: Backend LLM engine for running open-source language models
- **Open WebUI**: Modern web interface for interacting with the LLM
- **OAuth2 Proxy**: SSO authentication layer using OIDC
- **Nginx**: Reverse proxy with HTTPS termination and automatic HTTP→HTTPS redirects
- **Certbot**: Automatic Let's Encrypt SSL certificate provisioning and renewal
- **Docker Compose**: Container orchestration for all services

All services run in Docker containers and are secured behind SSO authentication with TLS encryption, ensuring only authorized users can access the LLM over HTTPS.

## 📋 Prerequisites

- **Target System**: Ubuntu/Debian-based Linux server
- **Ansible**: Version 2.9 or higher installed on your control machine
- **System Requirements**:
  - 4GB+ RAM (8GB+ recommended for better model performance)
  - 20GB+ disk space for models
  - CPU with AVX/AVX2 support (recommended for faster inference)
- **DNS & Network Requirements**:
  - **DNS A record** pointing your FQDN to the server's public IP (required for Let's Encrypt)
  - **Port 80** open to the internet (required for Let's Encrypt HTTP-01 challenge)
  - **Port 443** open to the internet (for HTTPS access)
  - Server must be able to:
    - Pull Docker images from public registries
    - Reach your OIDC provider endpoint
    - Reach Let's Encrypt servers for certificate validation

## 🔑 Required Variables

The playbook requires the following variables to be provided:

| Variable | Description | Example | Required |
|----------|-------------|---------|----------|
| `PUBLIC_FQDN` | Fully qualified domain name for accessing the service (must have DNS A record) | `llm.example.com` | **Yes** |
| `OIDC_ISSUER_URL` | Your OIDC provider's issuer URL | `https://auth.example.com/realms/myrealm` | **Yes** |
| `OIDC_CLIENT_ID` | OAuth2 client ID registered with your OIDC provider | `local-llm-client` | **Yes** |
| `OIDC_CLIENT_SECRET` | OAuth2 client secret | `super-secret-key-here` | **Yes** |
| `CERTBOT_EMAIL` | Email for Let's Encrypt certificate notifications | `admin@example.com` | No (recommended) |
| `COOKIE_SECRET` | Secret for cookie encryption | Auto-generated if not provided | No |

### Setting Up Variables

You can provide these variables in several ways:

#### Option 1: Ansible Extra Vars
```bash
ansible-playbook playbook.yml \
  -e PUBLIC_FQDN=llm.example.com \
  -e OIDC_ISSUER_URL=https://auth.example.com/... \
  -e OIDC_CLIENT_ID=your-client-id \
  -e OIDC_CLIENT_SECRET=your-secret \
  -e CERTBOT_EMAIL=admin@example.com
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
CERTBOT_EMAIL=admin@example.com
```

#### Option 3: Group Vars File
Create `group_vars/all.yml`:
```yaml
PUBLIC_FQDN: llm.example.com
OIDC_ISSUER_URL: https://auth.example.com/...
OIDC_CLIENT_ID: your-client-id
OIDC_CLIENT_SECRET: your-secret
CERTBOT_EMAIL: admin@example.com
```

## 🎯 Usage

### Basic Deployment

1. **Ensure DNS is configured**:
   - Create an A record pointing your FQDN to the server's public IP
   - Wait for DNS propagation (use `nslookup` or `dig` to verify)

2. **Prepare your inventory**:
```bash
echo "your-server-ip ansible_user=ubuntu" > inventory.ini
```

3. **Run the playbook**:
```bash
ansible-playbook -i inventory.ini playbook.yml \
  -e PUBLIC_FQDN=llm.example.com \
  -e OIDC_ISSUER_URL=https://auth.example.com/... \
  -e OIDC_CLIENT_ID=your-client-id \
  -e OIDC_CLIENT_SECRET=your-secret \
  -e CERTBOT_EMAIL=admin@example.com
```

4. **Access your LLM**:
   - Navigate to `https://your-fqdn/` in your browser (HTTPS enforced)
   - HTTP requests are automatically redirected to HTTPS
   - You'll be redirected to your SSO provider for authentication
   - After authentication, you'll access the Open WebUI interface with a valid SSL certificate

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
   - Starts base services (nginx, oauth2-proxy, open-webui, ollama)
   - Downloads the `gemma3:4b` model automatically
   - Configures health checks and service dependencies

4. **SSL Certificate Management**:
   - Obtains Let's Encrypt SSL certificate using HTTP-01 challenge
   - Configures nginx with HTTPS termination
   - Sets up automatic certificate renewal via cron job (runs daily at 3:15 AM)
   - Restarts nginx to load the new certificates

5. **Model Download**:
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
                    │   Nginx :80/443 │
                    │  (HTTPS Proxy + │
                    │   HTTP→HTTPS)   │
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
                                    │
                            ┌───────▼────────┐
                            │   Certbot      │
                            │ (Let's Encrypt)│
                            └────────────────┘
```

### Service Details

- **Nginx (Ports 80/443)**: HTTPS-enabled reverse proxy
  - **Port 80**: Handles Let's Encrypt HTTP-01 challenges and redirects all other traffic to HTTPS
  - **Port 443**: HTTPS termination with Let's Encrypt certificates
  - Routes `/` to Open WebUI (with auth)
  - Routes `/oauth2/` to OAuth2 Proxy
  - Routes `/ollama/` to Ollama API (with auth)
  - Handles WebSocket upgrades for real-time streaming
  - Enforces HTTPS with automatic HTTP→HTTPS redirects

- **OAuth2 Proxy (Port 4180)**: Authentication gateway
  - Validates all requests via OIDC
  - Manages authentication cookies (secure flag enabled)
  - Redirects unauthenticated users to SSO provider
  - Configured for HTTPS-only operation

- **Open WebUI (Port 8080)**: Web interface
  - Chat interface for LLM interaction
  - Model management
  - Conversation history
  - User settings

- **Ollama (Port 11434)**: LLM runtime
  - Runs language models (default: Gemma 3 4B)
  - Provides REST API for inference
  - Manages model storage and loading

- **Certbot**: SSL certificate management
  - Obtains Let's Encrypt certificates on initial deployment
  - Automatic renewal via cron job (daily at 3:15 AM)
  - Stores certificates in persistent Docker volume

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

1. **HTTPS Enforced**: The setup automatically configures HTTPS with Let's Encrypt certificates:
   - All HTTP traffic is redirected to HTTPS
   - SSL certificates are automatically renewed
   - TLS 1.2+ with modern cipher suites

2. **Cookie Security**: Cookies are marked secure and HTTP-only in OAuth2 Proxy configuration

3. **Access Control**: All endpoints are protected by SSO authentication via OIDC

4. **Secrets Management**: The `.env` file contains sensitive data and has restricted permissions (0600)

5. **Network Isolation**: All services communicate over a private Docker network

6. **Security Headers**: Nginx includes security headers:
   - X-Frame-Options: SAMEORIGIN
   - X-Content-Type-Options: nosniff
   - Referrer-Policy: strict-origin-when-cross-origin

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
docker compose logs -f nginx
```

### Check SSL Certificate Status
```bash
# Check certificate expiration
docker compose exec nginx openssl x509 -in /etc/letsencrypt/live/your-domain.com/fullchain.pem -text -noout | grep "Not After"

# Test certificate renewal
/usr/local/bin/renew-llm-certs.sh
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

### SSL Certificate Issues
- **Certificate not obtained**: Ensure DNS A record points to server and port 80 is accessible
- **Certificate renewal fails**: Check cron job: `sudo crontab -l`
- **HTTPS not working**: Verify certificate files exist: `docker compose exec nginx ls -la /etc/letsencrypt/live/`
- **Mixed content errors**: Ensure all resources use HTTPS URLs

### Authentication Loop
- Verify `PUBLIC_FQDN` matches your actual domain
- Check `OAUTH2_PROXY_REDIRECT_URL` is registered in your OIDC provider
- Ensure your OIDC provider is accessible from the server
- Verify HTTPS redirect URL in OIDC provider matches `https://your-domain/oauth2/callback`

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
- Verify port availability: `sudo netstat -tulpn | grep -E ':(80|443)'`
- Ensure Docker service is running: `sudo systemctl status docker`
- Check certificate volume mounts: `docker compose exec nginx ls -la /etc/letsencrypt/`

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

**Note**: This setup includes production-ready HTTPS with automatic certificate management. For additional production hardening, consider implementing monitoring, backups, and additional security measures.

