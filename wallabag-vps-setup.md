# Complete Wallabag VPS Setup with Let's Encrypt SSL

A comprehensive guide to setting up Wallabag on a VPS with real SSL certificates, avoiding browser security warnings and mobile app configuration issues.

## Overview

This tutorial will help you set up:
- **Wallabag** with PostgreSQL database
- **Real SSL certificate** from Let's Encrypt
- **Custom domain** using Duck DNS (free)
- **Docker-based deployment** for easy management
- **Auto-renewal** for SSL certificates

## Prerequisites

- **VPS or Cloud Server** (any provider: Hetzner, DigitalOcean, AWS, etc.)
- **Ubuntu 22.04 LTS** (or similar Debian-based distribution)
- **Root or sudo access**
- **Basic command line knowledge**

## Step 1: Initial Server Setup

### 1.1 Connect to Your Server

```bash
# SSH to your server
ssh root@YOUR_SERVER_IP

# Or if using a non-root user:
ssh username@YOUR_SERVER_IP
```

### 1.2 Configure Firewall

```bash
# Install and configure UFW firewall
sudo ufw allow 22      # SSH
sudo ufw allow 80      # HTTP (for Let's Encrypt)
sudo ufw allow 443     # HTTPS (standard)
sudo ufw allow 8083    # Wallabag HTTPS (custom port)
sudo ufw --force enable

# Verify firewall status
sudo ufw status
```

### 1.3 Install Required Software

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Install Certbot for Let's Encrypt certificates
sudo apt install -y certbot

# Verify installations
docker --version
docker-compose --version
certbot --version
```

## Step 2: Domain Setup with Duck DNS

### 2.1 Get Free Subdomain

1. **Visit**: [Duck DNS](https://www.duckdns.org/)
2. **Sign in** with Google, GitHub, or create account
3. **Create subdomain**: `your-wallabag.duckdns.org`
4. **Point to your server IP**: Enter your VPS IP address
5. **Update the domain** and note your Duck DNS token

### 2.2 Verify DNS Propagation

```bash
# Check if DNS is working (may take a few minutes)
nslookup your-wallabag.duckdns.org
# Should return your server IP
```

## Step 3: Project Directory Setup

### 3.1 Create Wallabag Directory

```bash
# Create project directory
sudo mkdir -p /opt/wallabag-vps
cd /opt/wallabag-vps

# Create required subdirectories
sudo mkdir -p {db-data,wallabag-data,wallabag-images,nginx-conf,ssl-certs}

# Set proper permissions
sudo chown -R 1000:1000 ./wallabag-data ./wallabag-images
sudo chown -R 999:999 ./db-data
```

## Step 4: SSL Certificate with Let's Encrypt

### 4.1 Obtain Certificate

```bash
# Stop any services using port 80
sudo systemctl stop nginx apache2 2>/dev/null
sudo fuser -k 80/tcp 2>/dev/null

# Generate Let's Encrypt certificate
sudo certbot certonly --standalone \
  --preferred-challenges http \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email \
  -d your-wallabag.duckdns.org

# Note: Replace your-email@example.com with your actual email
# Note: Replace your-wallabag.duckdns.org with your actual domain
```

### 4.2 Copy Certificates

```bash
# Copy certificates to project directory
sudo cp /etc/letsencrypt/live/your-wallabag.duckdns.org/fullchain.pem ./ssl-certs/wallabag.crt
sudo cp /etc/letsencrypt/live/your-wallabag.duckdns.org/privkey.pem ./ssl-certs/wallabag.key

# Set proper permissions
sudo chmod 644 ./ssl-certs/wallabag.crt
sudo chmod 600 ./ssl-certs/wallabag.key
```

## Step 5: Nginx SSL Configuration

### 5.1 Create Nginx Config

```bash
# Create nginx SSL configuration
sudo tee nginx-conf/default.conf > /dev/null << 'EOF'
server {
    listen 8083 ssl;
    server_name your-wallabag.duckdns.org;

    ssl_certificate /etc/ssl/certs/wallabag.crt;
    ssl_certificate_key /etc/ssl/private/wallabag.key;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    location / {
        proxy_pass http://wallabag-app:80;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name:$server_port;
        
        client_max_body_size 100M;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
EOF

# Important: Replace 'your-wallabag.duckdns.org' with your actual domain
sudo sed -i 's/your-wallabag.duckdns.org/YOUR_ACTUAL_DOMAIN/g' nginx-conf/default.conf
```

## Step 6: Docker Compose Configuration

### 6.1 Create Docker Compose File

```bash
# Create docker-compose.yml
sudo tee docker-compose.yml > /dev/null << 'EOF'
version: '3.8'

networks:
  wallabag-network:
    driver: bridge

services:
  wallabag-db:
    image: postgres:13
    container_name: wallabag-db
    restart: unless-stopped
    networks:
      - wallabag-network
    environment:
      POSTGRES_DB: wallabag
      POSTGRES_USER: wallabag
      POSTGRES_PASSWORD: your_secure_db_password_here
    volumes:
      - ./db-data:/var/lib/postgresql/data

  wallabag-app:
    image: wallabag/wallabag:latest
    container_name: wallabag-app
    restart: unless-stopped
    networks:
      - wallabag-network
    depends_on:
      - wallabag-db
    environment:
      SYMFONY__ENV__DOMAIN_NAME: "https://your-wallabag.duckdns.org:8083"
      SYMFONY__ENV__DATABASE_DRIVER: pdo_pgsql
      SYMFONY__ENV__DATABASE_HOST: wallabag-db
      SYMFONY__ENV__DATABASE_PORT: 5432
      SYMFONY__ENV__DATABASE_NAME: wallabag
      SYMFONY__ENV__DATABASE_USER: wallabag
      SYMFONY__ENV__DATABASE_PASSWORD: your_secure_db_password_here
    volumes:
      - ./wallabag-data:/var/www/wallabag/data
      - ./wallabag-images:/var/www/wallabag/web/assets/images

  wallabag-nginx:
    image: nginx:alpine
    container_name: wallabag-nginx
    restart: unless-stopped
    networks:
      - wallabag-network
    depends_on:
      - wallabag-app
    ports:
      - "8083:8083"
    volumes:
      - ./nginx-conf:/etc/nginx/conf.d
      - ./ssl-certs:/etc/ssl/certs
      - ./ssl-certs:/etc/ssl/private
EOF

# Important: Update the configuration with your values
sudo sed -i 's/your-wallabag.duckdns.org/YOUR_ACTUAL_DOMAIN/g' docker-compose.yml
sudo sed -i 's/your_secure_db_password_here/YOUR_SECURE_PASSWORD/g' docker-compose.yml
```

## Step 7: Launch Wallabag

### 7.1 Start Services

```bash
# Start all services
sudo docker-compose up -d

# Monitor startup (wait for "wallabag is ready!")
sudo docker-compose logs -f wallabag-app

# Press Ctrl+C when ready
```

### 7.2 Verify Container Status

```bash
# Check that all containers are running
sudo docker-compose ps

# All should show "Up" status
```

## Step 8: Initialize Wallabag

### 8.1 Run Setup Wizard

```bash
# Enter the Wallabag container
sudo docker-compose exec wallabag-app sh

# Run the installation wizard
php bin/console wallabag:install --env=prod
```

**Follow the prompts:**
1. **Database setup**: Choose "Continue" (uses existing database)
2. **Create admin user**: Choose "Yes"
   - **Username**: Choose your admin username
   - **Password**: Choose a secure password
   - **Email**: Enter your email address
3. **Config setup**: Choose "Continue"

```bash
# Exit the container
exit
```

### 8.2 Fix Permissions (if needed)

```bash
# If you encounter permission errors:
sudo docker-compose exec wallabag-app sh -c "
chown -R root:root /var/www/wallabag/var/cache
chmod -R 777 /var/www/wallabag/var/cache
php bin/console cache:clear --env=prod
"
```

## Step 9: Test Your Installation

### 9.1 Verify SSL and API

```bash
# Test API endpoint (should work without -k flag)
curl https://your-wallabag.duckdns.org:8083/api/info

# Should return:
# {"appname":"wallabag","version":"2.6.13","allowed_registration":false}
```

### 9.2 Access Web Interface

1. **Open browser**: `https://your-wallabag.duckdns.org:8083`
2. **Login** with your admin credentials
3. **Verify**: Green lock icon (trusted SSL certificate)

## Step 10: API Setup

### 10.1 Create API Client

1. **Go to**: Settings → API clients management
2. **Click**: "Create a new client"
3. **Note down**:
   - **Client ID**: `[generated_value]`
   - **Client Secret**: `[generated_value]`

### 10.2 Test API Authentication

```bash
# Test authentication (replace with your values)
TOKEN=$(curl -s -X POST "https://your-wallabag.duckdns.org:8083/oauth/v2/token" \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "password",
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET",
    "username": "YOUR_USERNAME",
    "password": "YOUR_PASSWORD"
  }' | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

# Test adding an article
curl -X POST "https://your-wallabag.duckdns.org:8083/api/entries.json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com", "tags": "test"}'
```

## Step 11: SSL Certificate Auto-Renewal

### 11.1 Create Renewal Script

```bash
# Create auto-renewal script
sudo tee /root/renew-wallabag-cert.sh > /dev/null << 'EOF'
#!/bin/bash

LOG_FILE="/var/log/wallabag-cert-renewal.log"
DOMAIN="your-wallabag.duckdns.org"
WALLABAG_DIR="/opt/wallabag-vps"

echo "$(date): Starting certificate renewal for $DOMAIN..." >> $LOG_FILE

# Stop any services using port 80
systemctl stop nginx apache2 2>/dev/null
fuser -k 80/tcp 2>/dev/null

# Stop Wallabag to free resources
cd $WALLABAG_DIR && docker-compose down

# Renew certificate
certbot renew --standalone --quiet

if [ $? -eq 0 ]; then
    echo "$(date): Certificate renewed successfully" >> $LOG_FILE
    
    # Copy renewed certificates
    cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem $WALLABAG_DIR/ssl-certs/wallabag.crt
    cp /etc/letsencrypt/live/$DOMAIN/privkey.pem $WALLABAG_DIR/ssl-certs/wallabag.key
    chmod 644 $WALLABAG_DIR/ssl-certs/wallabag.crt
    chmod 600 $WALLABAG_DIR/ssl-certs/wallabag.key
    
    echo "$(date): Certificates updated successfully" >> $LOG_FILE
else
    echo "$(date): Certificate renewal failed" >> $LOG_FILE
fi

# Restart Wallabag
cd $WALLABAG_DIR && docker-compose up -d

echo "$(date): Wallabag services restarted" >> $LOG_FILE
EOF

# Update script with your domain
sudo sed -i 's/your-wallabag.duckdns.org/YOUR_ACTUAL_DOMAIN/g' /root/renew-wallabag-cert.sh

# Make executable
sudo chmod +x /root/renew-wallabag-cert.sh
```

### 11.2 Schedule Auto-Renewal

```bash
# Add monthly cron job (1st of each month at 3 AM)
(sudo crontab -l 2>/dev/null; echo "0 3 1 * * /root/renew-wallabag-cert.sh") | sudo crontab -

# Verify cron job
sudo crontab -l
```

## Step 12: Mobile App Setup

### 12.1 Mobile Configuration

**For iOS/Android Wallabag apps:**
- **Server URL**: `https://your-wallabag.duckdns.org:8083`
- **Username**: Your admin username
- **Password**: Your admin password
- **SSL Certificate**: Trusted (no exceptions needed!)

### 12.2 Browser Bookmarklet

The web interface provides bookmarklets for easy article saving from any browser.

## Step 13: Maintenance & Troubleshooting

### 13.1 Common Commands

```bash
# View logs
sudo docker-compose -f /opt/wallabag-vps/docker-compose.yml logs

# Restart services
sudo docker-compose -f /opt/wallabag-vps/docker-compose.yml restart

# Update containers
sudo docker-compose -f /opt/wallabag-vps/docker-compose.yml pull
sudo docker-compose -f /opt/wallabag-vps/docker-compose.yml up -d

# Backup database
sudo docker-compose exec wallabag-db pg_dump -U wallabag wallabag > backup_$(date +%Y%m%d).sql
```

### 13.2 Check Certificate Status

```bash
# Check certificate expiration
sudo certbot certificates

# Test SSL connection
openssl s_client -connect your-wallabag.duckdns.org:8083 -servername your-wallabag.duckdns.org
```

### 13.3 Common Issues

**Port 80 in use during renewal:**
```bash
# Stop conflicting services
sudo systemctl stop nginx apache2
sudo fuser -k 80/tcp
# Then run renewal script
```

**Permission errors in container:**
```bash
sudo docker-compose exec wallabag-app sh -c "
chown -R root:root /var/www/wallabag/var
chmod -R 777 /var/www/wallabag/var/cache
php bin/console cache:clear --env=prod
"
```

**SSL certificate not working:**
```bash
# Verify certificate files exist and have correct permissions
ls -la /opt/wallabag-vps/ssl-certs/
```

## Step 14: Security Considerations

### 14.1 Firewall Rules

```bash
# Review and tighten firewall rules
sudo ufw status numbered

# Remove unnecessary rules if needed
sudo ufw delete [rule_number]
```

### 14.2 Regular Updates

```bash
# Monthly system updates
sudo apt update && sudo apt upgrade -y

# Update Docker containers quarterly
cd /opt/wallabag-vps
sudo docker-compose pull
sudo docker-compose up -d
```

### 14.3 Backup Strategy

**Recommended backup schedule:**
- **Daily**: Database backups
- **Weekly**: Complete data directory backup
- **Monthly**: Full system backup

## Success Criteria

After completing this tutorial, you should have:

✅ **Wallabag accessible** via HTTPS with custom domain  
✅ **Valid SSL certificate** (green lock in browsers)  
✅ **No security warnings** in browsers or mobile apps  
✅ **Working API** for integrations  
✅ **Auto-renewal** configured for SSL certificates  
✅ **Mobile app compatibility** without security exceptions  

## Additional Resources

- **Wallabag Documentation**: [doc.wallabag.org](https://doc.wallabag.org)
- **Let's Encrypt**: [letsencrypt.org](https://letsencrypt.org)
- **Duck DNS**: [duckdns.org](https://duckdns.org)
- **Docker Compose**: [docs.docker.com/compose](https://docs.docker.com/compose)

## Conclusion

You now have a fully functional, secure Wallabag installation with:
- **Professional SSL certificate** that works everywhere
- **Custom domain** that's easy to remember
- **Automatic certificate renewal** for hands-off maintenance
- **Mobile app compatibility** without security workarounds
