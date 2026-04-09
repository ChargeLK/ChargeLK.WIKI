# Quick Access Guide - ChargeLK Admin Portal

## 🌐 How to Access Your Admin Portal from Internet

### Step 1: Find Your Server IP Address

If you're already connected to your server via SSH, run:
```bash
# Get your public IP address
curl ifconfig.me
```

Or check your Hostinger VPS dashboard for the server's public IP address.

### Step 2: Access the Admin Portal

Once your ChargeLK application is deployed, you can access it from anywhere on the internet:

#### Using IP Address (Immediate Access)
```
http://YOUR_SERVER_IP
```

**Example:** If your server IP is `123.45.67.89`, open your browser and go to:
```
http://123.45.67.89
```

#### Using Domain Name (If Configured)
```
https://your-domain.com
```

**Example:** If you configured a domain like `chargelk.example.com`:
```
https://chargelk.example.com
```

### Step 3: Login to Admin Portal

Use the default credentials (change these after first login!):

- **Email:** `admin@chargelk.com`
- **Password:** `Admin@123`

⚠️ **IMPORTANT:** Change the default password immediately after your first login for security!

---

## 🔗 All Access URLs

### For Browser Access (Web UI)
- **Admin Portal:** `http://YOUR_SERVER_IP`
  - This is your main web interface

### For API Calls (Backend Services)
- **User Management API:** `http://YOUR_SERVER_IP/api/um`
- **OCPP API:** `http://YOUR_SERVER_IP/api/ocpp`
- **Health Check:** `http://YOUR_SERVER_IP/health`

### For Charge Points (WebSocket)
- **OCPP WebSocket:** `ws://YOUR_SERVER_IP/ocpp/{chargeBoxId}`
  - Replace `{chargeBoxId}` with your actual charge box identifier
  - Example: `ws://123.45.67.89/ocpp/CP001`

---

## 🔐 With SSL/TLS (HTTPS)

If you've configured SSL certificates using Let's Encrypt:

- **Admin Portal:** `https://your-domain.com`
- **User Management API:** `https://your-domain.com/api/um`
- **OCPP API:** `https://your-domain.com/api/ocpp`
- **OCPP WebSocket (Secure):** `wss://your-domain.com/ocpp/{chargeBoxId}`

---

## ✅ Quick Verification Checklist

Before accessing from the internet, verify:

### 1. Services Are Running
```bash
# SSH to your server
ssh chargelk@YOUR_SERVER_IP

# Check service status
sudo systemctl status chargelk-um
sudo systemctl status chargelk-ocpp
sudo systemctl status nginx

# All should show "active (running)" in green
```

### 2. Firewall Allows HTTP/HTTPS Traffic
```bash
# Check firewall status
sudo ufw status

# Should show:
# 80/tcp    ALLOW    Anywhere  (HTTP)
# 443/tcp   ALLOW    Anywhere  (HTTPS)
```

If not configured:
```bash
sudo ufw allow 'Nginx Full'
sudo ufw reload
```

### 3. Nginx Is Properly Configured
```bash
# Test Nginx configuration
sudo nginx -t

# Should show: "syntax is ok" and "test is successful"

# If you made changes, reload Nginx
sudo systemctl reload nginx
```

### 4. Test Health Endpoint
```bash
# From your server
curl http://localhost:5001/health
curl http://localhost:5002/health

# From the internet (replace YOUR_SERVER_IP)
curl http://YOUR_SERVER_IP/health
```

---

## 🌍 Access from Different Devices

### From Your Computer
1. Open any web browser (Chrome, Firefox, Safari, Edge)
2. Type: `http://YOUR_SERVER_IP`
3. Press Enter
4. Login with credentials

### From Your Phone/Tablet
1. Open Safari (iOS) or Chrome (Android)
2. Type: `http://YOUR_SERVER_IP`
3. Login with credentials

### From Another Location
As long as you have internet access, you can access your admin portal from anywhere:
- Your home
- Your office
- A coffee shop
- Another country

Just use the same URL: `http://YOUR_SERVER_IP`

---

## 🚫 Troubleshooting Access Issues

### Problem: "This site can't be reached" or "Connection refused"

**Solution 1: Check if services are running**
```bash
ssh chargelk@YOUR_SERVER_IP
sudo systemctl status chargelk-um chargelk-ocpp nginx
```

**Solution 2: Check firewall**
```bash
sudo ufw status
# Make sure ports 80 and 443 are allowed
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

**Solution 3: Check Nginx**
```bash
sudo systemctl status nginx
sudo nginx -t
sudo systemctl restart nginx
```

### Problem: "502 Bad Gateway"

This means Nginx is running but can't connect to your backend services.

**Solution:**
```bash
# Check if backend services are running
sudo systemctl status chargelk-um
sudo systemctl status chargelk-ocpp

# If stopped, start them
sudo systemctl start chargelk-um
sudo systemctl start chargelk-ocpp

# Check logs for errors
sudo journalctl -u chargelk-um -n 50
sudo journalctl -u chargelk-ocpp -n 50
```

### Problem: Blank page or "Cannot GET /"

**Solution:** Check if Admin UI files are properly deployed
```bash
# Check if files exist
ls -la /var/www/chargelk/adminui/

# Should see: index.html, assets/, etc.

# If missing, rebuild and deploy Admin UI
cd /var/www/chargelk/ChargeLK.AdminUI
npm run build
sudo cp -r dist/* /var/www/chargelk/adminui/
sudo chown -R www-data:www-data /var/www/chargelk/adminui/
```

### Problem: "Invalid credentials" when logging in

**Solution:** Check if database is initialized
```bash
# Check if database exists
psql -U chargelk -d chargelk_um -h localhost -c "\dt"

# If empty, run migrations
cd /var/www/chargelk/ChargeLK.UM/ChargeLK.API
dotnet ef database update --environment Production
```

---

## 🔧 Advanced: Custom Domain Setup

### Step 1: Point Domain to Server IP

In your domain registrar (GoDaddy, Namecheap, Cloudflare, etc.):

1. Add an **A Record**:
   - **Name:** `@` (or your subdomain like `chargelk`)
   - **Type:** `A`
   - **Value:** `YOUR_SERVER_IP`
   - **TTL:** `3600` (or automatic)

2. Wait for DNS propagation (5 minutes to 48 hours, usually ~10 minutes)

### Step 2: Update Nginx Configuration

```bash
# Edit Nginx config
sudo nano /etc/nginx/sites-available/chargelk

# Update server_name line:
server_name your-domain.com www.your-domain.com;

# Save and exit (Ctrl+X, Y, Enter)

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

### Step 3: Install SSL Certificate

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Get certificate
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# Follow the prompts
# Certbot will automatically configure HTTPS
```

### Step 4: Update Admin UI Environment

```bash
cd /var/www/chargelk/ChargeLK.AdminUI

# Create/update .env.production
cat > .env.production << 'EOF'
VITE_API_BASE_URL=https://your-domain.com/api
VITE_UM_API_URL=https://your-domain.com/api/um
VITE_OCPP_API_URL=https://your-domain.com/api/ocpp
VITE_WS_URL=wss://your-domain.com/ocpp
EOF

# Rebuild
npm run build
sudo cp -r dist/* /var/www/chargelk/adminui/
sudo systemctl reload nginx
```

### Step 5: Access via Domain

Now you can access:
- `https://your-domain.com` (automatically redirects from http)

---

## 📱 Share Access with Team Members

To give others access to your admin portal:

### For Viewing/Testing:
Share the URL: `http://YOUR_SERVER_IP`

### For Admin Access:
1. Share the URL
2. Create separate user accounts (don't share admin credentials!)
3. Go to Admin Portal → Users → Create New User
4. Send them their credentials securely

### For Charge Point Configuration:
Share the OCPP WebSocket URL:
```
ws://YOUR_SERVER_IP/ocpp/{chargeBoxId}
```

---

## 🔒 Security Best Practices

1. **Change Default Password Immediately**
   - Login → Profile → Change Password

2. **Enable HTTPS/SSL**
   - Get a free certificate from Let's Encrypt
   - Use `sudo certbot --nginx`

3. **Use Strong Passwords**
   - Minimum 12 characters
   - Mix of letters, numbers, symbols

4. **Regular Backups**
   - Database backups daily
   - See deployment guide for backup script

5. **Monitor Logs**
   ```bash
   sudo tail -f /var/log/chargelk/*.log
   sudo journalctl -u chargelk-um -f
   ```

6. **Keep Software Updated**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

## 🆘 Need More Help?

### Check Service Logs
```bash
# Real-time logs
sudo journalctl -u chargelk-um -u chargelk-ocpp -f

# Last 100 lines
sudo journalctl -u chargelk-um -n 100
```

### Check Application Logs
```bash
# UM Service logs
sudo tail -f /var/log/chargelk/um-*.log

# OCPP Service logs
sudo tail -f /var/log/chargelk/ocpp-*.log
```

### Check Nginx Logs
```bash
# Access logs
sudo tail -f /var/log/nginx/access.log

# Error logs
sudo tail -f /var/log/nginx/error.log
```

### Full Service Restart
```bash
# Restart all services
sudo systemctl restart chargelk-um
sudo systemctl restart chargelk-ocpp
sudo systemctl restart nginx
sudo systemctl restart postgresql

# Check status
sudo systemctl status chargelk-um chargelk-ocpp nginx
```

---

## 📊 Quick Test Script

Save this as `test-access.sh` and run it to verify everything:

```bash
#!/bin/bash

echo "=== ChargeLK Access Test ==="
echo ""

# Get server IP
SERVER_IP=$(curl -s ifconfig.me)
echo "Server IP: $SERVER_IP"
echo ""

# Test services
echo "Testing UM Service..."
curl -s http://localhost:5002/health && echo "✅ UM Service OK" || echo "❌ UM Service FAILED"

echo "Testing OCPP Service..."
curl -s http://localhost:5001/health && echo "✅ OCPP Service OK" || echo "❌ OCPP Service FAILED"

echo "Testing Nginx..."
curl -s http://localhost/health && echo "✅ Nginx OK" || echo "❌ Nginx FAILED"

echo ""
echo "=== Access URLs ==="
echo "Admin Portal: http://$SERVER_IP"
echo "UM API: http://$SERVER_IP/api/um"
echo "OCPP API: http://$SERVER_IP/api/ocpp"
echo ""
echo "Try opening in browser: http://$SERVER_IP"
```

Make it executable and run:
```bash
chmod +x test-access.sh
./test-access.sh
```

---

**Your ChargeLK Admin Portal is accessible from anywhere! 🌍🚀**

**Default login:** `admin@chargelk.com` / `Admin@123`

**⚠️ Remember to change the password after first login!**

