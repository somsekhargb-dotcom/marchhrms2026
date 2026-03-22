# FCT Energy HRMS - Local / Internal Domain Deployment Guide

This guide covers deploying the HRMS on your own server (physical, VM, or cloud) under any internal or custom domain. No Replit dependency — everything runs standalone.

---

## Prerequisites

- **Server**: Ubuntu 20.04+ / CentOS 8+ / Any Linux with systemd (Windows also supported)
- **Node.js** v20 or higher
- **PostgreSQL** v14 or higher (local or remote like AWS RDS)
- **npm** (comes with Node.js)
- **Nginx** (for reverse proxy with custom domain)
- **(Optional)** PM2 for process management

---

## Step 1: Transfer the Project

Copy the project files to your server:

```bash
# Option A: From Git
git clone <your-repo-url>
cd <project-folder>

# Option B: From ZIP
scp hrms-project.zip user@your-server:/opt/hrms/
ssh user@your-server
cd /opt/hrms && unzip hrms-project.zip
```

---

## Step 2: Install Dependencies

```bash
npm install
```

---

## Step 3: Set Up PostgreSQL Database

### Option A: Local PostgreSQL

```bash
sudo -u postgres psql
CREATE DATABASE hrms;
CREATE USER hrmsuser WITH PASSWORD 'your-strong-password';
GRANT ALL PRIVILEGES ON DATABASE hrms TO hrmsuser;
\q
```

### Option B: AWS RDS / Remote PostgreSQL

Use your RDS endpoint directly in the `DATABASE_URL` (see Step 4).

---

## Step 4: Configure Environment Variables

Create a `.env` file in the project root:

```env
# Database Connection
DATABASE_URL=postgresql://hrmsuser:your-strong-password@localhost:5432/hrms
# For AWS RDS:
# DATABASE_URL=postgresql://postgres:password@your-rds-endpoint.rds.amazonaws.com:5432/postgres

# Application
PORT=5000
NODE_ENV=production

# Your Domain URL (used for password reset emails, links, etc.)
APP_URL=https://hrms.yourcompany.com

# Session Secret (generate one using the command below)
SESSION_SECRET=generate-a-random-64-char-string

# SMTP Email Configuration
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
SMTP_FROM=your-email@gmail.com
```

### Generate a Session Secret

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### Gmail App Password Setup

1. Go to Google Account > Security > 2-Step Verification
2. At the bottom, click "App passwords"
3. Generate a new app password for "Mail"
4. Use that as `SMTP_PASS`

### For Internal SMTP Server

```env
SMTP_HOST=mail.yourcompany.com
SMTP_PORT=25
SMTP_USER=hrms@yourcompany.com
SMTP_PASS=your-password
SMTP_FROM=hrms@yourcompany.com
```

---

## Step 5: Build the Application

```bash
NODE_OPTIONS='--max-old-space-size=2048' npm run build
```

This creates:
- `dist/index.cjs` — Server bundle (backend + serves frontend)
- `dist/public/` — Frontend static assets

---

## Step 6: Start and Verify

```bash
NODE_ENV=production node dist/index.cjs
```

Open `http://your-server-ip:5000` in a browser. The database tables are auto-created on first startup.

Press `Ctrl+C` to stop after verifying.

---

## Step 7: Process Manager (PM2)

Install PM2 to keep the app running in the background and auto-restart on crashes or reboots:

```bash
# Install PM2
npm install -g pm2

# Create ecosystem config
cat > ecosystem.config.cjs << 'EOF'
module.exports = {
  apps: [{
    name: "hrms",
    script: "dist/index.cjs",
    env: {
      NODE_ENV: "production",
      PORT: "5000"
    },
    instances: 1,
    max_memory_restart: "512M",
    log_date_format: "YYYY-MM-DD HH:mm:ss",
    error_file: "/var/log/hrms/error.log",
    out_file: "/var/log/hrms/output.log",
    merge_logs: true
  }]
};
EOF

# Create log directory
sudo mkdir -p /var/log/hrms
sudo chown $USER:$USER /var/log/hrms

# Start the application
pm2 start ecosystem.config.cjs

# Auto-start on server reboot
pm2 startup
pm2 save

# Useful commands
pm2 status          # Check status
pm2 logs hrms       # View logs
pm2 restart hrms    # Restart
pm2 stop hrms       # Stop
pm2 monit           # Monitor CPU/Memory
```

---

## Step 8: Nginx Reverse Proxy (Custom Domain)

### Install Nginx

```bash
sudo apt update && sudo apt install nginx -y
```

### Configure Site

```bash
sudo nano /etc/nginx/sites-available/hrms
```

Paste the following (replace `hrms.yourcompany.com` with your domain):

```nginx
server {
    listen 80;
    server_name hrms.yourcompany.com;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
    }
}
```

Enable and restart:

```bash
sudo ln -s /etc/nginx/sites-available/hrms /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### DNS Setup

Point your domain to the server IP in your DNS settings (or internal DNS):

```
hrms.yourcompany.com  →  A record  →  192.168.x.x (your server IP)
```

For internal-only access, add to your internal DNS or each client machine's hosts file:

```
192.168.x.x  hrms.yourcompany.com
```

---

## Step 9: SSL Certificate

### Option A: Let's Encrypt (Public Domain)

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d hrms.yourcompany.com
```

### Option B: Self-Signed Certificate (Internal Domain)

```bash
sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/hrms.key \
  -out /etc/nginx/ssl/hrms.crt \
  -subj "/CN=hrms.yourcompany.com/O=YourCompany/C=IN"
```

Update Nginx config to use SSL:

```nginx
server {
    listen 80;
    server_name hrms.yourcompany.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name hrms.yourcompany.com;

    ssl_certificate /etc/nginx/ssl/hrms.crt;
    ssl_certificate_key /etc/nginx/ssl/hrms.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 300s;
    }
}
```

### Option C: Company CA Certificate

If your company has its own Certificate Authority:

```bash
# Place your cert files
sudo cp company-hrms.crt /etc/nginx/ssl/hrms.crt
sudo cp company-hrms.key /etc/nginx/ssl/hrms.key
```

Use the same SSL Nginx config as Option B above.

---

## Step 10: Firewall Configuration

```bash
# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# If accessing PostgreSQL remotely
sudo ufw allow from 192.168.0.0/16 to any port 5432

# Enable firewall
sudo ufw enable
```

---

## Alternative: systemd Service (Instead of PM2)

Create `/etc/systemd/system/hrms.service`:

```ini
[Unit]
Description=FCT Energy HRMS
After=network.target postgresql.service

[Service]
Type=simple
User=hrms
Group=hrms
WorkingDirectory=/opt/hrms
EnvironmentFile=/opt/hrms/.env
ExecStart=/usr/bin/node dist/index.cjs
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
# Create service user
sudo useradd -r -s /bin/false hrms
sudo chown -R hrms:hrms /opt/hrms

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable hrms
sudo systemctl start hrms

# Check status and logs
sudo systemctl status hrms
sudo journalctl -u hrms -f
```

---

## Database Backup

### Automated Daily Backups

Create `/opt/hrms/backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/opt/hrms/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# For local PostgreSQL
pg_dump -U hrmsuser hrms > "$BACKUP_DIR/hrms_$TIMESTAMP.sql"

# For AWS RDS
# PGPASSWORD=your-password pg_dump -h your-rds-endpoint -U postgres postgres > "$BACKUP_DIR/hrms_$TIMESTAMP.sql"

# Keep only last 30 backups
ls -t $BACKUP_DIR/hrms_*.sql | tail -n +31 | xargs -r rm

echo "Backup completed: hrms_$TIMESTAMP.sql"
```

```bash
chmod +x /opt/hrms/backup.sh

# Add to crontab (daily at 2 AM)
crontab -e
# Add this line:
0 2 * * * /opt/hrms/backup.sh >> /var/log/hrms/backup.log 2>&1
```

### Restore from Backup

```bash
psql -U hrmsuser hrms < /opt/hrms/backups/hrms_20260322_020000.sql
```

---

## Updating the Application

```bash
cd /opt/hrms

# Pull latest code
git pull origin main

# Install new dependencies
npm install

# Rebuild
NODE_OPTIONS='--max-old-space-size=2048' npm run build

# Restart
pm2 restart hrms
# Or: sudo systemctl restart hrms
```

---

## First-Time Admin Setup

1. Open `https://hrms.yourcompany.com` in your browser
2. Register the first admin account with email and password
3. Go to Entity Management and create entities (e.g., FCTEnergy, NonFCTEnergy)
4. Add departments, then start adding employees
5. Each employee gets their email/password login credentials

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `PORT` | No | Server port (default: 5000) |
| `NODE_ENV` | Yes | Set to `production` |
| `APP_URL` | Yes | Your full domain URL (e.g., `https://hrms.yourcompany.com`) |
| `SESSION_SECRET` | Yes | Random string for session encryption |
| `SMTP_HOST` | Yes | SMTP server hostname |
| `SMTP_PORT` | Yes | SMTP port (587 for TLS, 25 for plain) |
| `SMTP_USER` | Yes | SMTP username |
| `SMTP_PASS` | Yes | SMTP password |
| `SMTP_FROM` | Yes | From email address |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| App won't start | Check `NODE_ENV=production` is set, verify `DATABASE_URL` |
| Database connection refused | Ensure PostgreSQL is running, check credentials |
| Password reset links wrong URL | Set `APP_URL` in `.env` to your domain |
| SMTP email failures | Verify SMTP credentials, check firewall allows outbound port 587/25 |
| Nginx 502 Bad Gateway | App not running — check `pm2 status` or `systemctl status hrms` |
| Build out of memory | Use `NODE_OPTIONS='--max-old-space-size=2048'` |
| Session cookie not working | Ensure `SESSION_SECRET` is set and Nginx forwards headers correctly |
| SSL issues on internal domain | Use self-signed cert or company CA cert (see Step 9) |
| Tables not created | Auto-created on first startup — check server logs for SQL errors |
