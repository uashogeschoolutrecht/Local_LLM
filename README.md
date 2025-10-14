# Ollama + Open WebUI Integration with SURF Research Cloud (SRAM Authentication)

## Overview

This Ansible playbook deploys Ollama (local LLM) and Open WebUI with SRAM (SURF Research Access Management) authentication integration for SURF Research Cloud.

## Architecture

```
User (SRAM Login) 
    ↓
SURF Research Cloud Portal
    ↓
nginx (plugin-nginx) → SRAM OAuth2 Validation → Open WebUI → Ollama
    ↑                                                 ↓
  SSL/TLS                                        LLM Model
  Certbot
```

## How It Works

### 1. **User Access Flow**
   - User clicks the yellow "Access" button in SURF Research Cloud
   - User is redirected to SRAM login page (institutional SSO)
   - After successful authentication, SRAM redirects back to your service
   - nginx validates the SRAM token via `/validate` endpoint
   - User gains access to Open WebUI

### 2. **Component Responsibilities**

#### **plugin-nginx (1).yml** (SURF's Plugin)
- Installs and configures nginx
- Sets up SSL/TLS certificates with certbot
- Configures SRAM OAuth2 authentication endpoints
- Creates `/validate` endpoint for auth validation
- Manages the `@custom_401` redirect for unauthenticated users

#### **Ollama_OpenWeb.yml** (Your Application Playbook)
- Installs Docker and Docker Compose
- Deploys Ollama container with CPU support
- Pulls the specified LLM model (default: gemma2:2b)
- Deploys Open WebUI container
- Creates nginx location configs that integrate with SRAM auth
- Configures proxying with authentication protection

### 3. **Integration Points**

The playbooks integrate via:
- **Shared nginx configuration**: Your playbook writes to `/etc/nginx/app-location-conf.d/openwebui.conf`
- **Authentication validation**: Uses `auth_request /validate;` directive
- **RSC variables**: Both playbooks receive variables from SURF Research Cloud catalog

## SURF Research Cloud Catalog Configuration

### Required Variables (Set in RSC Catalog)

These variables are automatically provided by SURF Research Cloud:

```yaml
rsc_nginx_service_url: "your-app.surf-hosted.nl"
rsc_nginx_authorization_endpoint: "https://sram.surf.nl/oauth2/authorize"
rsc_nginx_user_info_endpoint: "https://sram.surf.nl/api/user"
rsc_nginx_oauth2_application:
  client_id: "your-client-id"
rsc_nginx_co_id: "your-collaboration-id"
rsc_nginx_co_role: "member"
```

### Optional Variables (Can be customized)

```yaml
ollama_model: "gemma2:2b"  # Change to any Ollama model
app_dir: "/opt/local-llm"   # Installation directory
```

## Deployment Order

1. **plugin-nginx** runs first and sets up:
   - nginx installation
   - SSL/TLS with certbot
   - SRAM OAuth2 authentication framework
   - Base authentication endpoints

2. **Ollama_OpenWeb** runs second and sets up:
   - Docker environment
   - Ollama + Open WebUI containers
   - Application-specific nginx configs
   - Triggers nginx reload to apply changes

## Key Features

### Security
- ✅ All traffic encrypted with SSL/TLS (Let's Encrypt)
- ✅ SRAM institutional authentication required
- ✅ No Open WebUI internal authentication (SRAM handles all auth)
- ✅ All services bound to localhost (only nginx exposed)

### Performance
- ✅ CPU-optimized Ollama deployment
- ✅ Lightweight model (gemma2:2b) by default
- ✅ Streaming response support for LLMs
- ✅ WebSocket support for real-time updates

### User Experience
- ✅ Single Sign-On via SRAM
- ✅ Automatic redirect to login if not authenticated
- ✅ Seamless access after authentication
- ✅ No separate Open WebUI login required

## File Structure

```
CPU_Based_LLM/
├── plugin-nginx (1).yml      # SURF's nginx + SRAM auth plugin
├── Ollama_OpenWeb.yml         # Your Ollama + Open WebUI deployment
└── README_INTEGRATION.md      # This file
```

## Testing the Deployment

After deployment, verify:

1. **Docker containers are running**:
   ```bash
   docker ps
   # Should show: ollama, open-webui, ollama-model-puller
   ```

2. **Model is loaded**:
   ```bash
   docker exec ollama ollama list
   # Should show your model (gemma2:2b)
   ```

3. **nginx is running**:
   ```bash
   systemctl status nginx
   ```

4. **SSL certificate is valid**:
   ```bash
   ls -l /etc/letsencrypt/live/$(hostname -f)/
   ```

5. **Access the service**:
   - Navigate to `https://your-app.surf-hosted.nl`
   - Should redirect to SRAM login
   - After login, should show Open WebUI

## Troubleshooting

### Authentication Issues

If users can't log in:
```bash
# Check nginx error logs
tail -f /var/log/nginx/error.log

# Verify SRAM endpoints are reachable
curl -I https://sram.surf.nl/oauth2/authorize

# Check authentication config
cat /etc/nginx/app-location-conf.d/authentication.conf
```

### Container Issues

If containers aren't running:
```bash
# Check container logs
docker logs ollama
docker logs open-webui

# Restart services
cd /opt/local-llm
docker compose restart
```

### Model Loading Issues

If the model doesn't load:
```bash
# Check model puller logs
docker logs ollama-model-puller

# Manually pull model
docker exec ollama ollama pull gemma2:2b
```

### nginx Issues

If nginx isn't proxying correctly:
```bash
# Test nginx config
nginx -t

# Reload nginx
systemctl reload nginx

# Check app location config
cat /etc/nginx/app-location-conf.d/openwebui.conf
```

## Customization

### Using a Different Model

Edit `Ollama_OpenWeb.yml` and change:
```yaml
vars:
  ollama_model: "llama2:7b"  # or any other Ollama model
```

### Adjusting Timeouts

Modify the nginx location config timeouts in the playbook:
```yaml
proxy_connect_timeout 600;  # Increase if model is slow
proxy_send_timeout 600;
proxy_read_timeout 600;
```

### Adding More Services

You can extend the docker-compose.yml in the playbook to add:
- Vector databases (ChromaDB, Qdrant)
- Additional models
- Monitoring tools

## SURF Research Cloud Catalog Item Setup

To use this in SURF Research Cloud:

1. **Create a new Catalog Item** in RSC
2. **Add the nginx plugin** as a dependency
3. **Set the playbook** to `Ollama_OpenWeb.yml`
4. **Configure required variables**:
   - Service URL (DNS name)
   - SRAM OAuth2 credentials
   - Collaboration ID and role
5. **Deploy** to a VM with sufficient resources

### Recommended VM Specifications

- **CPU**: 4+ cores
- **RAM**: 8+ GB (16 GB recommended for larger models)
- **Storage**: 50+ GB
- **OS**: Ubuntu 22.04 or Debian 11+

## Support

For issues related to:
- **SRAM authentication**: Contact SURF support
- **nginx plugin**: Check SURF RSC plugin repository
- **Ollama/Open WebUI**: Check respective GitHub repositories
- **This playbook**: Review logs and troubleshooting section above

## License

This playbook is provided as-is for use with SURF Research Cloud.

