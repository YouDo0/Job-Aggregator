# VPS Deployment Guide

This guide walks you through deploying the Job Aggregator & Notifier on a VPS (Ubuntu 22.04+).

## Prerequisites

- VPS with Ubuntu 22.04+ (2GB RAM minimum recommended)
- Domain name pointed to VPS IP (optional, for production email)
- Docker and Docker Compose installed on VPS

---

## Step 1: SSH into Your VPS

```bash
ssh root@your-vps-ip
```

---

## Step 2: Install Docker

```bash
# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh

# Install Docker Compose
apt install docker-compose -y

# Add current user to docker group
usermod -aG docker $USER
```

---

## Step 3: Create Project Directory

```bash
mkdir -p /opt/job-aggregator
cd /opt/job-aggregator
```

---

## Step 4: Upload Project Files

**Option A: Using Git**
```bash
git clone https://github.com/yourusername/job-aggregator.git .
```

**Option B: Using SCP (from your local machine)**
```bash
scp -r ./job_aggregator/* root@your-vps-ip:/opt/job-aggregator/
```

---

## Step 5: Configure Environment Variables

```bash
cd /opt/job-aggregator
cp .env.example .env
nano .env
```

Edit these values:

```env
# Database (keep the Docker Compose defaults)
DATABASE_URL=postgresql://jobuser:jobpass@db:5432/jobtracker
POSTGRES_USER=jobuser
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=jobtracker

# Redis (keep the Docker Compose default)
REDIS_URL=redis://redis:6379/0

# Email (REQUIRED - for sending digest emails)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your_email@gmail.com
EMAIL_PASSWORD=your_gmail_app_password
EMAIL_FROM=your_email@gmail.com

# Digest Schedule
DIGEST_SCHEDULE={"11:00": "Asia/Jakarta", "14:30": "Asia/Jakarta"}
```

### Gmail App Password Setup

1. Enable 2-Factor Authentication on your Google Account
2. Go to https://myaccount.google.com/apppasswords
3. Generate a new App Password for "Mail"
4. Use that 16-character password in `EMAIL_PASSWORD`

---

## Step 6: Update Docker Compose (Optional)

Edit `docker-compose.yml` if you want to change ports or resource limits:

```yaml
services:
  web:
    ports:
      - "80:5000"  # Change to port 80 for HTTP
    restart: unless-stopped

  celery_worker:
    restart: unless-stopped

  redis:
    restart: unless-stopped

  db:
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

---

## Step 7: Start the Application

```bash
# Build and start all services
docker-compose up -d --build

# Check status
docker-compose ps

# View logs
docker-compose logs -f web
docker-compose logs -f celery_worker
```

---

## Step 8: Initialize Database

```bash
# Run inside the web container
docker-compose exec web python -c "from app import create_app, db; app = create_app(); app.app_context().push(); db.create_all()"
```

Or use this one-liner:

```bash
docker-compose exec web flask db upgrade 2>/dev/null || docker-compose exec web python -c "from app import create_app, db; app = create_app(); app.app_context().push(); db.create_all()"
```

---

## Step 9: Verify Services

```bash
# Check all containers are running
docker-compose ps

# Check web is responding
curl http://localhost:5000

# Check celery worker is running
docker-compose exec celery_worker celery -A app.tasks.celery_app inspect stats
```

---

## Step 10: Set Up Systemd Service (Recommended)

Create a systemd service for auto-restart:

```bash
nano /etc/systemd/system/job-aggregator.service
```

Paste this content:

```ini
[Unit]
Description=Job Aggregator Web Service
Requires=docker-compose-app.service
After=docker-compose-app.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/job-aggregator
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Create the docker-compose service:

```bash
nano /etc/systemd/system/docker-compose-app.service
```

```ini
[Unit]
Description=Job Aggregator Application
Requires=docker.service
After=docker.service

[Service]
WorkingDirectory=/opt/job-aggregator
ExecStart=/usr/bin/docker-compose up
ExecStop=/usr/bin/docker-compose down
Restart=on-failure
RestartSec=10s
User=root

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reload
systemctl enable job-aggregator.service
systemctl start job-aggregator.service
systemctl status job-aggregator.service
```

---

## Step 11: Set Up Nginx Reverse Proxy (Optional)

For production with HTTPS:

```bash
apt install nginx -y
nano /etc/nginx/sites-available/job-aggregator
```

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/job-aggregator /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

---

## Step 12: Set Up Cron for Digest Email

The Celery beat scheduler handles this automatically, but verify:

```bash
# Check if celery beat is running (it's embedded in the worker with --beat flag)
docker-compose logs celery_worker | grep beat
```

If beat isn't running, update `docker-compose.yml`:

```yaml
celery_worker:
  build: .
  command: celery -A app.tasks.celery_app worker --loglevel=info --beat
```

Then restart:

```bash
docker-compose down
docker-compose up -d
```

---

## Troubleshooting

### Check Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f web
docker-compose logs -f celery_worker
docker-compose logs -f db
```

### Restart Services

```bash
docker-compose restart web
docker-compose restart celery_worker
```

### Database Migration Issues

```bash
# Drop and recreate tables
docker-compose exec web python -c "from app import create_app, db; app = create_app(); app.app_context().push(); db.drop_all(); db.create_all()"
```

### Celery Worker Not Processing Tasks

```bash
# Check worker is connected
docker-compose exec celery_worker celery -A app.tasks.celery_app inspect active

# Restart worker
docker-compose restart celery_worker
```

### Memory Issues

If VPS has low RAM, add swap:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

---

## Updating the Application

```bash
cd /opt/job-aggregator

# Pull latest code
git pull

# Rebuild and restart
docker-compose up -d --build

# View updates
docker-compose logs -f
```

---

## Security Checklist

- [ ] Change default `POSTGRES_PASSWORD`
- [ ] Use Gmail App Password (not your real password)
- [ ] Enable firewall: `ufw allow 80/tcp && ufw allow 443/tcp`
- [ ] Set `DEBUG=False` in production (add to .env)
- [ ]定期备份数据库:

```bash
# Backup
docker-compose exec db pg_dump -U jobuser jobtracker > backup_$(date +%Y%m%d).sql

# Restore
cat backup_file.sql | docker-compose exec -T db psql -U jobuser jobtracker
```
