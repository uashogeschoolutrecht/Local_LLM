# SURF Research Cloud Applications with SRAM Authentication

## Overview

This repository contains Ansible playbooks for deploying research applications with SRAM (SURF Research Access Management) authentication integration for SURF Research Cloud.

**Available Applications:**

### 🤖 Ollama + Open WebUI (Local LLM)
- **CPU-Based**: Optimized for CPU-only inference (suitable for smaller models)
- **GPU-Based**: NVIDIA GPU-accelerated for faster inference (5-10x speedup)

### 🏷️ Label Studio (Data Labeling Platform)
- Open-source data labeling for LLM fine-tuning, RLHF, and AI evaluation
- Supports images, text, audio, video, and time series data

📊 **See [COMPARISON.md](COMPARISON.md) for detailed comparison and selection guide**

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
ollama_models:              # List of models to download
  - gemma2:2b
  - phi4-mini-reasoning:3.8b
app_dir: "/opt/local-llm"   # Installation directory
```

## Deployment Order

1. **plugin-nginx** runs first and sets up:
   - nginx installation
   - SSL/TLS with certbot
   - SRAM OAuth2 authentication framework
   - Base authentication endpoints

2. **Ollama_OpenWeb (CPU or GPU)** runs second and sets up:
   - Docker environment
   - Ollama + Open WebUI containers (with or without GPU support)
   - Application-specific nginx configs
   - Triggers nginx reload to apply changes
   
   Choose one:
   - `CPU_Based_LLM/Ollama_OpenWeb.yml` for CPU deployment
   - `GPU_Based_LLM/Ollama_OpenWeb_GPU.yml` for GPU deployment

## Key Features

### Security
- ✅ All traffic encrypted with SSL/TLS (Let's Encrypt)
- ✅ SRAM institutional authentication required
- ✅ No Open WebUI internal authentication (SRAM handles all auth)
- ✅ All services bound to localhost (only nginx exposed)

### Performance
- ✅ CPU or GPU-optimized Ollama deployment options
- ✅ Flexible model selection (gemma2:2b default)
- ✅ Multiple model support (download several models at once)
- ✅ Streaming response support for LLMs
- ✅ WebSocket support for real-time updates
- ✅ GPU version offers 5-10x faster inference

### User Experience
- ✅ Single Sign-On via SRAM
- ✅ Automatic redirect to login if not authenticated
- ✅ Seamless access after authentication
- ✅ No separate Open WebUI login required

## File Structure

```
.
├── CPU_Based_LLM/
│   └── Ollama_OpenWeb.yml         # CPU-based Ollama deployment
│
├── GPU_Based_LLM/
│   └── Ollama_OpenWeb_GPU.yml     # GPU-accelerated Ollama deployment
│
├── LabelStudio/
│   ├── LabelStudio_SRAM.yml       # Label Studio deployment
│   └── README.md                  # Label Studio documentation
│
├── README.md                      # Main documentation
└── LICENSE
```

## Application Selection Guide

### Label Studio (`LabelStudio/LabelStudio_SRAM.yml`)
**Use for:**
- Data labeling for ML/AI training
- LLM fine-tuning and RLHF workflows
- AI model evaluation and benchmarking
- Multi-modal annotation (text, images, audio, video)
- Collaborative labeling projects

**Features:**
- Flexible labeling templates
- ML-assisted labeling
- Cloud storage integration (S3, GCS)
- Team collaboration support

📖 **See [LabelStudio/README.md](LabelStudio/README.md) for detailed documentation**

---

## Choosing Between CPU and GPU (Ollama)

### CPU Version (`CPU_Based_LLM/Ollama_OpenWeb.yml`)
**Use when:**
- No GPU available
- Running smaller models (< 3B parameters)
- Cost-sensitive deployment
- Lower inference volume

**Recommended for:**
- Testing and development
- Small teams (< 10 users)
- Non-critical workloads

### GPU Version (`GPU_Based_LLM/Ollama_OpenWeb_GPU.yml`)
**Use when:**
- NVIDIA GPU available
- Running larger models (7B+ parameters)
- High performance required
- Multiple concurrent users

**Recommended for:**
- Production workloads
- Research with large models
- High inference volume
- Latency-sensitive applications

**Performance Comparison:**
| Model | CPU Inference | GPU Inference | Speedup |
|-------|---------------|---------------|---------|
| gemma2:2b | 15-20 tok/s | 100-150 tok/s | 6-7x |
| phi4-mini:3.8b | 10-15 tok/s | 80-120 tok/s | 8x |
| llama3.1:8b | 5-8 tok/s | 60-90 tok/s | 10-12x |

## Testing the Deployment

After deployment, verify:

1. **Docker containers are running**:
   ```bash
   docker ps
   # Should show: ollama, open-webui
   ```

2. **Models are loaded**:
   ```bash
   docker exec ollama ollama list
   # Should show your models (gemma2:2b, phi4-mini-reasoning:3.8b, etc.)
   ```

3. **GPU access (GPU version only)**:
   ```bash
   docker exec ollama nvidia-smi
   # Should show GPU information and usage
   ```

4. **nginx is running**:
   ```bash
   systemctl status nginx
   ```

5. **SSL certificate is valid**:
   ```bash
   ls -l /etc/letsencrypt/live/$(hostname -f)/
   ```

6. **Access the service**:
   - Navigate to `https://your-app.surf-hosted.nl`
   - Should redirect to SRAM login
   - After login, should show Open WebUI
   - All downloaded models should be available in the dropdown

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

If models don't load:
```bash
# Check Ollama logs
docker logs ollama

# List available models
docker exec ollama ollama list

# Manually pull a model
docker exec ollama ollama pull gemma2:2b

# Remove a model to free space
docker exec ollama ollama rm <model-name>
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

### Using Multiple Models

Both CPU and GPU versions now support downloading multiple models at once:

```yaml
vars:
  ollama_models:
    - gemma2:2b
    - phi4-mini-reasoning:3.8b
    - llama3.2:3b
    # Add as many models as needed
```

**Note:** Make sure you have sufficient disk space for multiple models.

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

#### For CPU Version
- **CPU**: 8+ cores (16+ for better performance)
- **RAM**: 8-16 GB (depending on model size)
- **Storage**: 50+ GB
- **OS**: Ubuntu 22.04 or Debian 11+

#### For GPU Version
- **CPU**: 4+ cores
- **RAM**: 16+ GB
- **GPU**: NVIDIA GPU with 4GB+ VRAM (8GB+ recommended)
- **Storage**: 100+ GB
- **OS**: Ubuntu 22.04 or Debian 11+
- **Drivers**: NVIDIA drivers (will be validated by playbook)

## Support

For issues related to:
- **SRAM authentication**: Contact SURF support
- **nginx plugin**: Check SURF RSC plugin repository
- **Ollama/Open WebUI**: Check respective GitHub repositories
- **This playbook**: Review logs and troubleshooting section above

## License

This playbook is provided as-is for use with SURF Research Cloud.


