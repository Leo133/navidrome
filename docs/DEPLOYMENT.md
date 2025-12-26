# Deployment Guide

This document describes how to deploy the Navidrome fork to various environments.

## Deployment Options

| Method | Complexity | Best For |
|--------|------------|----------|
| Docker | Low | Most deployments |
| Docker Compose | Low | Development, small instances |
| Kubernetes | High | Large-scale, production |
| Binary | Medium | Custom configurations |

## Docker Deployment

### Quick Start

```bash
docker run -d \
  --name navidrome \
  -p 4533:4533 \
  -v /path/to/music:/music:ro \
  -v /path/to/data:/data \
  -e ND_MUSICFOLDER=/music \
  <your-registry>/navidrome:latest
```

### Docker Compose

Create `docker-compose.yml`:

```yaml
version: "3"
services:
  navidrome:
    image: <your-registry>/navidrome:latest
    ports:
      - "4533:4533"
    environment:
      ND_MUSICFOLDER: /music
      ND_DATAFOLDER: /data
      ND_LOGLEVEL: info
      ND_SCANSCHEDULE: "@every 1h"
    volumes:
      - /path/to/music:/music:ro
      - navidrome-data:/data

volumes:
  navidrome-data:
```

Start:
```bash
docker-compose up -d
```

### Full Stack (with External Services)

For social and analytics features, deploy the full stack:

```yaml
version: "3"
services:
  navidrome:
    image: <your-registry>/navidrome:latest
    ports:
      - "4533:4533"
    environment:
      ND_MUSICFOLDER: /music
      ND_DATAFOLDER: /data
      ND_NATS_URL: nats://nats:4222
    volumes:
      - /path/to/music:/music:ro
      - navidrome-data:/data
    depends_on:
      - nats

  nats:
    image: nats:2.10
    ports:
      - "4222:4222"
    command: ["--jetstream"]

  social-service:
    image: <your-registry>/social-service:latest
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/social
      NATS_URL: nats://nats:4222
    depends_on:
      - postgres
      - nats

  analytics-service:
    image: <your-registry>/analytics-service:latest
    environment:
      DATABASE_URL: postgres://user:pass@timescaledb:5432/analytics
      NATS_URL: nats://nats:4222
    depends_on:
      - timescaledb
      - nats

  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: social
    volumes:
      - postgres-data:/var/lib/postgresql/data

  timescaledb:
    image: timescale/timescaledb:latest-pg15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: analytics
    volumes:
      - timescale-data:/var/lib/postgresql/data

  redis:
    image: redis:7
    volumes:
      - redis-data:/data

volumes:
  navidrome-data:
  postgres-data:
  timescale-data:
  redis-data:
```

## Binary Deployment

### Download Binary

```bash
# Download latest release
curl -L https://github.com/<your-org>/navidrome/releases/latest/download/navidrome_linux_amd64.tar.gz | tar xz
```

### Configuration

Create `navidrome.toml`:

```toml
MusicFolder = "/path/to/music"
DataFolder = "/path/to/data"
LogLevel = "info"
Port = 4533
ScanSchedule = "@every 1h"

# Optional: External services
# NATSUrl = "nats://localhost:4222"
```

### Systemd Service

Create `/etc/systemd/system/navidrome.service`:

```ini
[Unit]
Description=Navidrome Music Server
After=network.target

[Service]
User=navidrome
Group=navidrome
Type=simple
ExecStart=/opt/navidrome/navidrome
WorkingDirectory=/opt/navidrome
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable navidrome
sudo systemctl start navidrome
```

## Kubernetes Deployment

### Helm Chart (Basic)

```yaml
# values.yaml
replicaCount: 1

image:
  repository: <your-registry>/navidrome
  tag: latest

service:
  type: ClusterIP
  port: 4533

persistence:
  data:
    enabled: true
    size: 10Gi
  music:
    enabled: true
    existingClaim: music-pvc

env:
  ND_MUSICFOLDER: /music
  ND_DATAFOLDER: /data
  ND_LOGLEVEL: info

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: navidrome
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  rules:
    - host: music.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: navidrome
                port:
                  number: 4533
```

## Configuration Reference

### Core Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `MusicFolder` | - | Path to music library |
| `DataFolder` | `.` | Path to data directory |
| `LogLevel` | `info` | Log level |
| `Address` | `0.0.0.0` | Bind address |
| `Port` | `4533` | Server port |
| `BaseUrl` | `` | External URL base |

### Scanner Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `ScanSchedule` | `@every 1m` | Cron expression |
| `EnableAutomaticScan` | `true` | Auto-scan on startup |

### Security Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `SessionTimeout` | `24h` | JWT session timeout |
| `AuthRequestLimit` | `5` | Login rate limit |

### External Services (Fork)

| Setting | Default | Description |
|---------|---------|-------------|
| `NATSUrl` | `` | NATS server URL |
| `EnableTelemetry` | `true` | Enable event emission |
| `TelemetryBatchSize` | `50` | Event batch size |

## Environment Variables

All settings can be configured via environment variables with `ND_` prefix:

```bash
ND_MUSICFOLDER=/music
ND_DATAFOLDER=/data
ND_LOGLEVEL=debug
ND_PORT=4533
```

## Reverse Proxy

### Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name music.example.com;

    ssl_certificate /etc/letsencrypt/live/music.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/music.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:4533;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Disable buffering for streaming
        proxy_buffering off;
    }
}
```

### Caddy

```
music.example.com {
    reverse_proxy localhost:4533
}
```

## Monitoring

### Health Check

```bash
curl http://localhost:4533/ping
# Returns: {"status":"ok"}
```

### Prometheus Metrics

Enable metrics endpoint:
```toml
EnablePrometheus = true
PrometheusMetricsPath = "/metrics"
```

Scrape config:
```yaml
scrape_configs:
  - job_name: 'navidrome'
    static_configs:
      - targets: ['navidrome:4533']
```

### Logs

```bash
# Docker
docker logs -f navidrome

# Systemd
journalctl -u navidrome -f
```

## Backup

### SQLite Database

```bash
# Stop service (optional, for consistency)
docker-compose stop navidrome

# Backup
cp /path/to/data/navidrome.db /path/to/backup/navidrome.db.$(date +%Y%m%d)

# Start service
docker-compose start navidrome
```

### Automated Backup Script

```bash
#!/bin/bash
BACKUP_DIR=/backups/navidrome
DATE=$(date +%Y%m%d_%H%M%S)

# Backup database
sqlite3 /data/navidrome.db ".backup ${BACKUP_DIR}/navidrome_${DATE}.db"

# Keep last 7 days
find ${BACKUP_DIR} -name "*.db" -mtime +7 -delete
```

## Troubleshooting

### Common Issues

**Port already in use:**
```bash
# Find process
lsof -i :4533
# Kill or change port
```

**Permission denied on music folder:**
```bash
# Fix permissions
chown -R 1000:1000 /path/to/music
# Or run container as correct user
docker run --user $(id -u):$(id -g) ...
```

**Database locked:**
- Ensure only one instance is running
- Check for zombie processes

### Debug Mode

```bash
# Enable debug logging
ND_LOGLEVEL=debug ./navidrome
```

## References

- [Navidrome Documentation](https://www.navidrome.org/docs/)
- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
