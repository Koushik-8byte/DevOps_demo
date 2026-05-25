# Complete Nginx Guide for Ubuntu: Web Server, Reverse Proxy, Load Balancer & SSL/DNS

## Table of Contents
1. [Nginx Basics & Installation](#1-nginx-basics--installation)
2. [Nginx as a Web Server](#2-nginx-as-a-web-server)
3. [Nginx as a Reverse Proxy](#3-nginx-as-a-reverse-proxy)
4. [Nginx as a Load Balancer](#4-nginx-as-a-load-balancer)
5. [SSL/TLS Certificate Setup](#5-ssltls-certificate-setup)
6. [DNS Configuration](#6-dns-configuration)
7. [Common Commands & Troubleshooting](#7-common-commands--troubleshooting)

---

## 1. Nginx Basics & Installation

### What is Nginx?
Nginx is a lightweight, high-performance web server that can act as:
- **Web Server**: Serves static and dynamic content
- **Reverse Proxy**: Forwards requests to backend servers
- **Load Balancer**: Distributes traffic across multiple servers
- **API Gateway**: Routes API requests

### Installation on Ubuntu

```bash
# Update package manager
sudo apt update
sudo apt upgrade

# Install Nginx
sudo apt install nginx

# Start Nginx
sudo systemctl start nginx

# Enable on startup (starts automatically after reboot)
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

### Verify Installation

```bash
# Check Nginx is running
sudo systemctl status nginx

# Check if listening on port 80
sudo ss -tlnp | grep nginx

# Check Nginx version
nginx -v
```

### Key Directories on Ubuntu

```
/etc/nginx/                    - Main configuration directory
├── nginx.conf                 - Main configuration file
├── sites-available/           - Available site configurations
├── sites-enabled/             - Active site configurations (symlinks)
├── conf.d/                    - Additional configuration files
└── snippets/                  - Reusable config snippets

/var/www/                      - Default web root directory
/var/www/html/                 - Default website files

/var/log/nginx/                - Log files
├── access.log                 - HTTP request logs
└── error.log                  - Error logs

/usr/sbin/nginx                - Nginx executable
```

### Understanding the Workflow

```
1. Client sends request to http://mysite.com:80
2. Ubuntu routes to Nginx (listening on port 80)
3. Nginx checks /etc/nginx/sites-enabled/ for matching config
4. Config specifies root directory (e.g., /var/www/mysite/html)
5. Nginx serves file or proxies to backend
6. Response sent back to client
```

---

## 2. Nginx as a Web Server

A web server serves static files (HTML, CSS, JS, images) directly to clients.

### Step 1: Create Website Directory

```bash
# Create directory structure
sudo mkdir -p /var/www/mywebsite.com/html

# Create a simple index.html file
sudo nano /var/www/mywebsite.com/html/index.html
```

**Paste this HTML content:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to My Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        h1 {
            color: #333;
        }
    </style>
</head>
<body>
    <h1>✓ Nginx is Working!</h1>
    <p>Your website is live and running on Nginx.</p>
    <p>Server: <strong>Ubuntu</strong></p>
    <p>Time: <?php echo date('Y-m-d H:i:s'); ?></p>
</body>
</html>
```

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 2: Set Permissions

```bash
# Change owner to www-data (Nginx user on Ubuntu)
sudo chown -R www-data:www-data /var/www/mywebsite.com

# Set directory permissions
sudo chmod -R 755 /var/www/mywebsite.com

# Set file permissions
sudo chmod 644 /var/www/mywebsite.com/html/*

# Verify ownership
ls -la /var/www/mywebsite.com/
```

### Step 3: Create Nginx Server Block Configuration

```bash
# Create configuration file
sudo nano /etc/nginx/sites-available/mywebsite.com
```

**Paste this configuration:**
```nginx
server {
    # Listen on port 80 (HTTP)
    listen 80;
    listen [::]:80;
    
    # Domain names this server responds to
    server_name mywebsite.com www.mywebsite.com;
    
    # Root directory where files are stored
    root /var/www/mywebsite.com/html;
    
    # Default index files to serve
    index index.html index.htm;
    
    # Log files
    access_log /var/log/nginx/mywebsite.com.access.log;
    error_log /var/log/nginx/mywebsite.com.error.log;
    
    # Main location block
    location / {
        # Try to serve the requested file, if not found try as directory, if still not found return 404
        try_files $uri $uri/ =404;
    }
    
    # Cache static assets for 30 days
    location ~ \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # Deny access to hidden files (starting with .)
    location ~ /\. {
        deny all;
    }
    
    # Deny access to sensitive config files
    location ~ \.(conf|json|env|yml)$ {
        deny all;
    }
}
```

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 4: Enable the Site

```bash
# Create symbolic link to enable the site
sudo ln -s /etc/nginx/sites-available/mywebsite.com \
    /etc/nginx/sites-enabled/mywebsite.com

# Verify the symlink was created
ls -la /etc/nginx/sites-enabled/

# You should see: mywebsite.com -> ../sites-available/mywebsite.com
```

### Step 5: Disable Default Site (Optional)

```bash
# Remove default site if you don't need it
sudo rm /etc/nginx/sites-enabled/default

# Verify it's disabled
ls /etc/nginx/sites-enabled/
```

### Step 6: Test Configuration

```bash
# Test Nginx configuration syntax
sudo nginx -t

# You should see:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Step 7: Apply Configuration

```bash
# Reload Nginx (no downtime)
sudo systemctl reload nginx

# Check status
sudo systemctl status nginx
```

### Step 8: Test in Browser

```bash
# Get your server's IP address
hostname -I

# Or use curl to test
curl http://localhost
curl http://mywebsite.com   # if DNS is configured
```

### Common Location Blocks Reference

```nginx
# Serve static files with caching
location ~ \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}

# Block access to hidden files
location ~ /\. {
    deny all;
}

# Block access to config files
location ~ \.(conf|json|env|pem)$ {
    deny all;
}

# Compress text responses
location ~ \.(html|xml|txt|css|js)$ {
    gzip on;
    gzip_types text/plain text/css text/xml text/javascript;
}

# Redirect to HTTPS (when using SSL)
location / {
    return 301 https://$server_name$request_uri;
}

# Custom error page
error_page 404 /404.html;
location = /404.html {
    internal;
}
```

---

## 3. Nginx as a Reverse Proxy

A reverse proxy forwards client requests to backend servers and returns responses. This is used when you have an application running on a different port.

### Common Scenario:
- Client connects to Nginx on port 80
- Nginx forwards to your Node.js/Python/Java app on port 3000/5000/8080
- Backend sends response back to Nginx
- Nginx sends to client

### Step 1: Start Your Backend Application

Example: Node.js app running on localhost:3000

```bash
# Example Node.js app (app.js)
cd /home/ubuntu/myapp
node app.js

# Should output: Server running on port 3000
```

### Step 2: Create Reverse Proxy Configuration

```bash
# Create configuration
sudo nano /etc/nginx/sites-available/myapp.com
```

**Paste this configuration:**
```nginx
# Define upstream backend
upstream backend {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    listen [::]:80;
    
    server_name myapp.com www.myapp.com;
    
    # Access and error logs
    access_log /var/log/nginx/myapp.com.access.log;
    error_log /var/log/nginx/myapp.com.error.log;
    
    # Proxy all requests to backend
    location / {
        # Forward request to backend server
        proxy_pass http://backend;
        
        # Important: Send original client information to backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Connection timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering configuration
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Static files - serve directly from disk if available
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg)$ {
        root /var/www/myapp;
        expires 30d;
    }
}
```

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 3: Enable the Site

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/myapp.com \
    /etc/nginx/sites-enabled/myapp.com

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 4: Test the Reverse Proxy

```bash
# If backend is running, test with curl
curl http://localhost

# Check logs to see if requests are going through
sudo tail -f /var/log/nginx/myapp.com.access.log
```

### Advanced: Multiple Backends - Different Routes

```nginx
# Backend 1: Node.js API on port 3000
upstream api_backend {
    server 127.0.0.1:3000;
}

# Backend 2: Python admin on port 5000
upstream admin_backend {
    server 127.0.0.1:5000;
}

server {
    listen 80;
    server_name myapp.com;
    
    # Route /api to Node.js
    location /api {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # Route /admin to Python
    location /admin {
        proxy_pass http://admin_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # Everything else to API
    location / {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Important Headers Explained

```nginx
# The original Host header (important for virtual hosts)
proxy_set_header Host $host;

# The real client IP (backend needs to know who made the request)
proxy_set_header X-Real-IP $remote_addr;

# All IPs in the chain (useful for proxies behind proxies)
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

# Original protocol (http or https)
proxy_set_header X-Forwarded-Proto $scheme;

# Original host
proxy_set_header X-Forwarded-Host $server_name;

# Original port
proxy_set_header X-Forwarded-Port $server_port;
```

---

## 4. Nginx as a Load Balancer

Distribute incoming traffic across multiple backend servers for high availability and performance.

### Scenario:
You have 3 identical backend servers:
- Server 1: 192.168.1.10:3000
- Server 2: 192.168.1.11:3000
- Server 3: 192.168.1.12:3000

Nginx will distribute requests among them.

### Step 1: Create Load Balancer Configuration

```bash
# Create configuration
sudo nano /etc/nginx/sites-available/load-balancer
```

**Paste this configuration:**
```nginx
# Define upstream servers (backend pool)
upstream backend_servers {
    # Default: Round-robin (each server gets equal requests)
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    listen 80;
    listen [::]:80;
    
    server_name myapp.com www.myapp.com;
    
    access_log /var/log/nginx/loadbalancer.access.log;
    error_log /var/log/nginx/loadbalancer.error.log;
    
    location / {
        # Proxy to upstream servers
        proxy_pass http://backend_servers;
        
        # Headers for backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Keep connections alive
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

### Step 2: Load Balancing Methods

Different ways to distribute traffic:

```nginx
# Method 1: Round-robin (DEFAULT - each server gets equal requests)
upstream backend_servers {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
# Request distribution: 1→Server1, 2→Server2, 3→Server3, 4→Server1, 5→Server2...

# Method 2: Least connections (send to server with fewer active connections)
upstream backend_servers {
    least_conn;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
# Good for: Long-running requests, WebSockets

# Method 3: IP Hash (same client IP always goes to same server)
upstream backend_servers {
    ip_hash;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
# Good for: Session persistence, sticky sessions

# Method 4: Weighted round-robin (some servers handle more traffic)
upstream backend_servers {
    server 127.0.0.1:3000 weight=5;   # Gets 50% of requests
    server 127.0.0.1:3001 weight=3;   # Gets 30% of requests
    server 127.0.0.1:3002 weight=2;   # Gets 20% of requests
}
# Good for: Different server capacities

# Method 5: Least time (send to server with fastest response)
upstream backend_servers {
    least_time header;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
# Good for: Real-time optimization
```

### Step 3: Advanced Load Balancer with Health Checks

```nginx
upstream backend_servers {
    # Round-robin with health monitoring
    server 127.0.0.1:3000 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
    
    # Keep connections alive
    keepalive 32;
}

server {
    listen 80;
    server_name myapp.com;
    
    location / {
        proxy_pass http://backend_servers;
        
        # Essential headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Keep-alive
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # Health check endpoint (optional)
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

**Health Check Parameters:**
- `max_fails=3`: Mark server down after 3 failed attempts
- `fail_timeout=30s`: Keep server marked as down for 30 seconds
- `keepalive 32`: Keep 32 idle connections per worker

### Step 4: Enable Load Balancer

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/load-balancer \
    /etc/nginx/sites-enabled/load-balancer

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 5: Monitor Load Balancing

```bash
# Watch requests in real-time
sudo tail -f /var/log/nginx/loadbalancer.access.log

# Count requests per backend server
sudo awk '{print $11}' /var/log/nginx/loadbalancer.access.log | sort | uniq -c
```

---

## 5. SSL/TLS Certificate Setup

Secure your website with HTTPS using free SSL certificates from Let's Encrypt.

### What You'll Get:
- ✓ HTTPS encryption (data in transit)
- ✓ Browser shows "Secure" lock icon
- ✓ Auto-renewal (certificates last 90 days)
- ✓ No cost (Let's Encrypt is free)

### Step 1: Install Certbot (Certificate Manager)

```bash
# Update package manager
sudo apt update

# Install Certbot and Nginx plugin
sudo apt install certbot python3-certbot-nginx

# Verify installation
certbot --version
```

### Step 2: Generate SSL Certificate

**Option A: Automatic (Recommended)**

```bash
# Certbot will modify your Nginx config automatically
sudo certbot --nginx \
    -d mywebsite.com \
    -d www.mywebsite.com
```

**Option B: Manual (More Control)**

```bash
# Generate certificate without modifying config
sudo certbot certonly --nginx \
    -d mywebsite.com \
    -d www.mywebsite.com
```

**During the process:**
- Enter your email address
- Accept terms of service
- Choose to share email (optional)
- Wait for certificate to be issued

### Step 3: Certificate Location

```bash
# List your certificates
sudo certbot certificates

# Certificate files are stored at:
ls -la /etc/letsencrypt/live/mywebsite.com/

# You'll see:
# - cert.pem           (certificate)
# - privkey.pem        (private key)
# - chain.pem          (CA chain)
# - fullchain.pem      (certificate + chain, use this!)
```

### Step 4: Manual Nginx Configuration (If Not Using --nginx)

```bash
# Edit your site configuration
sudo nano /etc/nginx/sites-available/mywebsite.com
```

**Replace with this SSL-enabled configuration:**
```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    
    server_name mywebsite.com www.mywebsite.com;
    
    # Allow Let's Encrypt verification
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    # Redirect all other traffic to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS server block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    server_name mywebsite.com www.mywebsite.com;
    
    # Root directory
    root /var/www/mywebsite.com/html;
    index index.html index.htm;
    
    # SSL Certificate paths (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/mywebsite.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mywebsite.com/privkey.pem;
    
    # SSL Protocol and Ciphers
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # SSL Session caching
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # Logging
    access_log /var/log/nginx/mywebsite.com.access.log;
    error_log /var/log/nginx/mywebsite.com.error.log;
    
    # Main location
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Step 5: Apply and Test

```bash
# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# Check status
sudo systemctl status nginx

# Test in browser: https://mywebsite.com
```

### Step 6: Auto-Renewal Setup

Let's Encrypt certificates expire after 90 days. Certbot handles renewal automatically.

```bash
# Check auto-renewal status
sudo systemctl status certbot.timer

# View renewal configuration
sudo cat /etc/letsencrypt/renewal/mywebsite.com.conf

# Test renewal (dry run, doesn't actually renew)
sudo certbot renew --dry-run

# Manual renewal if needed
sudo certbot renew

# View renewal logs
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

### Step 7: SSL + Reverse Proxy Combined

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name myapp.com;
    return 301 https://$server_name$request_uri;
}

# Define backend
upstream backend {
    server 127.0.0.1:3000;
}

# HTTPS with reverse proxy
server {
    listen 443 ssl http2;
    server_name myapp.com;
    
    # SSL certificates
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    
    # Strong SSL
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Verify SSL Certificate

```bash
# Check certificate details
sudo openssl x509 -in /etc/letsencrypt/live/mywebsite.com/fullchain.pem -text -noout

# Check expiration date
sudo openssl x509 -in /etc/letsencrypt/live/mywebsite.com/fullchain.pem -noout -dates

# Test SSL with online tools
# https://www.ssllabs.com/ssltest/ (external service)
```

---

## 6. DNS Configuration

DNS maps your domain name (mywebsite.com) to your server's IP address.

### Understanding DNS Flow

```
User types: mywebsite.com in browser
    ↓
Browser queries DNS resolver: "What's the IP of mywebsite.com?"
    ↓
DNS resolver checks:
  1. Root nameserver
  2. TLD nameserver (.com nameserver)
  3. Domain nameserver (your registrar)
    ↓
Domain nameserver returns: 192.168.1.100
    ↓
Browser connects to: 192.168.1.100 (your Nginx server)
    ↓
Nginx serves website
```

### Step 1: Find Your Server's Public IP

```bash
# Method 1: Using curl
curl https://api.ipify.org
# Output: 192.168.1.100

# Method 2: Using dig
dig +short myip.opendns.com @resolver1.opendns.com

# Method 3: Check AWS/hosting provider dashboard
# DigitalOcean, Linode, AWS, Azure, etc. show your IP
```

### Step 2: DNS Records You Need

**A Record** - Maps domain to IPv4 address (most common):
```
Type:    A
Name:    @  (represents mywebsite.com)
Value:   192.168.1.100
TTL:     3600 (seconds)
```

**CNAME Record** - Creates alias for another domain:
```
Type:    CNAME
Name:    www
Points to: mywebsite.com
TTL:     3600
```

**AAAA Record** - Maps domain to IPv6 address (optional):
```
Type:    AAAA
Name:    @
Value:   2001:0db8:85a3:0000:0000:8a2e:0370:7334
TTL:     3600
```

**MX Record** - Email server (if using email):
```
Type:       MX
Name:       @
Value:      mail.mywebsite.com
Priority:   10 (lower = higher priority)
TTL:        3600
```

### Step 3: Configure DNS at Domain Registrar

**Popular Ubuntu-friendly registrars:**
- Namecheap
- GoDaddy
- Route53 (AWS)
- Cloudflare
- DigitalOcean

#### Example: Namecheap

1. Login to Namecheap
2. Go to Dashboard → Domain List → Manage
3. Click "Manage Domain"
4. Go to "Advanced DNS" tab
5. Add these records:

```
Type   | Host | Value               | TTL
-------|------|---------------------|------
A      | @    | 192.168.1.100       | 3600
A      | www  | 192.168.1.100       | 3600
CNAME  | www  | mywebsite.com       | 3600
```

6. Click "Save All Records"
7. Changes take effect in 5 minutes to 24 hours

#### Example: Route53 (AWS)

1. Login to AWS Console
2. Go to Route 53 → Hosted Zones
3. Click your domain
4. Create Record Set:
   - Name: mywebsite.com
   - Type: A
   - Value: 192.168.1.100
   - TTL: 300

### Step 4: Verify DNS Resolution

```bash
# Method 1: nslookup (simple)
nslookup mywebsite.com

# Output should show your IP:
# Non-authoritative answer:
# Name:    mywebsite.com
# Address: 192.168.1.100

# Method 2: dig (detailed)
dig mywebsite.com

# Method 3: getent
getent hosts mywebsite.com

# Method 4: host command
host mywebsite.com

# Wait 24 hours for DNS propagation
# Check status at: https://whatsmydns.net/
```

### Step 5: Update Nginx Configuration

Make sure your Nginx config includes the correct domains:

```bash
# Edit your site config
sudo nano /etc/nginx/sites-available/mywebsite.com
```

**Verify server_name includes both versions:**
```nginx
server {
    server_name mywebsite.com www.mywebsite.com;
    # ... rest of config
}
```

```bash
# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

### Step 6: Using Cloudflare (Advanced)

Cloudflare provides DNS, DDoS protection, and CDN.

**Setup:**
1. Go to https://cloudflare.com
2. Add site → Enter mywebsite.com
3. Choose Free plan
4. Update nameservers at your registrar to Cloudflare's:
   - `grace.ns.cloudflare.com`
   - `oscar.ns.cloudflare.com`
5. Add DNS records in Cloudflare dashboard

**In Nginx, add Cloudflare header to get real client IP:**
```nginx
location / {
    proxy_set_header X-Forwarded-For $http_cf_connecting_ip;
}
```

### DNS Troubleshooting

```bash
# Check if DNS is propagated globally
# Visit: https://whatsmydns.net/ and enter your domain

# Flush local DNS cache on Ubuntu
sudo systemctl restart systemd-resolved

# Check if domain resolves to your IP
nslookup mywebsite.com

# Check all DNS records
dig mywebsite.com ANY

# Check specific record type
dig mywebsite.com A
dig mywebsite.com MX
dig mywebsite.com CNAME

# Check nameserver
dig mywebsite.com NS

# Trace DNS lookup path
dig mywebsite.com +trace
```

---

## 7. Common Commands & Troubleshooting

### Essential Nginx Commands on Ubuntu

```bash
# Test configuration syntax (ALWAYS do this before reload)
sudo nginx -t

# Reload Nginx (graceful, no downtime)
sudo systemctl reload nginx

# Restart Nginx (stops then starts)
sudo systemctl restart nginx

# Stop Nginx
sudo systemctl stop nginx

# Start Nginx
sudo systemctl start nginx

# Check Nginx status
sudo systemctl status nginx

# Check Nginx version
nginx -v

# Check full version info
nginx -V

# Check if Nginx is running
ps aux | grep nginx
```

### Site Configuration Management

```bash
# Enable a site (create symlink from sites-available to sites-enabled)
sudo ln -s /etc/nginx/sites-available/mysite \
    /etc/nginx/sites-enabled/mysite

# Disable a site (remove symlink)
sudo rm /etc/nginx/sites-enabled/mysite

# List enabled sites
ls -la /etc/nginx/sites-enabled/

# List available sites
ls -la /etc/nginx/sites-available/

# Edit site configuration
sudo nano /etc/nginx/sites-available/mysite
```

### Viewing and Analyzing Logs

```bash
# Real-time access log
sudo tail -f /var/log/nginx/access.log

# Real-time error log
sudo tail -f /var/log/nginx/error.log

# View last 50 lines
sudo tail -50 /var/log/nginx/access.log

# View entire log file
sudo cat /var/log/nginx/access.log

# Search for specific pattern
sudo grep "error" /var/log/nginx/error.log
sudo grep "192.168.1.1" /var/log/nginx/access.log

# Count HTTP status codes
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Count requests per IP
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Count requests per domain
sudo awk '{print $11}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Real-time monitoring
sudo tail -f /var/log/nginx/access.log | grep "GET"

# Log rotation
sudo logrotate -f /etc/logrotate.d/nginx
```

### Troubleshooting Common Issues

#### Issue 1: "nginx: [emerg] unexpected end of file"

**Cause:** Syntax error in configuration

**Solution:**
```bash
# Test to see more details
sudo nginx -t

# Common mistakes:
# - Missing semicolon (;) at end of line
# - Missing closing brace (})
# - Missing quotes around values
# - Typos in directives

# Edit the config file
sudo nano /etc/nginx/sites-available/yoursite

# Look for missing ; or }
# Save and test again
sudo nginx -t
```

#### Issue 2: "Address already in use" (Port 80 in use)

**Cause:** Port 80 is taken by another process

**Solution:**
```bash
# Find what's using port 80
sudo lsof -i :80

# Or use netstat
sudo netstat -tlnp | grep :80

# Kill the process if it's not Nginx
sudo kill -9 <PID>

# Or use different port temporarily (for testing)
# Edit config, change: listen 8080;
# Then test on http://localhost:8080
```

#### Issue 3: "502 Bad Gateway" Error

**Cause:** Backend server not responding

**Solution:**
```bash
# Check if backend is running
sudo netstat -tlnp | grep :3000

# Or check specific port
curl http://127.0.0.1:3000

# If backend is not running, start it
cd /home/ubuntu/myapp
node app.js

# Check Nginx error log for details
sudo tail -f /var/log/nginx/error.log

# Verify backend address in config
sudo cat /etc/nginx/sites-enabled/mysite | grep proxy_pass
```

#### Issue 4: "403 Forbidden" Error

**Cause:** Permission denied

**Solution:**
```bash
# Check file permissions
ls -la /var/www/mysite/

# File permissions should be:
# Directories: 755
# Files: 644

# Fix permissions
sudo chown -R www-data:www-data /var/www/mysite
sudo chmod -R 755 /var/www/mysite/
sudo chmod 644 /var/www/mysite/html/*

# Check log
sudo tail -f /var/log/nginx/error.log
```

#### Issue 5: "Connection refused" on Reverse Proxy

**Cause:** Backend not running or wrong port

**Solution:**
```bash
# Check what's listening on the backend port
sudo lsof -i :3000

# If nothing listening, start backend:
cd /home/ubuntu/myapp
node app.js &

# Verify it's listening
curl http://127.0.0.1:3000

# Test Nginx config
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# Check error log
sudo tail -f /var/log/nginx/error.log
```

#### Issue 6: SSL Certificate Errors

**Problem:** Browser shows SSL error

**Solution:**
```bash
# Check certificate exists
ls -la /etc/letsencrypt/live/mysite.com/

# Check certificate expiration
sudo openssl x509 -in /etc/letsencrypt/live/mysite.com/fullchain.pem -noout -dates

# Verify paths in Nginx config
sudo grep ssl_certificate /etc/nginx/sites-enabled/mysite

# Renew certificate
sudo certbot renew

# Force renewal if needed
sudo certbot renew --force-renewal

# Check renewal logs
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

#### Issue 7: High CPU/Memory Usage

**Solution:**
```bash
# Check Nginx processes
ps aux | grep nginx

# Check worker configuration
sudo cat /etc/nginx/nginx.conf | grep worker_processes

# Optimal: set to number of CPU cores
nproc  # shows number of cores

# Edit main config
sudo nano /etc/nginx/nginx.conf

# Change to:
worker_processes auto;  # Automatically detects CPU cores

# Also check:
worker_connections 1024;  # connections per worker

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

#### Issue 8: Domain Not Resolving

**Solution:**
```bash
# Check DNS resolution
nslookup mysite.com

# If not resolving:
# 1. Check DNS records at registrar
# 2. Wait 24 hours for propagation
# 3. Flush DNS cache:
sudo systemctl restart systemd-resolved

# 4. Check DNS with dig
dig mysite.com

# 5. Check nameservers
dig mysite.com NS

# 6. Try from public DNS (8.8.8.8)
nslookup mysite.com 8.8.8.8
```

---

## Production Setup Example

Complete configuration combining everything.

### Scenario:
- Domain: `api.example.com`
- Backend: 3 Node.js servers (load balanced)
- SSL with Let's Encrypt
- Rate limiting
- Security headers

### Configuration File

```bash
# Create the config
sudo nano /etc/nginx/sites-available/api.example.com
```

**Paste this:**
```nginx
# Rate limiting zone
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

# Define upstream servers (load balancing)
upstream backend_servers {
    least_conn;
    server 192.168.1.10:3000 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:3000 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:3000 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name api.example.com;
    
    # Let's Encrypt verification
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    # Redirect to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS server with load balancing
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    server_name api.example.com;
    
    # SSL Certificate
    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    
    # Strong SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Logging
    access_log /var/log/nginx/api.example.com.access.log combined;
    error_log /var/log/nginx/api.example.com.error.log warn;
    
    # Client max body size
    client_max_body_size 10M;
    
    # Rate limiting
    limit_req zone=api_limit burst=20 nodelay;
    
    # Root location - proxy with load balancing
    location / {
        proxy_pass http://backend_servers;
        
        # Headers for backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
        
        # Connection settings
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    # Block access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
    
    # Block access to config files
    location ~ \.(conf|json|env|pem)$ {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

### Deploy Production Configuration

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/api.example.com \
    /etc/nginx/sites-enabled/

# Remove default site
sudo rm /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# Check status
sudo systemctl status nginx

# Monitor logs
sudo tail -f /var/log/nginx/api.example.com.access.log
```

---

## Quick Reference Cheat Sheet

```bash
# ===== INSTALLATION =====
sudo apt update && sudo apt upgrade
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# ===== CREATE WEBSITE =====
sudo mkdir -p /var/www/mysite.com/html
sudo nano /var/www/mysite.com/html/index.html
sudo chown -R www-data:www-data /var/www/mysite.com
sudo chmod -R 755 /var/www/mysite.com

# ===== CREATE NGINX CONFIG =====
sudo nano /etc/nginx/sites-available/mysite.com
sudo ln -s /etc/nginx/sites-available/mysite.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# ===== SSL CERTIFICATE =====
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d mysite.com -d www.mysite.com
sudo certbot renew --dry-run

# ===== MONITORING =====
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
sudo systemctl status nginx

# ===== TROUBLESHOOTING =====
sudo nginx -t                           # Test syntax
sudo lsof -i :80                        # Check port 80
ps aux | grep nginx                    # Check processes
sudo systemctl restart nginx           # Full restart
```

---

## Key Concepts Summary

| Concept | Purpose | Example |
|---------|---------|---------|
| **Web Server** | Serves static files (HTML, CSS, JS) | Serve website from `/var/www/html` |
| **Reverse Proxy** | Forwards requests to backend | Forward to Node.js on port 3000 |
| **Load Balancer** | Distributes across multiple servers | 3 backends with round-robin |
| **SSL/TLS** | Encrypts traffic (HTTPS) | Let's Encrypt certificate |
| **DNS** | Maps domain to IP | mysite.com → 192.168.1.100 |
| **Upstream** | Group of backend servers | Define servers to load balance to |
| **Location** | URL pattern matching | `/api`, `/admin`, `/static` |
| **Proxy Headers** | Pass client info to backend | `X-Real-IP`, `X-Forwarded-For` |

---

## Resources

- **Official Nginx Docs**: https://nginx.org/en/docs/
- **Let's Encrypt**: https://letsencrypt.org/
- **Certbot Ubuntu**: https://certbot.eff.org/instructions?os=ubuntu
- **Mozilla SSL Generator**: https://ssl-config.mozilla.org/
- **Nginx Pitfalls**: https://nginx.org/en/docs/pitfalls.html

---

## Next Steps

1. **Start with Web Server** - Get your first static site running
2. **Add SSL** - Secure your website with HTTPS
3. **Try Reverse Proxy** - Forward to your backend app
4. **Scale with Load Balancer** - Distribute across multiple servers
5. **Monitor & Optimize** - Watch logs, tune performance

Good luck! 🚀
