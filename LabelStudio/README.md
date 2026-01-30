# Label Studio with SRAM Authentication

## Overview

This Ansible playbook deploys [Label Studio](https://labelstud.io/) - an open-source data labeling platform - with SRAM (SURF Research Access Management) authentication for SURF Research Cloud.

Label Studio is ideal for:
- **LLM Fine-Tuning**: Label data for supervised fine-tuning or RLHF
- **LLM Evaluations**: Response moderation, grading, and side-by-side comparison
- **RAG Evaluation**: Human feedback and Ragas scores
- **Computer Vision**: Image classification, object detection, semantic segmentation
- **NLP**: Named entity recognition, sentiment analysis, text classification
- **Audio**: Transcription, speaker diarization, emotion recognition

## Architecture

```
User (SRAM Login) 
    ↓
SURF Research Cloud Portal
    ↓
nginx (plugin-nginx) → SRAM OAuth2 Validation → Label Studio
    ↑                                               ↓
  SSL/TLS                                     Data Storage
  Certbot                                   (Docker Volume)
```

## How It Works

### Authentication Flow

1. User clicks "Access" button in SURF Research Cloud
2. User is redirected to SRAM login page (institutional SSO)
3. After successful authentication, SRAM redirects back
4. nginx validates the SRAM token via `/validate` endpoint
5. Username is passed to Label Studio via `REMOTE_USER` header
6. User gains access to Label Studio with their institutional identity

### Key Features

- ✅ **SRAM SSO**: Institutional single sign-on authentication
- ✅ **SSL/TLS**: All traffic encrypted via Let's Encrypt
- ✅ **Persistent Data**: Labeling data stored in Docker volumes
- ✅ **Large File Support**: Up to 100MB file uploads configured
- ✅ **WebSocket Support**: Real-time collaboration features
- ✅ **Health Monitoring**: Built-in health check endpoint

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

### Optional Variables

```yaml
label_studio_version: "latest"    # Docker image tag
app_dir: "/opt/label-studio"      # Installation directory
```

## Deployment Order

1. **plugin-nginx** runs first:
   - nginx installation
   - SSL/TLS with certbot
   - SRAM OAuth2 authentication framework

2. **LabelStudio_SRAM.yml** runs second:
   - Docker environment
   - Label Studio container
   - nginx location configs
   - Triggers nginx reload

## Testing the Deployment

After deployment, verify:

1. **Docker container is running**:
   ```bash
   docker ps
   # Should show: label-studio
   ```

2. **Label Studio is healthy**:
   ```bash
   curl http://localhost:8080/health
   # Should return OK
   ```

3. **nginx is configured**:
   ```bash
   nginx -t
   cat /etc/nginx/app-location-conf.d/labelstudio.conf
   ```

4. **Access the service**:
   - Navigate to `https://your-app.surf-hosted.nl`
   - Should redirect to SRAM login
   - After login, should show Label Studio

## Troubleshooting

### Authentication Issues

```bash
# Check nginx error logs
tail -f /var/log/nginx/error.log

# Verify SRAM endpoints
curl -I https://sram.surf.nl/oauth2/authorize

# Check auth config
cat /etc/nginx/app-location-conf.d/authentication.conf
```

### Container Issues

```bash
# Check container logs
docker logs label-studio

# Restart service
cd /opt/label-studio
docker compose restart

# Check container health
docker inspect label-studio | grep -A 10 Health
```

### File Upload Issues

If large file uploads fail:
```bash
# Check nginx config for client_max_body_size
grep client_max_body_size /etc/nginx/app-location-conf.d/labelstudio.conf
```

## Data Management

### Using SurfDrive / ResearchDrive with rclone

Label Studio can access video and image files from SurfDrive/ResearchDrive using rclone.

#### Initial Setup

1. **Configure rclone** (one-time setup):
   ```bash
   rclone config
   # Follow prompts to add your SurfDrive/ResearchDrive
   # Name it something like: ResearchDrive_BamBam
   ```

2. **Create mount point**:
   ```bash
   sudo mkdir -p /mnt/BamBam
   sudo chown $USER:$USER /mnt/BamBam
   ```

3. **Mount your research data**:
   ```bash
   rclone mount "ResearchDrive_BamBam:250814736_BAMBAM (Projectfolder)" /mnt/BamBam \
     --allow-other \
     --vfs-cache-mode writes \
     --daemon
   ```

4. **Verify mount**:
   ```bash
   ls -la /mnt/BamBam/
   # Should show your project folders
   ```

5. **Restart Label Studio** to pick up the mount:
   ```bash
   cd /opt/label-studio
   sudo docker compose down
   sudo docker compose up -d
   ```

6. **Verify container can see files**:
   ```bash
   sudo docker exec -it label-studio ls -la /label-studio/BamBam/
   # Should show your project folders
   ```

#### After Server Restart

If your VM restarts, the rclone mount will be lost. Remount it with:

```bash
# 1. Remount rclone
rclone mount "ResearchDrive_BamBam:250814736_BAMBAM (Projectfolder)" /mnt/BamBam \
  --allow-other \
  --vfs-cache-mode writes \
  --daemon

# 2. Verify mount is active
ls -la /mnt/BamBam/

# 3. Restart Docker containers to pick up the mount
cd /opt/label-studio
sudo docker compose down
sudo docker compose up -d

# 4. Verify container sees the files
sudo docker exec -it label-studio ls -la /label-studio/BamBam/
```

#### Auto-mount on Boot (Optional)

To automatically mount on server restart, create a systemd service:

```bash
# Create the service file
sudo nano /etc/systemd/system/rclone-bambam.service
```

Add this content:
```ini
[Unit]
Description=RClone mount for BamBam ResearchDrive
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=aras
Group=aras
ExecStart=/usr/bin/rclone mount \
  "ResearchDrive_BamBam:250814736_BAMBAM (Projectfolder)" /mnt/BamBam \
  --allow-other \
  --vfs-cache-mode writes \
  --vfs-cache-max-age 24h
ExecStop=/bin/fusermount -u /mnt/BamBam
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable rclone-bambam.service
sudo systemctl start rclone-bambam.service

# Check status
sudo systemctl status rclone-bambam.service
```

#### Connecting Storage in Label Studio

1. Go to your project → **Settings → Cloud Storage**
2. Click **Add Source Storage**
3. Select **Local Files**
4. Configure:
   - **Storage Title**: `BamBam`
   - **Absolute local path**: `/label-studio/BamBam/BAMBAM_Onderzoek`
   - **Import Method**: `Files - Automatically creates a task for each storage object`
   - **File Name Filter**: `.*\.mp4$` (for videos only)
   - **Scan all sub-folders**: ✅ Toggle ON
5. Click **Load Preview** to verify
6. Click **Next** → **Sync Storage**

### Backup Label Studio Data

```bash
# Create backup of Label Studio data
docker run --rm -v label-studio-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/labelstudio-backup-$(date +%Y%m%d).tar.gz /data
```

### Restore Data

```bash
# Restore from backup
docker run --rm -v label-studio-data:/data -v $(pwd):/backup \
  alpine tar xzf /backup/labelstudio-backup-YYYYMMDD.tar.gz -C /
```

## Recommended VM Specifications

- **CPU**: 4+ cores
- **RAM**: 8+ GB (16GB for large datasets)
- **Storage**: 50+ GB (depends on data volume)
- **OS**: Ubuntu 22.04 or Debian 11+

## Customization

### Using PostgreSQL Database

For production workloads, consider using PostgreSQL instead of SQLite:

```yaml
# Add to docker-compose.yml
services:
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_USER=labelstudio
      - POSTGRES_PASSWORD=secure_password
      - POSTGRES_DB=labelstudio
    volumes:
      - postgres-data:/var/lib/postgresql/data

  label-studio:
    environment:
      - DJANGO_DB=default
      - POSTGRE_NAME=labelstudio
      - POSTGRE_USER=labelstudio
      - POSTGRE_PASSWORD=secure_password
      - POSTGRE_HOST=postgres
      - POSTGRE_PORT=5432
```

### Connecting Cloud Storage

Label Studio supports S3, GCS, and Azure Blob storage for data import/export:
- Configure in Label Studio UI under Project Settings → Cloud Storage
- Use pre-signed URLs for secure access

## References

- [Label Studio Documentation](https://labelstud.io/guide/)
- [Label Studio Docker Deployment](https://labelstud.io/guide/install.html#Docker)
- [Label Studio API Reference](https://labelstud.io/api/)
- [Labeling Templates](https://labelstud.io/templates/)
