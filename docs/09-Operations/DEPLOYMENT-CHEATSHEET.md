# 🚀 ChargeLK Deployment - One Page Reference

## 📦 What You Have

✅ **3 Automated Scripts:**
- `deploy-to-server.sh` - Server setup
- `upload-to-server.sh` - Upload code
- `build-and-deploy.sh` - Build & start

✅ **2 Comprehensive Guides:**
- `DEPLOYMENT-GUIDE.md` - Manual steps
- `DEPLOYMENT-QUICKSTART.md` - Quick reference

---

## ⚡ 3-Step Quick Deploy

### Step 1: Prepare Server (15 min)
```bash
# Upload setup script to server
scp deploy-to-server.sh root@YOUR_SERVER_IP:/root/

# SSH and run setup
ssh root@YOUR_SERVER_IP
chmod +x /root/deploy-to-server.sh
sudo bash /root/deploy-to-server.sh
```
**Installs:** .NET 9, Node.js, PostgreSQL, Nginx

---

### Step 2: Upload Code (5 min)
```bash
# From YOUR LOCAL machine
cd /home/kiviamarakoon/Documents/Projects/my-projects/ChargeLK
bash upload-to-server.sh

# Enter when prompted:
# - Server IP: YOUR_SERVER_IP
# - SSH user: root
```
**Uploads:** All services + Admin UI

---

### Step 3: Build & Deploy (10 min)
```bash
# SSH to server
ssh root@YOUR_SERVER_IP

# Run build script
cd /var/www/chargelk
bash build-and-deploy.sh
```
**Builds & Starts:** Everything!

---

## 🎯 Access After Deployment

```
Admin Portal:  http://YOUR_SERVER_IP
Login:         admin@chargelk.com / Admin@123

APIs:
- UM:          http://YOUR_SERVER_IP/api/um
- OCPP:        http://YOUR_SERVER_IP/api/ocpp
- WebSocket:   ws://YOUR_SERVER_IP/ocpp/{chargeBoxId}
```

---

## 🔍 Quick Status Check

```bash
# Check services
sudo systemctl status chargelk-um chargelk-ocpp nginx

# Test health
curl http://localhost:5002/health  # UM
curl http://localhost:5001/health  # OCPP

# View logs
sudo journalctl -u chargelk-um -u chargelk-ocpp -f
```

---

## 🛠️ Common Tasks

### Restart Services
```bash
sudo systemctl restart chargelk-um chargelk-ocpp nginx
```

### View Logs
```bash
sudo journalctl -u chargelk-um -n 50
sudo tail -f /var/log/chargelk/um-*.log
```

### Update Code
```bash
# Local: Upload new code
bash upload-to-server.sh

# Server: Rebuild
cd /var/www/chargelk && bash build-and-deploy.sh
```

### Register RFID Tag
```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:5002/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@chargelk.com","password":"Admin@123"}' \
  | grep -o '"accessToken":"[^"]*' | sed 's/"accessToken":"//')

# Register tag
curl -X POST http://localhost:5001/api/OcppTags \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"idTag":"04B0267AE05C87","note":"Test tag","expiryDate":"2026-12-31T23:59:59Z"}'
```

---

## 🔥 Troubleshooting Quick Fixes

### Service Won't Start
```bash
sudo journalctl -u chargelk-um -xe  # Check logs
sudo systemctl restart chargelk-um   # Restart
```

### 502 Bad Gateway
```bash
sudo systemctl restart chargelk-um chargelk-ocpp
sudo nginx -t && sudo systemctl restart nginx
```

### Can't Access Portal
```bash
sudo ufw allow 80
sudo systemctl restart nginx
ls /var/www/chargelk/adminui/  # Check files exist
```

### Database Error
```bash
psql -U chargelk -d chargelk_um -h localhost -W
# Check password matches appsettings.Production.json
```

---

## 📊 File Locations

```
/var/www/chargelk/          # Application root
├── ChargeLK.UM/            # UM source
├── ChargeLK.OCPP/          # OCPP source
├── ChargeLK.AdminUI/       # UI source
├── publish/
│   ├── um/                 # UM compiled
│   └── ocpp/               # OCPP compiled
├── adminui/                # UI static files
└── config/                 # Production configs

/var/log/chargelk/          # Application logs
├── um-YYYYMMDD.log
└── ocpp-YYYYMMDD.log

/etc/systemd/system/        # Service files
├── chargelk-um.service
└── chargelk-ocpp.service

/etc/nginx/sites-available/ # Nginx config
└── chargelk
```

---

## 🎯 Success Indicators

✅ **All systems operational when:**
- Services show "active (running)"
- Health endpoints return "Healthy"
- Admin portal loads
- Can login successfully
- No errors in logs

---

## 🔗 Quick Links

| What | Where |
|------|-------|
| Full Guide | `DEPLOYMENT-GUIDE.md` |
| Quick Start | `DEPLOYMENT-QUICKSTART.md` |
| Testing | `OCPP-1.6-Chargebox-Simulator/TESTING-GUIDE.md` |
| Config | `CHARGE-POINT-CONFIG.md` |

---

## 📞 Emergency Commands

### Stop Everything
```bash
sudo systemctl stop chargelk-um chargelk-ocpp nginx
```

### Start Everything
```bash
sudo systemctl start chargelk-um chargelk-ocpp nginx
```

### Nuclear Reset (Fresh Start)
```bash
sudo systemctl stop chargelk-um chargelk-ocpp
sudo -u postgres dropdb chargelk_um chargelk_ocpp
cd /var/www/chargelk
bash build-and-deploy.sh
```

---

## 💾 Backup Database
```bash
sudo -u postgres pg_dump chargelk_um > backup_um.sql
sudo -u postgres pg_dump chargelk_ocpp > backup_ocpp.sql
```

## 📥 Restore Database
```bash
sudo -u postgres psql chargelk_um < backup_um.sql
sudo -u postgres psql chargelk_ocpp < backup_ocpp.sql
```

---

## 🚀 Deploy Now!

```bash
# Start here:
cd /home/kiviamarakoon/Documents/Projects/my-projects/ChargeLK
bash upload-to-server.sh
```

**Time: ~30 minutes from start to running system!**

---

**Your ChargeLK MVP deployment toolkit is ready! 🎉**

