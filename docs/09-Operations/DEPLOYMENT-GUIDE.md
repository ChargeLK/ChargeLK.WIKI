# ChargeLK MVP Deployment Guide - Hostinger VPS

## Overview

This guide will help you deploy:
- ✅ ChargeLK.UM Service (User Management) - Port 5002
- ✅ ChargeLK.OCPP Service - Port 5001  
- ✅ ChargeLK Admin Portal (React UI) - Port 80/443
- ✅ PostgreSQL Database (both services)
- ✅ Nginx Reverse Proxy
- ✅ SSL/TLS with Let's Encrypt

## Prerequisites

- ✅ Hostinger VPS with Ubuntu 20.04/22.04
- ✅ Root password
- ✅ Server IP address
- ✅ Domain name (optional but recommended)

## Architecture

```
Internet
    ↓
Nginx (Port 80/443) - SSL Termination
    ↓
├─→ Admin UI (Port 3000) - Static Files
├─→ UM Service (Port 5002) - /api/um/*
└─→ OCPP Service (Port 5001) - /api/ocpp/*, WebSocket /ocpp/*
    ↓
PostgreSQL (Port 5432)
├─→ chargelk_um database
└─→ chargelk_ocpp database
```

## Deployment Steps

### Step 1: Connect to Server

```bash
# From your local machine
ssh root@YOUR_SERVER_IP

# Enter your root password when prompted
```

### Step 2: Initial Server Setup

```bash
# Update system
apt update && apt upgrade -y

# Install essential tools
apt install -y git curl wget vim sudo ufw

# Create a deployment user (recommended)
adduser chargelk
usermod -aG sudo chargelk
su - chargelk
```

### Step 3: Install Required Software

#### Install .NET 9 SDK
```bash
# Add Microsoft package repository
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# Install .NET SDK
sudo apt update
sudo apt install -y dotnet-sdk-9.0

# Verify installation
dotnet --version
```

#### Install Entity Framework Core Tools
```bash
# Install dotnet-ef globally
dotnet tool install --global dotnet-ef

# Add .NET tools to PATH
echo 'export PATH="$PATH:$HOME/.dotnet/tools"' >> ~/.bashrc
source ~/.bashrc

# Verify installation
dotnet ef --version

# If running scripts with sudo, also install for root
sudo -i
dotnet tool install --global dotnet-ef
echo 'export PATH="$PATH:/root/.dotnet/tools"' >> ~/.bashrc
exit
```

#### Install Node.js 20 (for Admin UI)
```bash
# Install Node.js 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version
npm --version
```

#### Install PostgreSQL
```bash
# Install PostgreSQL 15
sudo apt install -y postgresql postgresql-contrib

# Start and enable PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Verify installation
sudo systemctl status postgresql
```

#### Install Nginx
```bash
# Install Nginx
sudo apt install -y nginx

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Step 4: Configure Firewall

```bash
# Configure UFW
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw allow 5001/tcp  # OCPP Service
sudo ufw allow 5002/tcp  # UM Service
sudo ufw enable

# Check status
sudo ufw status
```

### Step 5: Setup PostgreSQL Databases

```bash
# Switch to postgres user
sudo -i -u postgres

# Create databases and user
psql << EOF
-- Create user
CREATE USER chargelk WITH PASSWORD 'YOUR_STRONG_PASSWORD_HERE';

-- Create databases
CREATE DATABASE chargelk_um OWNER chargelk;
CREATE DATABASE chargelk_ocpp OWNER chargelk;

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE chargelk_um TO chargelk;
GRANT ALL PRIVILEGES ON DATABASE chargelk_ocpp TO chargelk;

-- Exit
\q
EOF

# Exit postgres user
exit

# Test connection
psql -U chargelk -d chargelk_um -h localhost -W
# Enter password when prompted, then \q to exit
```

### Step 6: Clone and Deploy Application

```bash
# Create application directory
sudo mkdir -p /var/www/chargelk
sudo chown -R chargelk:chargelk /var/www/chargelk
cd /var/www/chargelk

# Clone your repository (use your actual repo URL)
git clone https://github.com/your-username/ChargeLK.git .
# OR upload files via SCP (see below)

# If you don't have a git repo, upload from local machine:
# From your LOCAL machine (in ChargeLK directory):
# scp -r ChargeLK.OCPP ChargeLK.UM ChargeLK.AdminUI chargelk@YOUR_SERVER_IP:/var/www/chargelk/
```

### Step 7: Configure Application Settings

#### UM Service Configuration
```bash
cd /var/www/chargelk/ChargeLK.UM/ChargeLK.API

# Create production settings
cat > appsettings.Production.json << 'EOF'
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=chargelk_um;Username=chargelk;Password=YOUR_STRONG_PASSWORD_HERE"
  },
  "Jwt": {
    "SecretKey": "GENERATE_A_STRONG_64_CHARACTER_SECRET_KEY_HERE_USE_openssl_rand_base64_64",
    "Issuer": "chargelk-um-service",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  },
  "AllowedOrigins": [
    "http://YOUR_DOMAIN.com",
    "https://YOUR_DOMAIN.com",
    "http://YOUR_SERVER_IP"
  ],
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information"
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "/var/log/chargelk/um-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      }
    ]
  }
}
EOF

# Create log directory
sudo mkdir -p /var/log/chargelk
sudo chown -R chargelk:chargelk /var/log/chargelk
```

#### OCPP Service Configuration
```bash
cd /var/www/chargelk/ChargeLK.OCPP/ChargeLK.API

# Create production settings
cat > appsettings.Production.json << 'EOF'
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=chargelk_ocpp;Username=chargelk;Password=YOUR_STRONG_PASSWORD_HERE"
  },
  "Jwt": {
    "SecretKey": "SAME_SECRET_KEY_AS_UM_SERVICE",
    "Issuer": "chargelk-um-service",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  },
  "AllowedOrigins": [
    "http://YOUR_DOMAIN.com",
    "https://YOUR_DOMAIN.com",
    "http://YOUR_SERVER_IP"
  ],
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information"
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "/var/log/chargelk/ocpp-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      }
    ]
  }
}
EOF
```

**IMPORTANT:** Replace placeholders:
- `YOUR_STRONG_PASSWORD_HERE` - Your PostgreSQL password
- `GENERATE_A_STRONG_64_CHARACTER_SECRET_KEY_HERE` - Generate with: `openssl rand -base64 64`
- `YOUR_DOMAIN.com` - Your actual domain (or remove if using IP only)
- `YOUR_SERVER_IP` - Your server IP address

### Step 8: Build and Publish Applications

#### Build UM Service
```bash
cd /var/www/chargelk/ChargeLK.UM/ChargeLK.API

# Restore dependencies
dotnet restore

# Run migrations
dotnet ef database update --environment Production

# Build for production
dotnet publish -c Release -o /var/www/chargelk/publish/um

# Verify
ls -la /var/www/chargelk/publish/um
```

#### Build OCPP Service
```bash
cd /var/www/chargelk/ChargeLK.OCPP/ChargeLK.API

# Restore dependencies
dotnet restore

# Run migrations
dotnet ef database update --environment Production

# Build for production
dotnet publish -c Release -o /var/www/chargelk/publish/ocpp

# Verify
ls -la /var/www/chargelk/publish/ocpp
```

#### Build Admin UI
```bash
cd /var/www/chargelk/ChargeLK.AdminUI

# Install dependencies
npm install

# Create production environment file
cat > .env.production << 'EOF'
VITE_API_BASE_URL=https://YOUR_DOMAIN.com/api
VITE_UM_API_URL=https://YOUR_DOMAIN.com/api/um
VITE_OCPP_API_URL=https://YOUR_DOMAIN.com/api/ocpp
VITE_WS_URL=wss://YOUR_DOMAIN.com/ocpp
EOF

# Build for production
npm run build

# Move build to web root
sudo mkdir -p /var/www/chargelk/adminui
sudo cp -r dist/* /var/www/chargelk/adminui/
sudo chown -R www-data:www-data /var/www/chargelk/adminui
```

### Step 9: Create Systemd Services

#### UM Service
```bash
sudo tee /etc/systemd/system/chargelk-um.service << 'EOF'
[Unit]
Description=ChargeLK User Management Service
After=network.target postgresql.service

[Service]
Type=notify
User=chargelk
WorkingDirectory=/var/www/chargelk/publish/um
ExecStart=/usr/bin/dotnet /var/www/chargelk/publish/um/ChargeLK.API.dll
Restart=always
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=chargelk-um
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
Environment=ASPNETCORE_URLS=http://localhost:5002

[Install]
WantedBy=multi-user.target
EOF
```

#### OCPP Service
```bash
sudo tee /etc/systemd/system/chargelk-ocpp.service << 'EOF'
[Unit]
Description=ChargeLK OCPP Service
After=network.target postgresql.service

[Service]
Type=notify
User=chargelk
WorkingDirectory=/var/www/chargelk/publish/ocpp
ExecStart=/usr/bin/dotnet /var/www/chargelk/publish/ocpp/ChargeLK.API.dll
Restart=always
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=chargelk-ocpp
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
Environment=ASPNETCORE_URLS=http://localhost:5001

[Install]
WantedBy=multi-user.target
EOF
```

#### Enable and Start Services
```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable services to start on boot
sudo systemctl enable chargelk-um
sudo systemctl enable chargelk-ocpp

# Start services
sudo systemctl start chargelk-um
sudo systemctl start chargelk-ocpp

# Check status
sudo systemctl status chargelk-um
sudo systemctl status chargelk-ocpp

# Check logs
sudo journalctl -u chargelk-um -f
sudo journalctl -u chargelk-ocpp -f
```

### Step 10: Configure Nginx Reverse Proxy

```bash
# Create Nginx configuration
sudo tee /etc/nginx/sites-available/chargelk << 'EOF'
# Upstream definitions
upstream um_backend {
    server localhost:5002;
}

upstream ocpp_backend {
    server localhost:5001;
}

# HTTP server (redirect to HTTPS in production)
server {
    listen 80;
    server_name YOUR_DOMAIN.com www.YOUR_DOMAIN.com YOUR_SERVER_IP;

    # For Let's Encrypt verification
    location /.well-known/acme-challenge/ {
        root /var/www/chargelk/adminui;
    }

    # Redirect to HTTPS (comment out for initial testing)
    # return 301 https://$server_name$request_uri;

    # Admin UI
    location / {
        root /var/www/chargelk/adminui;
        try_files $uri $uri/ /index.html;
        add_header Cache-Control "no-cache";
    }

    # UM Service API
    location /api/um/ {
        proxy_pass http://um_backend/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # OCPP Service API
    location /api/ocpp/ {
        proxy_pass http://ocpp_backend/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # OCPP WebSocket
    location /ocpp/ {
        proxy_pass http://ocpp_backend/ocpp/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    # Health checks
    location /health {
        proxy_pass http://ocpp_backend/health;
        access_log off;
    }
}
EOF

# Enable the site
sudo ln -s /etc/nginx/sites-available/chargelk /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 11: Setup SSL with Let's Encrypt (Optional but Recommended)

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain SSL certificate (replace with your domain)
sudo certbot --nginx -d YOUR_DOMAIN.com -d www.YOUR_DOMAIN.com

# Certbot will automatically configure Nginx for HTTPS
# Test automatic renewal
sudo certbot renew --dry-run
```

### Step 12: Verify Deployment

```bash
# Check service status
sudo systemctl status chargelk-um
sudo systemctl status chargelk-ocpp
sudo systemctl status nginx
sudo systemctl status postgresql

# Test endpoints
curl http://localhost:5002/health
curl http://localhost:5001/health
curl http://YOUR_SERVER_IP/health

# Check logs
sudo journalctl -u chargelk-um --since "10 minutes ago"
sudo journalctl -u chargelk-ocpp --since "10 minutes ago"
sudo tail -f /var/log/chargelk/um-*.log
sudo tail -f /var/log/chargelk/ocpp-*.log
```

## Post-Deployment

### Access URLs

- **Admin Portal**: `http://YOUR_SERVER_IP` or `https://YOUR_DOMAIN.com`
- **UM API**: `http://YOUR_SERVER_IP/api/um` or `https://YOUR_DOMAIN.com/api/um`
- **OCPP API**: `http://YOUR_SERVER_IP/api/ocpp` or `https://YOUR_DOMAIN.com/api/ocpp`
- **OCPP WebSocket**: `ws://YOUR_SERVER_IP/ocpp/{chargeBoxId}` or `wss://YOUR_DOMAIN.com/ocpp/{chargeBoxId}`

### Default Login

- **Email**: `admin@chargelk.com`
- **Password**: `Admin@123`

**⚠️ IMPORTANT: Change this password immediately after first login!**

### Register Test RFID Tag

```bash
# SSH to server
ssh chargelk@YOUR_SERVER_IP

# Get token
TOKEN=$(curl -s -X POST http://localhost:5002/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@chargelk.com","password":"Admin@123"}' \
  | grep -o '"accessToken":"[^"]*' | sed 's/"accessToken":"//')

# Register tag
curl -X POST http://localhost:5001/api/OcppTags \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "idTag": "04B0267AE05C87",
    "note": "Test RFID tag",
    "expiryDate": "2026-12-31T23:59:59Z"
  }'
```

## Maintenance

### Update Application

```bash
# SSH to server
ssh chargelk@YOUR_SERVER_IP

# Pull latest code
cd /var/www/chargelk
git pull

# Rebuild UM Service
cd ChargeLK.UM/ChargeLK.API
dotnet publish -c Release -o /var/www/chargelk/publish/um
sudo systemctl restart chargelk-um

# Rebuild OCPP Service
cd /var/www/chargelk/ChargeLK.OCPP/ChargeLK.API
dotnet publish -c Release -o /var/www/chargelk/publish/ocpp
sudo systemctl restart chargelk-ocpp

# Rebuild Admin UI
cd /var/www/chargelk/ChargeLK.AdminUI
npm run build
sudo cp -r dist/* /var/www/chargelk/adminui/
```

### Backup Database

```bash
# Create backup script
cat > ~/backup-chargelk.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/var/backups/chargelk"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup databases
sudo -u postgres pg_dump chargelk_um > $BACKUP_DIR/chargelk_um_$DATE.sql
sudo -u postgres pg_dump chargelk_ocpp > $BACKUP_DIR/chargelk_ocpp_$DATE.sql

# Compress
tar -czf $BACKUP_DIR/chargelk_backup_$DATE.tar.gz $BACKUP_DIR/*.sql
rm $BACKUP_DIR/*.sql

# Keep only last 7 days
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: chargelk_backup_$DATE.tar.gz"
EOF

chmod +x ~/backup-chargelk.sh

# Add to crontab (daily at 2 AM)
(crontab -l 2>/dev/null; echo "0 2 * * * /home/chargelk/backup-chargelk.sh") | crontab -
```

### Monitor Services

```bash
# Real-time logs
sudo journalctl -u chargelk-um -u chargelk-ocpp -f

# Check resource usage
htop
df -h
free -h

# Check port usage
sudo netstat -tulpn | grep LISTEN
```

## Remote Database Access

### Access PostgreSQL from Local Machine

There are two main methods to access your server's PostgreSQL databases from your local development machine:

#### Method 1: SSH Tunnel (Recommended - More Secure)

This method creates an encrypted SSH tunnel, so you don't need to expose PostgreSQL to the internet.

**From Your Local Machine:**

```bash
# Create SSH tunnel
# This forwards local port 5433 to server's PostgreSQL port 5432
ssh -L 5433:localhost:5432 chargelk@YOUR_SERVER_IP -N

# Keep this terminal open while working
```

**Connect Using Database Client:**

```bash
# Using psql
psql -h localhost -p 5433 -U chargelk -d chargelk_um
psql -h localhost -p 5433 -U chargelk -d chargelk_ocpp

# Connection details for GUI tools (DBeaver, pgAdmin, DataGrip, etc.):
# Host: localhost
# Port: 5433
# Database: chargelk_um (or chargelk_ocpp)
# Username: chargelk
# Password: YOUR_STRONG_PASSWORD_HERE
```

**Connection String for Your Applications:**

```
Host=localhost;Port=5433;Database=chargelk_um;Username=chargelk;Password=YOUR_PASSWORD
```

**Create Convenient SSH Tunnel Script:**

```bash
# On your LOCAL machine, create: ~/tunnel-chargelk-db.sh
cat > ~/tunnel-chargelk-db.sh << 'EOF'
#!/bin/bash
echo "Creating SSH tunnel to ChargeLK PostgreSQL..."
echo "Local port 5433 -> Server port 5432"
echo "Press Ctrl+C to close the tunnel"
ssh -L 5433:localhost:5432 chargelk@YOUR_SERVER_IP -N
EOF

chmod +x ~/tunnel-chargelk-db.sh

# Usage:
./tunnel-chargelk-db.sh
```

#### Method 2: Direct Access (Less Secure - Use with Caution)

This method exposes PostgreSQL to the internet. Only use if necessary and ensure strong security measures.

**⚠️ WARNING:** Only use this method if you:
- Have a strong firewall configured
- Use strong passwords
- Restrict access to specific IP addresses
- Understand the security implications

**On Your Server:**

```bash
# 1. Configure PostgreSQL to accept remote connections
sudo vim /etc/postgresql/15/main/postgresql.conf

# Find and modify:
listen_addresses = '*'  # or 'localhost,YOUR_LOCAL_IP'

# 2. Configure client authentication
sudo vim /etc/postgresql/15/main/pg_hba.conf

# Add this line (replace YOUR_LOCAL_IP with your actual IP):
# For specific IP:
host    chargelk_um     chargelk        YOUR_LOCAL_IP/32        md5
host    chargelk_ocpp   chargelk        YOUR_LOCAL_IP/32        md5

# OR for any IP (NOT RECOMMENDED for production):
host    chargelk_um     chargelk        0.0.0.0/0               md5
host    chargelk_ocpp   chargelk        0.0.0.0/0               md5

# 3. Restart PostgreSQL
sudo systemctl restart postgresql

# 4. Open firewall port (restrict to your IP)
sudo ufw allow from YOUR_LOCAL_IP to any port 5432
# OR for any IP (NOT RECOMMENDED):
sudo ufw allow 5432/tcp

# Check firewall status
sudo ufw status
```

**From Your Local Machine:**

```bash
# Connect using psql
psql -h YOUR_SERVER_IP -p 5432 -U chargelk -d chargelk_um

# Connection details for GUI tools:
# Host: YOUR_SERVER_IP
# Port: 5432
# Database: chargelk_um (or chargelk_ocpp)
# Username: chargelk
# Password: YOUR_STRONG_PASSWORD_HERE
```

### Database Client Tools

#### Using pgAdmin (GUI)

1. **Download and Install pgAdmin**: [https://www.pgadmin.org/](https://www.pgadmin.org/)

2. **Create New Server Connection:**
   - Right-click "Servers" → "Create" → "Server"
   - **General Tab:**
     - Name: `ChargeLK UM` (or `ChargeLK OCPP`)
   - **Connection Tab:**
     - Host: `localhost` (if using SSH tunnel) or `YOUR_SERVER_IP` (direct)
     - Port: `5433` (if using SSH tunnel) or `5432` (direct)
     - Maintenance database: `chargelk_um`
     - Username: `chargelk`
     - Password: `YOUR_PASSWORD`
     - Save password: ✓

3. **Using SSH Tunnel with pgAdmin:**
   - First, start the SSH tunnel in a terminal
   - Then connect pgAdmin to `localhost:5433`

#### Using DBeaver (GUI)

1. **Download and Install DBeaver**: [https://dbeaver.io/](https://dbeaver.io/)

2. **Create New Connection:**
   - Database → New Database Connection → PostgreSQL
   - **Main Tab:**
     - Host: `localhost` (tunnel) or `YOUR_SERVER_IP` (direct)
     - Port: `5433` (tunnel) or `5432` (direct)
     - Database: `chargelk_um`
     - Username: `chargelk`
     - Password: `YOUR_PASSWORD`
   
3. **Test Connection** and Save

#### Using DataGrip (JetBrains)

**Method 1: Using SSH Tunnel (Recommended)**

1. **First, create SSH tunnel in terminal:**
   ```bash
   # On your LOCAL machine, open a terminal and run:
   ssh -L 5433:localhost:5432 chargelk@YOUR_SERVER_IP -N
   # Keep this terminal open!
   ```

2. **In DataGrip, create New Data Source:**
   - Click `+` → Data Source → PostgreSQL
   - **General Tab:**
     - Host: `localhost`
     - Port: `5433`
     - Database: `chargelk_um`
     - User: `chargelk`
     - Password: `YOUR_PASSWORD`
   - **SSH/SSL Tab:** Leave empty (tunnel is already active)
   - Click "Test Connection" - should succeed
   - Click "OK" or "Apply"

**Method 2: Using DataGrip's Built-in SSH Tunnel**

1. **Create New Data Source:**
   - Click `+` → Data Source → PostgreSQL
   - **General Tab:**
     - Host: `localhost`
     - Port: `5432` (remote server's PostgreSQL port)
     - Database: `chargelk_um`
     - User: `chargelk`
     - Password: `YOUR_PASSWORD`
   
2. **SSH/SSL Tab:**
   - Check "Use SSH tunnel"
   - Host: `YOUR_SERVER_IP`
   - Port: `22`
   - User: `chargelk`
   - Authentication type: Password (or Key pair)
   - Password: Your SSH password (or select private key file)
   
3. **Test Connection** and Apply

**Method 3: Direct Connection (Less Secure)**

Only use if PostgreSQL is configured for remote access (see "Method 2: Direct Access" above).

1. **Create New Data Source:**
   - Click `+` → Data Source → PostgreSQL
   - Host: `YOUR_SERVER_IP` (e.g., `168.231.120.50`)
   - Port: `5432`
   - Database: `chargelk_um`
   - User: `chargelk`
   - Password: `YOUR_PASSWORD`

2. **Test Connection** and Apply

**Troubleshooting DataGrip Connection:**

If you see "The connection attempt failed":

1. **Check SSH tunnel is running** (Method 1):
   ```bash
   # On your LOCAL machine:
   netstat -an | grep 5433
   # or
   lsof -i :5433
   # Should show a listening port
   ```

2. **Verify SSH access to server:**
   ```bash
   ssh chargelk@YOUR_SERVER_IP
   # If this fails, fix SSH access first
   ```

3. **Check if using correct port:**
   - Port 5433 = SSH tunnel method
   - Port 5432 = Direct connection (requires firewall rules)

4. **Download PostgreSQL driver in DataGrip:**
   - When creating connection, DataGrip will prompt to download drivers
   - Click "Download missing drivers"

5. **Check server PostgreSQL is running:**
   ```bash
   # SSH to server
   ssh chargelk@YOUR_SERVER_IP
   sudo systemctl status postgresql
   ```

6. **Test connection from command line first:**
   ```bash
   # If using tunnel (with tunnel running):
   psql -h localhost -p 5433 -U chargelk -d chargelk_um
   
   # If direct:
   psql -h YOUR_SERVER_IP -p 5432 -U chargelk -d chargelk_um
   ```

#### Using VS Code with PostgreSQL Extension

1. **Install Extension:** "PostgreSQL" by Chris Kolkman

2. **Add Connection:**
   - Click PostgreSQL icon in sidebar
   - Add Connection
   - Enter details:
     - Host: `localhost` or `YOUR_SERVER_IP`
     - Port: `5433` or `5432`
     - Database: `chargelk_um`
     - Username: `chargelk`
     - Password: `YOUR_PASSWORD`

### Quick Connection Test

```bash
# From your LOCAL machine (after setting up tunnel or direct access):

# Test UM database connection
psql -h localhost -p 5433 -U chargelk -d chargelk_um -c "SELECT version();"

# Test OCPP database connection
psql -h localhost -p 5433 -U chargelk -d chargelk_ocpp -c "SELECT version();"

# List all tables in UM database
psql -h localhost -p 5433 -U chargelk -d chargelk_um -c "\dt"

# List all tables in OCPP database
psql -h localhost -p 5433 -U chargelk -d chargelk_ocpp -c "\dt"
```

### Common Use Cases

#### Run Database Migrations from Local

```bash
# With SSH tunnel active on local machine:
cd /path/to/ChargeLK.UM/ChargeLK.API

# Update connection string temporarily or use environment variable
export ConnectionStrings__DefaultConnection="Host=localhost;Port=5433;Database=chargelk_um;Username=chargelk;Password=YOUR_PASSWORD"

# Run migrations
dotnet ef database update

# For OCPP:
cd /path/to/ChargeLK.OCPP/ChargeLK.API
export ConnectionStrings__DefaultConnection="Host=localhost;Port=5433;Database=chargelk_ocpp;Username=chargelk;Password=YOUR_PASSWORD"
dotnet ef database update
```

#### Query Data from Local

```bash
# With tunnel active
psql -h localhost -p 5433 -U chargelk -d chargelk_um

# Example queries:
SELECT * FROM "Users" LIMIT 10;
SELECT * FROM "Roles";
SELECT COUNT(*) FROM "Users";

# For OCPP database:
psql -h localhost -p 5433 -U chargelk -d chargelk_ocpp
SELECT * FROM "ChargePoints" LIMIT 10;
SELECT * FROM "OcppTags";
SELECT * FROM "Transactions" WHERE "StopTimestamp" IS NULL;
```

#### Backup Database to Local Machine

```bash
# With SSH tunnel active
pg_dump -h localhost -p 5433 -U chargelk -d chargelk_um > chargelk_um_backup.sql
pg_dump -h localhost -p 5433 -U chargelk -d chargelk_ocpp > chargelk_ocpp_backup.sql

# Or use SSH to dump directly
ssh chargelk@YOUR_SERVER_IP "sudo -u postgres pg_dump chargelk_um" > chargelk_um_backup.sql
```

#### Restore Database from Local

```bash
# With SSH tunnel active
psql -h localhost -p 5433 -U chargelk -d chargelk_um < chargelk_um_backup.sql

# Or via SSH
cat chargelk_um_backup.sql | ssh chargelk@YOUR_SERVER_IP "psql -U chargelk -d chargelk_um"
```

### Security Best Practices for Remote Access

1. **Always Prefer SSH Tunnel** over direct access
2. **Use Strong Passwords** for database user
3. **Restrict IP Access** if using direct connection
4. **Use SSH Keys** instead of password authentication for SSH
5. **Enable Firewall Rules** to limit PostgreSQL access
6. **Monitor Access Logs** regularly:
   ```bash
   sudo tail -f /var/log/postgresql/postgresql-15-main.log
   ```
7. **Rotate Passwords** periodically
8. **Use Read-Only Users** for non-admin tasks:
   ```sql
   -- Create read-only user
   CREATE USER chargelk_readonly WITH PASSWORD 'strong_password';
   GRANT CONNECT ON DATABASE chargelk_um TO chargelk_readonly;
   GRANT USAGE ON SCHEMA public TO chargelk_readonly;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO chargelk_readonly;
   ```

### Troubleshooting Remote Access

#### SSH Tunnel Not Working

```bash
# Check if SSH is working
ssh chargelk@YOUR_SERVER_IP

# Test with verbose mode
ssh -v -L 5433:localhost:5432 chargelk@YOUR_SERVER_IP -N

# Check if local port is already in use
lsof -i :5433
# or
netstat -an | grep 5433
```

#### Connection Refused (Direct Access)

```bash
# On server, check PostgreSQL is listening
sudo netstat -plnt | grep 5432

# Check PostgreSQL configuration
sudo grep "listen_addresses" /etc/postgresql/15/main/postgresql.conf

# Check pg_hba.conf
sudo cat /etc/postgresql/15/main/pg_hba.conf | grep -v "^#"

# Check firewall
sudo ufw status
```

#### Authentication Failed

```bash
# On server, verify user exists
sudo -u postgres psql -c "\du chargelk"

# Reset password if needed
sudo -u postgres psql -c "ALTER USER chargelk WITH PASSWORD 'new_password';"

# Check pg_hba.conf authentication method
sudo grep "chargelk" /etc/postgresql/15/main/pg_hba.conf
```

## Troubleshooting

### 401 Unauthorized Errors After Login

**Symptoms:**
- Login page works and you can authenticate
- After login, you see multiple 401 (Unauthorized) errors in browser console
- API calls to `/api/ocpp/*` endpoints fail
- Dashboard doesn't load or shows no data

**Root Cause:**
JWT `Issuer` configuration mismatch between UM and OCPP services.

**Solution:**

1. **SSH to server:**
   ```bash
   ssh chargelk@168.231.120.50
   ```

2. **Check current issuer values:**
   ```bash
   grep '"Issuer"' /var/www/chargelk/config/um-appsettings.Production.json
   grep '"Issuer"' /var/www/chargelk/config/ocpp-appsettings.Production.json
   ```

3. **If they show `"http://localhost:5002"`, fix them:**
   ```bash
   # Backup configs
   sudo cp /var/www/chargelk/config/um-appsettings.Production.json \
           /var/www/chargelk/config/um-appsettings.Production.json.backup
   sudo cp /var/www/chargelk/config/ocpp-appsettings.Production.json \
           /var/www/chargelk/config/ocpp-appsettings.Production.json.backup
   
   # Fix UM config
   sudo sed -i 's/"Issuer": "http:\/\/localhost:5002"/"Issuer": "chargelk-um-service"/g' \
           /var/www/chargelk/config/um-appsettings.Production.json
   
   # Fix OCPP config
   sudo sed -i 's/"Issuer": "http:\/\/localhost:5002"/"Issuer": "chargelk-um-service"/g' \
           /var/www/chargelk/config/ocpp-appsettings.Production.json
   
   # Restart services
   sudo systemctl restart chargelk-um chargelk-ocpp
   ```

4. **Clear browser cache or use incognito mode and try logging in again**

**Verification:**
```bash
# Both should show the same issuer
grep '"Issuer"' /var/www/chargelk/config/um-appsettings.Production.json
grep '"Issuer"' /var/www/chargelk/config/ocpp-appsettings.Production.json

# Expected output (both identical):
# "Issuer": "chargelk-um-service",
```

### Services Won't Start

```bash
# Check logs
sudo journalctl -u chargelk-um -xe
sudo journalctl -u chargelk-ocpp -xe

# Check configuration
dotnet /var/www/chargelk/publish/um/ChargeLK.API.dll --environment Production

# Check database connection
psql -U chargelk -d chargelk_um -h localhost
```

### Database Connection Issues

```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Check connection string in appsettings.Production.json
cat /var/www/chargelk/ChargeLK.UM/ChargeLK.API/appsettings.Production.json

# Test connection
psql -U chargelk -h localhost -d chargelk_um -W
```

### Nginx Issues

```bash
# Test configuration
sudo nginx -t

# Check error logs
sudo tail -f /var/log/nginx/error.log

# Restart Nginx
sudo systemctl restart nginx
```

### WebSocket Not Working

```bash
# Check Nginx WebSocket configuration
sudo grep -A 10 "location /ocpp/" /etc/nginx/sites-available/chargelk

# Test WebSocket from command line
wscat -c ws://YOUR_SERVER_IP/ocpp/TEST-CP-001

# Check firewall
sudo ufw status
```

## Security Checklist

- [ ] Change default admin password
- [ ] Use strong PostgreSQL passwords
- [ ] Enable UFW firewall
- [ ] Setup SSL/TLS certificates
- [ ] Regular security updates: `sudo apt update && sudo apt upgrade`
- [ ] Regular database backups
- [ ] Monitor logs for suspicious activity
- [ ] Restrict SSH access (use SSH keys, disable password auth)
- [ ] Use fail2ban for brute force protection

## Performance Tuning

### PostgreSQL
```bash
# Edit PostgreSQL config
sudo vim /etc/postgresql/15/main/postgresql.conf

# Adjust these settings based on your server RAM:
# shared_buffers = 256MB  (25% of RAM)
# effective_cache_size = 1GB  (50% of RAM)
# maintenance_work_mem = 64MB
# checkpoint_completion_target = 0.9
# wal_buffers = 16MB
# default_statistics_target = 100
# random_page_cost = 1.1

# Restart PostgreSQL
sudo systemctl restart postgresql
```

### Nginx
```bash
# Edit Nginx config
sudo vim /etc/nginx/nginx.conf

# worker_processes auto;
# worker_connections 1024;
# keepalive_timeout 65;
# gzip on;
```

## VPS Management

### Restart VPS

```bash
# Reboot the server (recommended - graceful restart)
sudo reboot

# Alternative: shutdown and restart
sudo shutdown -r now

# Schedule restart in 5 minutes (gives time to notify users)
sudo shutdown -r +5 "System will restart in 5 minutes for maintenance"

# Schedule restart at specific time (24-hour format)
sudo shutdown -r 22:00 "System restart scheduled for 10 PM"

# Cancel scheduled restart
sudo shutdown -c
```

**What happens during restart:**
1. All services stop gracefully
2. Database connections close properly
3. System reboots
4. Services auto-start (because they're enabled with systemd)
5. Usually takes 1-3 minutes

**After restart, verify services:**
```bash
# Wait a minute after restart, then check
sudo systemctl status chargelk-um
sudo systemctl status chargelk-ocpp
sudo systemctl status postgresql
sudo systemctl status nginx

# Check all services at once
sudo systemctl status chargelk-* postgresql nginx
```

### Shutdown VPS

```bash
# Shutdown immediately
sudo shutdown -h now

# Alternative
sudo poweroff

# Schedule shutdown in 10 minutes
sudo shutdown -h +10 "Server shutting down in 10 minutes"

# Shutdown at specific time
sudo shutdown -h 23:30 "Server shutdown scheduled for 11:30 PM"
```

### Check System Uptime

```bash
# Show how long system has been running
uptime

# Show last reboot time
who -b

# Show reboot history
last reboot | head
```

### Restart Individual Services

If you don't need to restart the entire VPS, you can restart specific services:

```bash
# Restart UM service only
sudo systemctl restart chargelk-um

# Restart OCPP service only
sudo systemctl restart chargelk-ocpp

# Restart both ChargeLK services
sudo systemctl restart chargelk-um chargelk-ocpp

# Restart Nginx
sudo systemctl restart nginx

# Restart PostgreSQL (be careful - will disconnect all database connections)
sudo systemctl restart postgresql

# Restart all ChargeLK services
sudo systemctl restart chargelk-*
```

### Graceful Service Restarts

```bash
# Reload Nginx configuration without dropping connections
sudo systemctl reload nginx

# Graceful restart with status check
sudo systemctl restart chargelk-um && sudo systemctl status chargelk-um
sudo systemctl restart chargelk-ocpp && sudo systemctl status chargelk-ocpp
```

### Pre-Restart Checklist

Before restarting the VPS, it's good practice to:

```bash
# 1. Check active connections
sudo netstat -tulpn | grep LISTEN

# 2. Check active user sessions
who

# 3. Check running transactions (if any critical operations)
psql -U chargelk -d chargelk_ocpp -c "SELECT * FROM pg_stat_activity WHERE state = 'active';"

# 4. Notify users (if you have active charging sessions)
# Check active transactions
psql -U chargelk -d chargelk_ocpp -c "SELECT * FROM \"Transactions\" WHERE \"StopTimestamp\" IS NULL;"

# 5. Check for pending updates
sudo apt update && sudo apt list --upgradable

# 6. Create quick backup (optional but recommended)
~/backup-chargelk.sh
```

### Post-Restart Verification

```bash
# Quick health check script
cat > ~/check-chargelk-health.sh << 'EOF'
#!/bin/bash
echo "=== ChargeLK Health Check ==="
echo ""

echo "1. Service Status:"
systemctl is-active chargelk-um && echo "✓ UM Service: Running" || echo "✗ UM Service: STOPPED"
systemctl is-active chargelk-ocpp && echo "✓ OCPP Service: Running" || echo "✗ OCPP Service: STOPPED"
systemctl is-active postgresql && echo "✓ PostgreSQL: Running" || echo "✗ PostgreSQL: STOPPED"
systemctl is-active nginx && echo "✓ Nginx: Running" || echo "✗ Nginx: STOPPED"

echo ""
echo "2. Port Listening:"
sudo netstat -tulpn | grep -E ":(5001|5002|5432|80|443)" | awk '{print $4, $7}'

echo ""
echo "3. Health Endpoints:"
curl -s http://localhost:5002/health && echo "✓ UM API: Healthy" || echo "✗ UM API: Not responding"
curl -s http://localhost:5001/health && echo "✓ OCPP API: Healthy" || echo "✗ OCPP API: Not responding"

echo ""
echo "4. Recent Errors (last 5 minutes):"
sudo journalctl -u chargelk-um -u chargelk-ocpp --since "5 minutes ago" --priority=err --no-pager | tail -10

echo ""
echo "=== Health Check Complete ==="
EOF

chmod +x ~/check-chargelk-health.sh

# Run the health check
~/check-chargelk-health.sh
```

### Emergency Recovery

If services don't start after restart:

```bash
# 1. Check service status and logs
sudo systemctl status chargelk-um
sudo journalctl -u chargelk-um -xe

sudo systemctl status chargelk-ocpp
sudo journalctl -u chargelk-ocpp -xe

# 2. Check PostgreSQL first (services depend on it)
sudo systemctl status postgresql
sudo journalctl -u postgresql -xe

# 3. Try starting services manually to see errors
cd /var/www/chargelk/publish/um
dotnet ChargeLK.API.dll --environment Production

# 4. Check configuration files
cat /var/www/chargelk/ChargeLK.UM/ChargeLK.API/appsettings.Production.json
cat /var/www/chargelk/ChargeLK.OCPP/ChargeLK.API/appsettings.Production.json

# 5. Test database connectivity
psql -U chargelk -d chargelk_um -h localhost -W

# 6. Check file permissions
ls -la /var/www/chargelk/publish/um/
ls -la /var/www/chargelk/publish/ocpp/

# 7. Restart in correct order
sudo systemctl restart postgresql
sleep 5
sudo systemctl restart chargelk-um
sudo systemctl restart chargelk-ocpp
sudo systemctl restart nginx
```

### Maintenance Window Best Practices

```bash
# 1. Schedule during low-usage hours (e.g., 2-4 AM)
# 2. Create notification script
cat > ~/notify-maintenance.sh << 'EOF'
#!/bin/bash
echo "$(date): Maintenance window starting" | sudo tee -a /var/log/chargelk/maintenance.log
wall "ChargeLK maintenance starting in 5 minutes. Services will be briefly unavailable."
sleep 300
sudo systemctl restart chargelk-um chargelk-ocpp
sleep 10
~/check-chargelk-health.sh
echo "$(date): Maintenance window complete" | sudo tee -a /var/log/chargelk/maintenance.log
EOF

chmod +x ~/notify-maintenance.sh

# 3. Run maintenance
~/notify-maintenance.sh
```

## Support

For issues or questions:
- Check logs: `/var/log/chargelk/`
- Service status: `sudo systemctl status chargelk-*`
- Application logs: `sudo journalctl -u chargelk-um -u chargelk-ocpp`

---

**Deployment Complete! Your ChargeLK MVP is now running! 🚀⚡**

