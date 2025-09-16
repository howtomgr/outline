# Outline Installation Guide

Modern team knowledge base with real-time collaboration, powerful search, and intuitive interface. Essential tool for documentation, note-taking, and team knowledge sharing with enterprise-grade security.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Linux system (any modern distribution)
- Docker and Docker Compose
- Root or sudo access
- PostgreSQL or MySQL database
- Redis for caching
- S3-compatible storage (MinIO, AWS S3)
- SMTP server for notifications


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### Docker Compose (Recommended)
```bash
# Create Outline directory
mkdir -p ~/outline
cd ~/outline

# Generate secrets
SECRET_KEY=$(openssl rand -hex 32)
UTILS_SECRET=$(openssl rand -hex 32)

# Create Docker Compose configuration
cat > docker-compose.yml <<EOF
version: '3.8'

services:
  outline:
    image: outlinewiki/outline:latest
    container_name: outline
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      # Server configuration
      - SECRET_KEY=${SECRET_KEY}
      - UTILS_SECRET=${UTILS_SECRET}
      - DATABASE_URL=postgres://outline:outline_password@postgres:5432/outline
      - REDIS_URL=redis://redis:6379
      
      # Application settings
      - URL=https://docs.example.com
      - PORT=3000
      - FORCE_HTTPS=true
      
      # Authentication (Slack example)
      - SLACK_CLIENT_ID=your_slack_client_id
      - SLACK_CLIENT_SECRET=your_slack_client_secret
      
      # Alternative: Google OAuth
      # - GOOGLE_CLIENT_ID=your_google_client_id
      # - GOOGLE_CLIENT_SECRET=your_google_client_secret
      
      # File storage (S3-compatible)
      - AWS_ACCESS_KEY_ID=minio_access_key
      - AWS_SECRET_ACCESS_KEY=minio_secret_key
      - AWS_REGION=us-east-1
      - AWS_S3_UPLOAD_BUCKET_URL=http://minio:9000
      - AWS_S3_UPLOAD_BUCKET_NAME=outline-uploads
      - AWS_S3_UPLOAD_MAX_SIZE=26214400
      - AWS_S3_FORCE_PATH_STYLE=true
      
      # Email configuration
      - SMTP_HOST=smtp.example.com
      - SMTP_PORT=587
      - SMTP_USERNAME=outline@example.com
      - SMTP_PASSWORD=smtp_secure_password
      - SMTP_FROM_EMAIL=outline@example.com
      - SMTP_REPLY_EMAIL=outline@example.com
      - SMTP_TLS_CIPHERS=
      - SMTP_SECURE=true
      
      # Security settings
      - ENABLE_UPDATES=false
      - WEB_CONCURRENCY=1
      - MAXIMUM_IMPORT_SIZE=5120000
      - DEBUG=cache,presenters,events
      
    volumes:
      - outline_data:/var/lib/outline/data
    depends_on:
      - postgres
      - redis
      - minio
    networks:
      - outline

  postgres:
    image: postgres:15-alpine
    container_name: outline-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=outline
      - POSTGRES_USER=outline
      - POSTGRES_PASSWORD=outline_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - outline

  redis:
    image: redis:alpine
    container_name: outline-redis
    restart: unless-stopped
    command: redis-server --requirepass redis_secure_password
    volumes:
      - redis_data:/data
    networks:
      - outline

  minio:
    image: minio/minio:latest
    container_name: outline-minio
    restart: unless-stopped
    ports:
      - "127.0.0.1:9000:9000"
      - "127.0.0.1:9001:9001"
    environment:
      - MINIO_ROOT_USER=minio_access_key
      - MINIO_ROOT_PASSWORD=minio_secret_key
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"
    networks:
      - outline

networks:
  outline:
    driver: bridge

volumes:
  outline_data:
  postgres_data:
  redis_data:
  minio_data:
EOF

# Start Outline stack
docker-compose up -d

# Create S3 bucket for uploads
docker exec outline-minio mc alias set local http://localhost:9000 minio_access_key minio_secret_key
docker exec outline-minio mc mb local/outline-uploads
docker exec outline-minio mc policy set public local/outline-uploads

# Access Outline at https://localhost:3000
```

### NGINX Reverse Proxy with SSL
```bash
sudo tee /etc/nginx/sites-available/outline > /dev/null <<EOF
server {
    listen 80;
    server_name docs.example.com;
    return 301 https://\$server_name\$request_uri;
}

server {
    listen 443 ssl http2;
    server_name docs.example.com;

    ssl_certificate /etc/letsencrypt/live/docs.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/docs.example.com/privkey.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-XSS-Protection "1; mode=block" always;

    # File upload limits
    client_max_body_size 25M;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        
        # WebSocket support for real-time collaboration
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/outline /etc/nginx/sites-enabled/
sudo certbot --nginx -d docs.example.com
sudo nginx -t && sudo systemctl reload nginx
```

## Backup and Monitoring

### Backup Strategy
```bash
sudo tee /usr/local/bin/outline-backup.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DIR="/backup/outline"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_DIR}/{database,uploads}

echo "Starting Outline backup..."

# Backup PostgreSQL database
docker exec outline-postgres pg_dump -U outline outline > ${BACKUP_DIR}/database/outline-${DATE}.sql
gzip ${BACKUP_DIR}/database/outline-${DATE}.sql

# Backup MinIO uploads
docker exec outline-minio mc cp --recursive local/outline-uploads /tmp/uploads-backup/
docker cp outline-minio:/tmp/uploads-backup ${BACKUP_DIR}/uploads/outline-uploads-${DATE}

# Upload to cloud storage
aws s3 cp ${BACKUP_DIR}/ s3://outline-backups/ --recursive

# Keep last 14 backups
find ${BACKUP_DIR} -name "outline-*" -type f -mtime +14 -delete

echo "Outline backup completed: ${DATE}"
EOF

sudo chmod +x /usr/local/bin/outline-backup.sh
echo "0 2 * * * root /usr/local/bin/outline-backup.sh" | sudo tee -a /etc/crontab
```

### Health Monitoring
```bash
sudo tee /usr/local/bin/outline-health.sh > /dev/null <<'EOF'
#!/bin/bash
LOG="/var/log/outline-health.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a ${LOG}
}

# Check Outline container
if docker ps --filter name=outline --filter status=running -q | grep -q .; then
    log_message "✓ Outline container running"
else
    log_message "✗ Outline container not running"
fi

# Check web interface
if curl -f http://localhost:3000/health >/dev/null 2>&1; then
    log_message "✓ Outline web interface responding"
else
    log_message "✗ Outline web interface not responding"
fi

# Check dependencies
if docker ps --filter name=outline-postgres --filter status=running -q | grep -q .; then
    log_message "✓ PostgreSQL database running"
else
    log_message "✗ PostgreSQL database not running"
fi

if docker ps --filter name=outline-redis --filter status=running -q | grep -q .; then
    log_message "✓ Redis cache running"
else
    log_message "✗ Redis cache not running"
fi

log_message "Outline health check completed"
EOF

sudo chmod +x /usr/local/bin/outline-health.sh
echo "*/15 * * * * root /usr/local/bin/outline-health.sh" | sudo tee -a /etc/crontab
```

## Additional Resources

- [Outline Documentation](https://docs.getoutline.com/)
- [Outline GitHub](https://github.com/outline/outline)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection.