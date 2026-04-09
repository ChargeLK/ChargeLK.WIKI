# Operations & Deployment Guide

## Overview

This section contains comprehensive documentation for deploying, operating, and maintaining the ChargeLK system in production environments.

## 📚 Documentation Index

### Deployment Guides

1. **[Deployment Guide](./DEPLOYMENT-GUIDE.md)**
   - Complete production deployment guide
   - Hostinger VPS setup
   - PostgreSQL database configuration
   - Nginx reverse proxy setup
   - SSL/TLS with Let's Encrypt
   - Service configuration and systemd setup
   - Detailed step-by-step instructions

2. **[Deployment Quickstart](./DEPLOYMENT-QUICKSTART.md)**
   - Fast deployment reference
   - Essential commands
   - Quick configuration
   - Common issues and solutions

3. **[Deployment Cheatsheet](./DEPLOYMENT-CHEATSHEET.md)**
   - Quick reference commands
   - Common operations
   - Troubleshooting commands
   - Monitoring and logs

## System Architecture

### Production Architecture

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

## Quick Start

### Prerequisites

- Ubuntu 20.04/22.04 server
- Root or sudo access
- Domain name (recommended)
- Minimum 2GB RAM, 2 CPU cores
- 20GB disk space

### Fast Deployment

```bash
# 1. Connect to server
ssh root@YOUR_SERVER_IP

# 2. Update system
apt update && apt upgrade -y

# 3. Install dependencies
apt install -y git nginx postgresql certbot python3-certbot-nginx

# 4. Install .NET 9
wget https://dot.net/v1/dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 9.0

# 5. Clone repository
git clone https://github.com/your-org/ChargeLK.git
cd ChargeLK

# 6. Configure and deploy
# Follow detailed steps in DEPLOYMENT-GUIDE.md
```

## Key Services

### Backend Services

1. **ChargeLK.UM Service**
   - Port: 5002
   - Purpose: User Management & Authentication
   - Database: chargelk_um (PostgreSQL)

2. **ChargeLK.OCPP Service**
   - Port: 5001
   - Purpose: Charge Point Management
   - Database: chargelk_ocpp (PostgreSQL)
   - WebSocket: Port 5000

### Frontend

3. **Admin Portal**
   - Port: 3000 (proxied via Nginx on 80/443)
   - Technology: React + Vite
   - Static files served by Nginx

### Infrastructure

4. **PostgreSQL Database**
   - Port: 5432
   - Databases: chargelk_um, chargelk_ocpp

5. **Nginx Reverse Proxy**
   - Ports: 80 (HTTP), 443 (HTTPS)
   - SSL/TLS termination
   - Static file serving
   - API request routing

## Common Operations

### Service Management

```bash
# Check service status
sudo systemctl status chargelk-um
sudo systemctl status chargelk-ocpp

# Restart services
sudo systemctl restart chargelk-um
sudo systemctl restart chargelk-ocpp

# View logs
sudo journalctl -u chargelk-um -f
sudo journalctl -u chargelk-ocpp -f
```

### Database Operations

```bash
# Connect to database
sudo -u postgres psql -d chargelk_um
sudo -u postgres psql -d chargelk_ocpp

# Backup database
pg_dump -U postgres chargelk_um > backup_um_$(date +%Y%m%d).sql
pg_dump -U postgres chargelk_ocpp > backup_ocpp_$(date +%Y%m%d).sql

# Restore database
psql -U postgres chargelk_um < backup_um.sql
psql -U postgres chargelk_ocpp < backup_ocpp.sql
```

### Nginx Operations

```bash
# Test configuration
sudo nginx -t

# Reload configuration
sudo systemctl reload nginx

# Restart Nginx
sudo systemctl restart nginx

# View access logs
sudo tail -f /var/log/nginx/access.log

# View error logs
sudo tail -f /var/log/nginx/error.log
```

### SSL Certificate Management

```bash
# Initial certificate setup
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Renew certificates (automatic)
sudo certbot renew --dry-run

# Force renewal
sudo certbot renew --force-renewal
```

## Monitoring & Troubleshooting

### Health Checks

```bash
# Check service endpoints
curl http://localhost:5001/health
curl http://localhost:5002/health

# Check from external
curl https://yourdomain.com/api/ocpp/health
curl https://yourdomain.com/api/um/health
```

### Common Issues

1. **Service won't start**
   - Check logs: `sudo journalctl -u chargelk-um -n 50`
   - Verify database connection
   - Check port availability: `sudo netstat -tulpn | grep 5001`

2. **Database connection errors**
   - Verify PostgreSQL is running: `sudo systemctl status postgresql`
   - Check connection string in appsettings.json
   - Test connection: `psql -U postgres -d chargelk_um`

3. **Nginx 502 Bad Gateway**
   - Verify backend services are running
   - Check backend service logs
   - Verify proxy_pass URLs in Nginx config

4. **SSL certificate issues**
   - Check certificate expiry: `sudo certbot certificates`
   - Verify DNS records
   - Check firewall allows ports 80, 443

### Performance Monitoring

```bash
# System resources
htop
df -h
free -h

# Database performance
sudo -u postgres psql -d chargelk_um -c "SELECT * FROM pg_stat_activity;"

# Service memory usage
ps aux | grep ChargeLK
```

## Backup & Recovery

### Automated Backups

```bash
# Create backup script
cat > /usr/local/bin/backup-chargelk.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/var/backups/chargelk"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup databases
pg_dump -U postgres chargelk_um | gzip > $BACKUP_DIR/um_$DATE.sql.gz
pg_dump -U postgres chargelk_ocpp | gzip > $BACKUP_DIR/ocpp_$DATE.sql.gz

# Keep only last 7 days
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete
EOF

chmod +x /usr/local/bin/backup-chargelk.sh

# Add to crontab (daily at 2 AM)
echo "0 2 * * * /usr/local/bin/backup-chargelk.sh" | sudo crontab -
```

## Security Considerations

### Production Checklist

- ✅ Use HTTPS only (enforce SSL)
- ✅ Strong database passwords
- ✅ Firewall configured (ufw or firewalld)
- ✅ Regular security updates
- ✅ Automated backups
- ✅ Strong JWT secret key
- ✅ Rate limiting enabled
- ✅ CORS properly configured
- ✅ Database encryption at rest
- ✅ Audit logging enabled

### Firewall Configuration

```bash
# Enable firewall
sudo ufw enable

# Allow SSH
sudo ufw allow 22/tcp

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow OCPP WebSocket (if needed externally)
sudo ufw allow 5000/tcp

# Check status
sudo ufw status
```

## Scaling Considerations

### Horizontal Scaling

- Load balancer (Nginx, HAProxy)
- Multiple service instances
- Shared PostgreSQL database
- Redis for distributed caching
- Session management

### Vertical Scaling

- Increase server resources (RAM, CPU)
- Optimize database indexes
- Connection pooling
- Caching strategies

## Related Documentation

- [Development Guide](../07-Development/README.md)
- [Security Guide](../08-Security/README.md)
- [Architecture Overview](../02-Architecture/README.md)
- [Infrastructure Guide](../06-Infrastructure/README.md)

## Support

For deployment issues:
- Check [Deployment Quickstart](./DEPLOYMENT-QUICKSTART.md) for fast reference
- Review [Deployment Cheatsheet](./DEPLOYMENT-CHEATSHEET.md) for common commands
- See [Deployment Guide](./DEPLOYMENT-GUIDE.md) for detailed instructions

