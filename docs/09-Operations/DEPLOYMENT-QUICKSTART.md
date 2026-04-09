# 🚀 ChargeLK MVP Deployment - Quick Start

## Prerequisites

✅ **What you need:**
- Hostinger VPS with Ubuntu 20.04/22.04
- Root SSH access (IP + password)
- Domain name (optional but recommended)

## 📋 Three-Step Deployment Process

### Step 1: Prepare Server (One-Time Setup)

**On your server** (via SSH):

```bash
# 1. Connect to server
ssh root@YOUR_SERVER_IP

# 2. Download the deployment script
wget https://raw.githubusercontent.com/YOUR-REPO/ChargeLK/main/deploy-to-server.sh

# OR upload it manually
# scp deploy-to-server.sh root@YOUR_SERVER_IP:/root/

# 3. Run the setup script
chmod +x deploy-to-server.sh
sudo bash deploy-to-server.sh
```

**What it does:**
- ✅ Installs .NET 9, Node.js 20, PostgreSQL, Nginx
- ✅ Creates databases (chargelk_um, chargelk_ocpp)
- ✅ Configures firewall
- ✅ Sets up systemd services
- ✅ Configures Nginx reverse proxy

**Time:** ~10-15 minutes

---

### Step 2: Upload Application Code

**On your local machine** (in ChargeLK directory):

```bash
# Run the upload script
bash upload-to-server.sh

# It will prompt for:
# - Server IP address
# - SSH user (default: root)
```

**What it does:**
- ✅ Packages your code
- ✅ Removes build artifacts (bin, obj, node_modules)
- ✅ Uploads to /var/www/chargelk/
- ✅ Includes deployment scripts

**Time:** ~5-10 minutes (depends on connection)

---

### Step 3: Build and Deploy

**On your server** (via SSH):

```bash
# 1. Connect to server
ssh root@YOUR_SERVER_IP

# 2. Install Entity Framework Core tools (first time only)
dotnet tool install --global dotnet-ef
echo 'export PATH="$PATH:$HOME/.dotnet/tools"' >> ~/.bashrc
source ~/.bashrc

# If running with sudo, also install for root
sudo -i
dotnet tool install --global dotnet-ef
echo 'export PATH="$PATH:/root/.dotnet/tools"' >> ~/.bashrc
exit

# 3. Navigate to app directory
cd /var/www/chargelk

# 4. Run build script
sudo bash build-and-deploy.sh
```

**What it does:**
- ✅ Installs EF Core tools (dotnet-ef)
- ✅ Builds .NET services (UM + OCPP)
- ✅ Runs database migrations
- ✅ Builds React Admin UI
- ✅ Starts all services
- ✅ Verifies deployment

**Time:** ~5-10 minutes

---

## 🎉 Access Your Application

After deployment completes:

### Admin Portal
```
http://YOUR_SERVER_IP
```

### Login Credentials
```
Email: admin@chargelk.com
Password: Admin@123
```

⚠️ **IMPORTANT:** Change this password immediately after first login!

### API Endpoints
- **UM API:** `http://YOUR_SERVER_IP/api/um`
- **OCPP API:** `http://YOUR_SERVER_IP/api/ocpp`
- **WebSocket:** `ws://YOUR_SERVER_IP/ocpp/{chargeBoxId}`

---

## 🔧 Useful Commands

### Check Service Status
```bash
sudo systemctl status chargelk-um
sudo systemctl status chargelk-ocpp
sudo systemctl status nginx
```

### View Logs
```bash
# Real-time logs
sudo journalctl -u chargelk-um -u chargelk-ocpp -f

# Recent logs
sudo journalctl -u chargelk-um --since "10 minutes ago"
sudo journalctl -u chargelk-ocpp --since "10 minutes ago"

# File logs
sudo tail -f /var/log/chargelk/um-*.log
sudo tail -f /var/log/chargelk/ocpp-*.log
```

### Restart Services
```bash
sudo systemctl restart chargelk-um
sudo systemctl restart chargelk-ocpp
sudo systemctl restart nginx
```

### Test Health Endpoints
```bash
curl http://localhost:5002/health  # UM Service
curl http://localhost:5001/health  # OCPP Service
curl http://YOUR_SERVER_IP/health  # Via Nginx
```

---

## 📦 Manual Deployment (Alternative)

If automated scripts don't work:

### 1. Server Setup
```bash
# Install dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y dotnet-sdk-8.0 nodejs npm postgresql nginx

# Create databases
sudo -u postgres psql -c "CREATE USER chargelk WITH PASSWORD 'your_password';"
sudo -u postgres psql -c "CREATE DATABASE chargelk_um OWNER chargelk;"
sudo -u postgres psql -c "CREATE DATABASE chargelk_ocpp OWNER chargelk;"
```

### 2. Upload Code
```bash
# From local machine
scp -r ChargeLK.UM ChargeLK.OCPP ChargeLK.AdminUI root@YOUR_SERVER_IP:/var/www/chargelk/
```

### 3. Build on Server
```bash
# UM Service
cd /var/www/chargelk/ChargeLK.UM/ChargeLK.API
dotnet publish -c Release -o /var/www/chargelk/publish/um

# OCPP Service
cd /var/www/chargelk/ChargeLK.OCPP/ChargeLK.API
dotnet publish -c Release -o /var/www/chargelk/publish/ocpp

# Admin UI
cd /var/www/chargelk/ChargeLK.AdminUI
npm install
npm run build
cp -r dist/* /var/www/chargelk/adminui/
```

See [DEPLOYMENT-GUIDE.md](./DEPLOYMENT-GUIDE.md) for detailed manual steps.

---

## 🔐 Security Checklist

After deployment:

- [ ] Change default admin password
- [ ] Update PostgreSQL password in production configs
- [ ] Setup SSL/TLS with Let's Encrypt
- [ ] Configure firewall (UFW)
- [ ] Setup regular backups
- [ ] Enable fail2ban
- [ ] Restrict SSH access (keys only)
- [ ] Regular security updates

---

## 🛠️ Troubleshooting

### Services Won't Start

```bash
# Check logs for errors
sudo journalctl -u chargelk-um -xe
sudo journalctl -u chargelk-ocpp -xe

# Test manual start
cd /var/www/chargelk/publish/um
dotnet ChargeLK.API.dll
```

### Database Connection Failed

```bash
# Test PostgreSQL
psql -U chargelk -d chargelk_um -h localhost -W

# Check connection string in:
cat /var/www/chargelk/ChargeLK.UM/ChargeLK.API/appsettings.Production.json
```

### Nginx 502 Bad Gateway

```bash
# Check services are running
sudo systemctl status chargelk-um chargelk-ocpp

# Check Nginx logs
sudo tail -f /var/log/nginx/error.log

# Test Nginx config
sudo nginx -t
```

### Can't Access Admin Portal

```bash
# Check firewall
sudo ufw status

# Check Nginx is serving files
curl http://localhost/
ls -la /var/www/chargelk/adminui/

# Check Nginx config
sudo nginx -t
sudo cat /etc/nginx/sites-available/chargelk
```

---

## 🔄 Update Deployment

To update after making changes:

```bash
# 1. From local machine, upload new code
bash upload-to-server.sh

# 2. On server, rebuild and deploy
ssh root@YOUR_SERVER_IP
cd /var/www/chargelk
bash build-and-deploy.sh
```

---

## 📚 Documentation

- **Detailed Guide:** [DEPLOYMENT-GUIDE.md](./DEPLOYMENT-GUIDE.md)
- **Testing Guide:** [OCPP-1.6-Chargebox-Simulator/TESTING-GUIDE.md](./OCPP-1.6-Chargebox-Simulator/TESTING-GUIDE.md)
- **Configuration:** [OCPP-1.6-Chargebox-Simulator/CHARGE-POINT-CONFIG.md](./OCPP-1.6-Chargebox-Simulator/CHARGE-POINT-CONFIG.md)

---

## 🆘 Support

If you encounter issues:

1. Check service logs: `sudo journalctl -u chargelk-um -u chargelk-ocpp`
2. Verify services are running: `sudo systemctl status chargelk-*`
3. Test health endpoints: `curl http://localhost:5001/health`
4. Check Nginx: `sudo nginx -t && sudo systemctl status nginx`
5. Review deployment guide: [DEPLOYMENT-GUIDE.md](./DEPLOYMENT-GUIDE.md)

---

## 📊 Architecture Overview

```
Internet
    ↓
Nginx (Port 80/443)
    ↓
├─→ Admin UI (Static Files)
├─→ UM API (localhost:5002) → PostgreSQL (chargelk_um)
└─→ OCPP API (localhost:5001) → PostgreSQL (chargelk_ocpp)
    ↓
WebSocket (OCPP Protocol)
```

---

## 🎯 What's Deployed

| Component | Port | Purpose |
|-----------|------|---------|
| **Nginx** | 80, 443 | Reverse proxy, SSL termination |
| **UM Service** | 5002 | User management, authentication |
| **OCPP Service** | 5001 | OCPP protocol, charge points |
| **Admin UI** | - | React dashboard (static files) |
| **PostgreSQL** | 5432 | Database (internal only) |

---

## ✅ Success Indicators

Your deployment is successful when:

✅ All services show "active (running)" status
✅ Health endpoints return 200 OK
✅ Admin portal loads in browser
✅ You can login with default credentials
✅ No errors in service logs

---

**Ready to deploy? Start with Step 1! 🚀**

