# Safe Deployment Guide for OpenClaw

**Document Purpose:** Practical guidance for securely deploying OpenClaw in production environments  
**Target Audience:** DevOps engineers, System administrators, Security practitioners  
**Last Updated:** February 2026  
**Warning:** This guide provides mitigation strategies, but cannot fully address all architectural security gaps

---

## Pre-Deployment Checklist

Before deploying OpenClaw to any environment beyond isolated development, complete these mandatory security hardening steps:

### Critical Security Hardening (Required)

- [ ] Enable full-disk encryption on host system (BitLocker/LUKS/FileVault)
- [ ] Configure firewall rules to block all unnecessary inbound traffic
- [ ] Set up isolated network segment for AI services
- [ ] Implement TLS termination proxy in front of gateway
- [ ] Configure comprehensive audit logging
- [ ] Set up monitoring and alerting infrastructure
- [ ] Create incident response runbook
- [ ] Establish backup and recovery procedures

### Operational Readiness (Required)

- [ ] Implement cost monitoring and budget alerts
- [ ] Configure rate limiting for all endpoints
- [ ] Set up health check monitoring
- [ ] Create maintenance procedures documentation
- [ ] Train operators on security procedures
- [ ] Establish on-call rotation for incidents
- [ ] Test disaster recovery procedures

---

## Network Security Configuration

### Recommended Architecture

```
                         Internet
                            |
                    [Firewall/WAF]
                            |
                    [TLS Termination]
                      (nginx/caddy)
                            |
                    [Reverse Proxy]
                    (authentication)
                            |
                  [OpenClaw Gateway]
                  (localhost only)
                            |
           +----------------+----------------+
           |                |                |
    [LLM Providers]  [Messaging APIs]  [Local Storage]
```

### Step 1: Configure Gateway for Localhost Only

**Modify docker-compose.yml:**

```yaml
services:
  openclaw-gateway:
    image: openclaw:local
    environment:
      HOME: /home/node
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
    # Remove port exposure - use reverse proxy instead
    # ports:
    #   - "18789:18789"
    networks:
      - openclaw-internal
    command:
      [
        "node",
        "dist/index.js",
        "gateway",
        "--bind", "loopback",  # Changed from "lan"
        "--port", "18789"
      ]

networks:
  openclaw-internal:
    driver: bridge
    internal: true  # No external access
```

### Step 2: Deploy TLS Termination Proxy

**Using Caddy (automatic HTTPS):**

Create `Caddyfile`:

```caddyfile
openclaw.yourdomain.com {
    # Automatic HTTPS with Let's Encrypt
    
    # Rate limiting
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 100
            window 1m
        }
    }
    
    # Authentication (choose one)
    
    # Option A: Basic Auth
    basicauth {
        alice $2a$14$Zkx19XLiW6VYouLHR5NmfOFU0z2GTNmpkT/5qqR7hx7wNQIqyP/l2
    }
    
    # Option B: OAuth2 Proxy (recommended)
    forward_auth oauth2-proxy:4180 {
        uri /oauth2/auth
        copy_headers X-Auth-Request-User X-Auth-Request-Email
    }
    
    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        X-XSS-Protection "1; mode=block"
        Content-Security-Policy "default-src 'self'"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
    
    # Proxy to gateway
    reverse_proxy openclaw-gateway:18789 {
        # Health check
        health_uri /health
        health_interval 30s
        health_timeout 10s
        
        # Timeouts
        transport http {
            dial_timeout 10s
            response_header_timeout 60s
        }
    }
    
    # Access logging
    log {
        output file /var/log/caddy/openclaw-access.log
        format json
    }
}
```

**Using nginx with certbot:**

```nginx
# /etc/nginx/sites-available/openclaw

# Rate limiting
limit_req_zone $binary_remote_addr zone=openclaw_limit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=openclaw_conn:10m;

# HTTP - redirect to HTTPS
server {
    listen 80;
    server_name openclaw.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS
server {
    listen 443 ssl http2;
    server_name openclaw.yourdomain.com;
    
    # SSL certificates (managed by certbot)
    ssl_certificate /etc/letsencrypt/live/openclaw.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/openclaw.yourdomain.com/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Content-Security-Policy "default-src 'self'" always;
    
    # Rate limiting
    limit_req zone=openclaw_limit burst=20 nodelay;
    limit_conn openclaw_conn 10;
    
    # Client body size limit
    client_max_body_size 10M;
    
    # Timeouts
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
    
    # Authentication - use auth_request module
    auth_request /auth;
    
    location = /auth {
        internal;
        proxy_pass http://oauth2-proxy:4180/oauth2/auth;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
    }
    
    # Main proxy
    location / {
        proxy_pass http://openclaw-gateway:18789;
        proxy_http_version 1.1;
        
        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Disable buffering for streaming
        proxy_buffering off;
    }
    
    # Access logging
    access_log /var/log/nginx/openclaw-access.log combined;
    error_log /var/log/nginx/openclaw-error.log warn;
}
```

### Step 3: Configure Firewall Rules

**Using iptables (Linux):**

```bash
#!/bin/bash
# firewall-openclaw.sh

# Flush existing rules
iptables -F
iptables -X

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (restrict to your IP)
iptables -A INPUT -p tcp --dport 22 -s YOUR_IP_ADDRESS -j ACCEPT

# Allow HTTPS only
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow HTTP for Let's Encrypt (temporary)
# iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Log dropped packets
iptables -A INPUT -j LOG --log-prefix "IPTables-Dropped: "

# Save rules
iptables-save > /etc/iptables/rules.v4
```

**Using UFW (Ubuntu/Debian):**

```bash
# Default deny
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH from your IP only
sudo ufw allow from YOUR_IP_ADDRESS to any port 22

# Allow HTTPS
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Verify
sudo ufw status verbose
```

---

## Credential Management

### Immediate Mitigations

Since OpenClaw stores credentials in plaintext, apply these compensating controls:

#### 1. Operating System Level Encryption

**Linux - LUKS Full Disk Encryption:**

```bash
# Verify encryption status
sudo cryptsetup status /dev/mapper/luks-root

# If not encrypted, you'll need to reinstall with encryption
# or migrate to encrypted partition
```

**macOS - FileVault:**

```bash
# Enable FileVault
sudo fdesetup enable

# Verify status
sudo fdesetup status
```

**Windows - BitLocker:**

```powershell
# Enable BitLocker
Enable-BitLocker -MountPoint "C:" -EncryptionMethod XtsAes256 -UsedSpaceOnly

# Verify status
Get-BitLockerVolume
```

#### 2. Restrict File System Permissions

```bash
# Make credentials directory accessible only to OpenClaw user
sudo chown -R openclaw:openclaw ~/.openclaw/credentials
sudo chmod 700 ~/.openclaw/credentials
sudo chmod 600 ~/.openclaw/credentials/*

# Verify
ls -la ~/.openclaw/credentials
```

#### 3. Disable Credential Backups

Add to `.gitignore` and backup exclusions:

```bash
# Add to ~/.gitignore_global
echo ".openclaw/credentials/" >> ~/.gitignore_global

# Exclude from Time Machine (macOS)
tmutil addexclusion ~/.openclaw/credentials

# Exclude from cloud sync
# Dropbox
attr -s com.dropbox.ignored -V 1 ~/.openclaw/credentials
# OneDrive - move outside OneDrive folder
# Google Drive - exclude via Drive settings
```

#### 4. Implement Credential Rotation

Create a rotation script:

```bash
#!/bin/bash
# rotate-credentials.sh

# Rotate OpenAI API key
NEW_OPENAI_KEY=$(generate-new-api-key openai)
echo "${NEW_OPENAI_KEY}" > ~/.openclaw/credentials/openai.txt
openclaw config set providers.openai.apiKey "${NEW_OPENAI_KEY}"

# Rotate Anthropic API key
NEW_ANTHROPIC_KEY=$(generate-new-api-key anthropic)
echo "${NEW_ANTHROPIC_KEY}" > ~/.openclaw/credentials/anthropic.txt
openclaw config set providers.anthropic.apiKey "${NEW_ANTHROPIC_KEY}"

# Rotate gateway token
NEW_GATEWAY_TOKEN=$(openssl rand -base64 32)
echo "${NEW_GATEWAY_TOKEN}" > ~/.openclaw/credentials/gateway-token.txt
openclaw config set gateway.token "${NEW_GATEWAY_TOKEN}"

# Update environment files
sed -i "s/OPENCLAW_GATEWAY_TOKEN=.*/OPENCLAW_GATEWAY_TOKEN=${NEW_GATEWAY_TOKEN}/" .env

echo "Credentials rotated successfully"
echo "Restart gateway for changes to take effect"
```

Schedule rotation:

```cron
# Rotate credentials monthly
0 0 1 * * /path/to/rotate-credentials.sh
```

---

## Cost Monitoring and Budget Controls

### Implement External Cost Tracking

Since OpenClaw lacks built-in cost monitoring, implement external tracking:

#### 1. API Usage Monitoring Script

```python
#!/usr/bin/env python3
# monitor-api-costs.py

import requests
import json
from datetime import datetime, timedelta
import os

# Configuration
ANTHROPIC_API_KEY = os.getenv('ANTHROPIC_API_KEY')
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')
ALERT_THRESHOLD_USD = 1000  # Alert if monthly costs exceed this
SLACK_WEBHOOK_URL = os.getenv('SLACK_WEBHOOK_URL')

def get_openai_usage():
    """Fetch OpenAI API usage for current month"""
    url = "https://api.openai.com/v1/usage"
    headers = {"Authorization": f"Bearer {OPENAI_API_KEY}"}
    
    # Get current month date range
    today = datetime.now()
    start_date = today.replace(day=1).strftime('%Y-%m-%d')
    end_date = today.strftime('%Y-%m-%d')
    
    params = {
        "start_date": start_date,
        "end_date": end_date
    }
    
    response = requests.get(url, headers=headers, params=params)
    data = response.json()
    
    total_cost = sum(day['cost'] for day in data.get('data', []))
    return total_cost / 100  # Convert cents to dollars

def get_anthropic_usage():
    """Fetch Anthropic API usage - requires dashboard scraping or credit monitoring"""
    # Anthropic doesn't have public usage API yet
    # Implement dashboard scraping or manual tracking
    # For now, return estimated based on token counts
    return 0

def send_alert(message):
    """Send alert to Slack"""
    if SLACK_WEBHOOK_URL:
        payload = {"text": f"🚨 OpenClaw Cost Alert: {message}"}
        requests.post(SLACK_WEBHOOK_URL, json=payload)

def main():
    try:
        openai_cost = get_openai_usage()
        anthropic_cost = get_anthropic_usage()
        total_cost = openai_cost + anthropic_cost
        
        print(f"Current month costs:")
        print(f"  OpenAI: ${openai_cost:.2f}")
        print(f"  Anthropic: ${anthropic_cost:.2f}")
        print(f"  Total: ${total_cost:.2f}")
        
        # Check threshold
        if total_cost > ALERT_THRESHOLD_USD:
            alert_msg = f"Monthly costs (${total_cost:.2f}) exceeded threshold (${ALERT_THRESHOLD_USD})"
            send_alert(alert_msg)
            print(f"ALERT: {alert_msg}")
        
        # Log to file for trending
        with open('/var/log/openclaw/cost-tracking.log', 'a') as f:
            timestamp = datetime.now().isoformat()
            f.write(f"{timestamp},{openai_cost},{anthropic_cost},{total_cost}\n")
            
    except Exception as e:
        print(f"Error monitoring costs: {e}")
        send_alert(f"Cost monitoring failed: {e}")

if __name__ == '__main__':
    main()
```

Schedule the script:

```cron
# Run cost check every 6 hours
0 */6 * * * /usr/local/bin/monitor-api-costs.py
```

#### 2. Set Provider-Level Budget Alerts

Configure budget alerts directly in provider dashboards:

**OpenAI:**
1. Go to https://platform.openai.com/account/billing/limits
2. Set "Hard limit" to your monthly budget
3. Set "Soft limit" to 80% of budget for warnings
4. Add email for notifications

**Anthropic:**
1. Go to Anthropic Console > Settings > Billing
2. Configure spending limits
3. Set up email notifications

**Google Cloud (for Gemini):**
1. Go to Billing > Budgets & alerts
2. Create budget for Vertex AI API
3. Set alerts at 50%, 80%, 90%, 100%

#### 3. Implement Rate Limiting

Add rate limiting at the reverse proxy level:

```nginx
# In nginx config
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=60r/m;

location /api {
    limit_req zone=api_limit burst=10;
    # ... proxy config
}
```

---

## Monitoring and Alerting

### Essential Monitoring Stack

Deploy monitoring infrastructure before deploying OpenClaw:

#### Docker Compose Monitoring Stack

```yaml
# monitoring-stack.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    ports:
      - "9093:9093"
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
  alertmanager-data:
```

#### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alerts.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'openclaw-gateway'
    static_configs:
      - targets: ['openclaw-gateway:18789']
    metrics_path: '/metrics'  # If OpenClaw exposes metrics
```

#### Alert Rules

```yaml
# alerts.yml
groups:
  - name: openclaw_alerts
    interval: 30s
    rules:
      # High memory usage
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 90% for 5 minutes"

      # High CPU usage
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for 5 minutes"

      # Disk space warning
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space"
          description: "Disk space is below 20%"

      # Gateway down
      - alert: GatewayDown
        expr: up{job="openclaw-gateway"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "OpenClaw gateway is down"
          description: "Gateway has been down for 1 minute"
```

#### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'default'
    email_configs:
      - to: 'ops@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'your-app-password'

  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#openclaw-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
```

---

## Audit Logging

### Implement Comprehensive Logging

Create a centralized logging configuration:

```yaml
# docker-compose.override.yml
version: '3.8'

services:
  openclaw-gateway:
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
        labels: "service=openclaw-gateway"
    environment:
      - LOG_LEVEL=info
      - AUDIT_LOG_FILE=/home/node/.openclaw/logs/audit.log
```

Create log rotation configuration:

```bash
# /etc/logrotate.d/openclaw
/var/log/openclaw/*.log {
    daily
    rotate 90
    compress
    delaycompress
    notifempty
    create 0640 openclaw openclaw
    sharedscripts
    postrotate
        systemctl reload openclaw-gateway >/dev/null 2>&1 || true
    endscript
}
```

### Monitor Security Events

Create a script to monitor for suspicious activity:

```python
#!/usr/bin/env python3
# security-monitor.py

import re
import time
from datetime import datetime
from collections import defaultdict

LOG_FILE = '/var/log/openclaw/audit.log'
ALERT_THRESHOLD = 10  # Failed auth attempts before alert
TIME_WINDOW = 300  # 5 minutes

failed_attempts = defaultdict(list)

def send_security_alert(ip, attempts):
    # Send to SIEM or alerting system
    print(f"SECURITY ALERT: {ip} had {attempts} failed authentication attempts")

def monitor_logs():
    with open(LOG_FILE, 'r') as f:
        # Seek to end
        f.seek(0, 2)
        
        while True:
            line = f.readline()
            if not line:
                time.sleep(0.1)
                continue
            
            # Check for failed authentication
            if 'authentication failed' in line.lower():
                match = re.search(r'ip=(\d+\.\d+\.\d+\.\d+)', line)
                if match:
                    ip = match.group(1)
                    now = time.time()
                    
                    # Record attempt
                    failed_attempts[ip].append(now)
                    
                    # Clean old attempts
                    failed_attempts[ip] = [
                        t for t in failed_attempts[ip]
                        if now - t < TIME_WINDOW
                    ]
                    
                    # Check threshold
                    if len(failed_attempts[ip]) >= ALERT_THRESHOLD:
                        send_security_alert(ip, len(failed_attempts[ip]))

if __name__ == '__main__':
    monitor_logs()
```

---

## Backup and Disaster Recovery

### Automated Backup Strategy

```bash
#!/bin/bash
# backup-openclaw.sh

BACKUP_DIR="/backup/openclaw"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="openclaw_${DATE}.tar.gz"

# Create backup directory
mkdir -p "${BACKUP_DIR}"

# Backup configuration and data
tar -czf "${BACKUP_DIR}/${BACKUP_NAME}" \
    ~/.openclaw/config/ \
    ~/.openclaw/sessions/ \
    ~/.openclaw/logs/ \
    ~/.openclaw/credentials/ \
    /etc/openclaw/ \
    --exclude='*.tmp' \
    --exclude='*.log'

# Encrypt backup
gpg --symmetric --cipher-algo AES256 "${BACKUP_DIR}/${BACKUP_NAME}"
rm "${BACKUP_DIR}/${BACKUP_NAME}"

# Upload to secure remote storage
aws s3 cp "${BACKUP_DIR}/${BACKUP_NAME}.gpg" \
    s3://your-backup-bucket/openclaw/ \
    --storage-class GLACIER

# Keep only last 30 days locally
find "${BACKUP_DIR}" -name "openclaw_*.tar.gz.gpg" -mtime +30 -delete

echo "Backup completed: ${BACKUP_NAME}.gpg"
```

Schedule backups:

```cron
# Daily backups at 2 AM
0 2 * * * /usr/local/bin/backup-openclaw.sh
```

### Disaster Recovery Procedure

Document and test recovery procedures:

```bash
#!/bin/bash
# restore-openclaw.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.tar.gz.gpg>"
    exit 1
fi

# Decrypt backup
gpg --decrypt "${BACKUP_FILE}" > /tmp/openclaw_restore.tar.gz

# Stop services
docker-compose down

# Restore files
tar -xzf /tmp/openclaw_restore.tar.gz -C /

# Restore permissions
sudo chown -R openclaw:openclaw ~/.openclaw
sudo chmod 700 ~/.openclaw/credentials
sudo chmod 600 ~/.openclaw/credentials/*

# Start services
docker-compose up -d

# Verify
sleep 10
docker-compose ps
curl -f https://openclaw.yourdomain.com/health || echo "Health check failed"

# Cleanup
rm /tmp/openclaw_restore.tar.gz

echo "Restore completed from ${BACKUP_FILE}"
```

---

## Production Deployment Procedure

### Step-by-Step Deployment

1. **Prepare Environment**

```bash
# Create openclaw user
sudo useradd -r -s /bin/bash -d /opt/openclaw openclaw

# Create directories
sudo mkdir -p /opt/openclaw/{config,data,logs,backups}
sudo chown -R openclaw:openclaw /opt/openclaw

# Set up environment
sudo -u openclaw cat > /opt/openclaw/.env << 'EOF'
OPENCLAW_GATEWAY_TOKEN=$(openssl rand -base64 32)
ANTHROPIC_API_KEY=your_key_here
OPENAI_API_KEY=your_key_here
GRAFANA_PASSWORD=$(openssl rand -base64 24)
EOF

sudo chmod 600 /opt/openclaw/.env
```

2. **Deploy Monitoring Stack**

```bash
cd /opt/openclaw
sudo -u openclaw docker-compose -f monitoring-stack.yml up -d

# Verify monitoring
curl http://localhost:9090/-/healthy  # Prometheus
curl http://localhost:3000/api/health  # Grafana
```

3. **Deploy Reverse Proxy**

```bash
# Deploy Caddy
sudo -u openclaw docker-compose -f caddy-stack.yml up -d

# Verify TLS
curl -I https://openclaw.yourdomain.com
```

4. **Deploy OpenClaw Gateway**

```bash
# Pull latest image
sudo -u openclaw docker pull openclaw:latest

# Start gateway
sudo -u openclaw docker-compose up -d openclaw-gateway

# Check logs
sudo -u openclaw docker-compose logs -f openclaw-gateway
```

5. **Verify Deployment**

```bash
# Health check
curl -f https://openclaw.yourdomain.com/health

# Test authentication
curl -H "Authorization: Bearer ${OPENCLAW_GATEWAY_TOKEN}" \
    https://openclaw.yourdomain.com/api/status

# Check monitoring
curl http://localhost:9090/api/v1/query?query=up{job=\"openclaw-gateway\"}
```

6. **Set Up Monitoring Dashboards**

- Access Grafana at `http://localhost:3000`
- Import OpenClaw dashboard (create custom)
- Configure alert channels
- Test alerting

7. **Run Security Validation**

```bash
# Verify no plaintext credentials in logs
sudo grep -ri "api.key\|password\|secret" /var/log/openclaw/ || echo "Good: No secrets in logs"

# Verify TLS configuration
nmap --script ssl-enum-ciphers -p 443 openclaw.yourdomain.com

# Verify firewall
sudo iptables -L -v -n
```

---

## Ongoing Maintenance

### Daily Checks

```bash
#!/bin/bash
# daily-health-check.sh

echo "=== OpenClaw Daily Health Check ==="
echo "Date: $(date)"

# Check service status
echo -e "\n1. Service Status:"
docker-compose ps

# Check disk space
echo -e "\n2. Disk Space:"
df -h | grep -E "Filesystem|/opt/openclaw"

# Check recent errors
echo -e "\n3. Recent Errors (last 24h):"
docker-compose logs --since 24h | grep -i error | tail -20

# Check costs (if monitoring script exists)
echo -e "\n4. API Costs (MTD):"
python3 /usr/local/bin/monitor-api-costs.py

# Check certificate expiry
echo -e "\n5. TLS Certificate:"
echo | openssl s_client -connect openclaw.yourdomain.com:443 2>/dev/null | \
    openssl x509 -noout -dates

echo -e "\n=== Health Check Complete ==="
```

### Weekly Tasks

- Review audit logs for suspicious activity
- Check backup integrity (test restore to staging)
- Review and rotate credentials
- Update security patches
- Review monitoring alerts and adjust thresholds

### Monthly Tasks

- Conduct disaster recovery drill
- Review and update documentation
- Security scan with updated vulnerability databases
- Review and optimize costs
- Update dependencies and base images

---

## Security Incident Response

### Incident Response Runbook

**1. Suspected Compromise**

```bash
# Immediate actions
# 1. Isolate system
sudo iptables -A INPUT -j DROP
docker-compose down

# 2. Preserve evidence
tar -czf /tmp/forensics_$(date +%s).tar.gz \
    /var/log/openclaw/ \
    ~/.openclaw/sessions/ \
    ~/.openclaw/logs/

# 3. Revoke all credentials
# - Revoke all API keys in provider dashboards
# - Change gateway token
# - Rotate all passwords

# 4. Analyze logs
grep -E "authentication|error|failed" /var/log/openclaw/* | \
    tee /tmp/incident_analysis.log

# 5. Contact security team
```

**2. Cost Spike Alert**

```bash
# 1. Check current usage
python3 /usr/local/bin/monitor-api-costs.py

# 2. Review recent API calls
docker-compose logs openclaw-gateway | grep "api_call" | tail -1000

# 3. Identify source
# - Which user/session?
# - Which model?
# - What prompts?

# 4. Take action
# - Temporarily disable high-cost models
# - Implement emergency rate limiting
# - Contact provider to pause API access if needed

# 5. Root cause analysis
# - Bug in code?
# - Infinite loop?
# - Abuse?
```

---

## Conclusion

This deployment guide provides practical steps to mitigate OpenClaw's security and operational gaps. However, these are compensating controls—they reduce risk but cannot eliminate it entirely.

**Key Reminders:**

- Full-disk encryption is mandatory, not optional
- Never expose gateway directly to internet
- Monitor costs vigilantly
- Maintain current backups
- Test disaster recovery regularly
- Stay current with security patches
- Have incident response procedures ready

**When NOT to Deploy:**

Even with these mitigations, do NOT deploy OpenClaw if:
- Processing healthcare data (HIPAA)
- Handling payment information (PCI DSS)
- Subject to SOC 2 audit requirements
- Processing EU citizen data requiring GDPR compliance
- Any regulated industry or compliance requirement

For these use cases, wait for fundamental architectural improvements addressing credential encryption, comprehensive audit logging, and compliance framework support.

**Support Resources:**

- OpenClaw Documentation: https://docs.openclaw.ai
- Security Issues: Refer to SECURITY.md in repository
- Community Support: Discord/GitHub discussions

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Maintainer:** DevOps Security Team  
**Review Schedule:** Monthly or after major version updates
