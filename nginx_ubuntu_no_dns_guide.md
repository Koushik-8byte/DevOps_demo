# Complete Nginx Guide for Ubuntu (WITHOUT DNS): Web Server, Reverse Proxy, Load Balancer & SSL

## Table of Contents
1. [Nginx Basics & Installation](#1-nginx-basics--installation)
2. [Testing Without DNS](#2-testing-without-dns)
3. [Nginx as a Web Server (No DNS)](#3-nginx-as-a-web-server-no-dns)
4. [Nginx as a Reverse Proxy (No DNS)](#4-nginx-as-a-reverse-proxy-no-dns)
5. [Nginx as a Load Balancer (No DNS)](#5-nginx-as-a-load-balancer-no-dns)
6. [SSL/TLS Certificate Setup (No DNS)](#6-ssltls-certificate-setup-no-dns)
7. [Using /etc/hosts for Local Testing](#7-using-etchosts-for-local-testing)
8. [Common Commands & Troubleshooting](#8-common-commands--troubleshooting)

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

# Test connection locally
curl http://localhost
curl http://127.0.0.1
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

---

## 2. Testing Without DNS

When you don't have DNS configured, you have several options to test your Nginx setup:

### Option 1: Using localhost (127.0.0.1)

Best for: Local development, testing on the same machine

```bash
# Access via localhost
curl http://localhost
curl http://127.0.0.1
curl http://localhost:8080

# In browser
# http://localhost
# http://localhost:8080
```

### Option 2: Using Server IP Address

Best for: Testing from other machines on your network

```bash
# Find your server's IP address
hostname -I

# Example output: 192.168.1.100

# Access from any machine on network
curl http://192.168.1.100
curl http://192.168.1.100:8080

# In browser
# http://192.168.1.100
# http://192.168.1.100:8080
```

### Option 3: Using /etc/hosts File (Local DNS Override)

Best for: Testing domain names locally without actual DNS

```bash
# Edit /etc/hosts file
sudo nano /etc/hosts

# Add these lines:
127.0.0.1       localhost mywebsite.local
127.0.0.1       api.local
192.168.1.100   myserver.local
```

**Then access locally:**
```bash
curl http://mywebsite.local
curl http://api.local
# Works like real DNS but only on your machine!
```

### Option 4: Using Port Numbers

Best for: Running multiple services on one server

```bash
# Listen on different ports
listen 80;       # Service 1
listen 8080;     # Service 2
listen 8081;     # Service 3
listen 3000;     # Service 4

# Access each service
curl http://localhost:80
curl http://localhost:8080
curl http://localhost:8081
curl http://localhost:3000

# Or with IP
curl http://192.168.1.100:80
curl http://192.168.1.100:8080
```

### Option 5: Using URL Paths (Same Port, Different Routes)

Best for: Multiple services on same port

```bash
# All on port 80, different URL paths
location /api { ... }
location /admin { ... }
location /static { ... }
location / { ... }

# Access each
curl http://localhost/api
curl http://localhost/admin
curl http://localhost/static
curl http://localhost/
```

---

## 3. Nginx as a Web Server (No DNS)

A web server serves static files (HTML, CSS, JS, images) directly to clients.

### Step 1: Create Website Directory

```bash
# Create directory structure
sudo mkdir -p /var/www/mywebsite/html

# Create a simple index.html file
sudo nano /var/www/mywebsite/html/index.html
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
        h1 { color: #333; }
        .info { background: white; padding: 15px; border-radius: 5px; }
    </style>
</head>
<body>
    <h1>✓ Nginx is Working!</h1>
    <div class="info">
        <p>Your website is running on Nginx</p>
        <p><strong>Server:</strong> Ubuntu</p>
        <p><strong>Access via:</strong> http://localhost or http://YOUR_IP</p>
    </div>
</body>
</html>
```

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 2: Set Permissions

```bash
# Change owner to www-data (Nginx user on Ubuntu)
sudo chown -R www-data:www-data /var/www/mywebsite

# Set directory permissions
sudo chmod -R 755 /var/www/mywebsite

# Set file permissions
sudo chmod 644 /var/www/mywebsite/html/*

# Verify ownership
ls -la /var/www/mywebsite/
```

### Step 3: Create Nginx Server Block Configuration

```bash
# Create configuration file
sudo nano /etc/nginx/sites-available/default-site
```

**Paste this configuration (NO domain names needed):**
```nginx
server {
    # Listen on port 80 (can use any port 1024+)
    listen 80;
    listen [::]:80;
    
    # NO server_name needed! Nginx will serve this for ANY hostname/IP
    # OR you can specify localhost
    # server_name localhost 127.0.0.1 192.168.1.100;
    
    # Root directory where files are stored
    root /var/www/mywebsite/html;
    
    # Default index files to serve
    index index.html index.htm;
    
    # Log files
    access_log /var/log/nginx/mywebsite.access.log;
    error_log /var/log/nginx/mywebsite.error.log;
    
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
# Remove default site if it exists
sudo rm /etc/nginx/sites-enabled/default

# Create symbolic link to enable the site
sudo ln -s /etc/nginx/sites-available/default-site \
    /etc/nginx/sites-enabled/default-site

# Verify the symlink was created
ls -la /etc/nginx/sites-enabled/
```

### Step 5: Test Configuration

```bash
# Test Nginx configuration syntax
sudo nginx -t

# You should see:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Step 6: Apply Configuration

```bash
# Reload Nginx (no downtime)
sudo systemctl reload nginx

# Check status
sudo systemctl status nginx
```

### Step 7: Test in Browser or with curl

```bash
# Option 1: Test with localhost
curl http://localhost
curl http://127.0.0.1

# Option 2: Test with server IP
# First get your IP:
hostname -I

# Then test:
curl http://192.168.1.100    # Replace with your IP

# Option 3: Test with specific port
curl http://localhost:80
curl http://192.168.1.100:80

# Option 4: In web browser
# http://localhost
# http://127.0.0.1
# http://192.168.1.100  (replace with your IP)
```

### Alternative: Listen on Specific Ports

If port 80 is already in use, use a different port:

```nginx
server {
    # Use port 8080 instead of 80
    listen 8080;
    listen [::]:8080;
    
    root /var/www/mywebsite/html;
    index index.html index.htm;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Then access via:**
```bash
curl http://localhost:8080
curl http://192.168.1.100:8080
```

---

## 4. Nginx as a Reverse Proxy (No DNS)

A reverse proxy forwards client requests to backend servers and returns responses.

### Scenario:
- Nginx listens on port 80
- Your Node.js/Python/Java app runs on port 3000 (localhost)
- Nginx forwards requests to the backend

### Step 1: Start Your Backend Application

```bash
# Example: Node.js app listening on localhost:3000

# Navigate to your app directory
cd /home/ubuntu/myapp

# Start the app (it should listen on port 3000)
node app.js

# You should see: "Server listening on port 3000"

# Or start in background
node app.js &

# Or use nohup to keep running even if terminal closes
nohup node app.js &
```

**Verify backend is running:**
```bash
# Check if port 3000 is listening
sudo lsof -i :3000

# Or test with curl
curl http://127.0.0.1:3000
curl http://localhost:3000
```

### Step 2: Create Reverse Proxy Configuration

```bash
# Create configuration
sudo nano /etc/nginx/sites-available/reverse-proxy
```

**Paste this configuration:**
```nginx
# Define upstream backend (no domain needed)
upstream backend {
    server 127.0.0.1:3000;      # localhost:3000
}

server {
    # Listen on port 80
    listen 80;
    listen [::]:80;
    
    # Can specify any hostname or leave blank for all
    # server_name localhost 127.0.0.1;
    
    # Or no server_name at all - will serve any request
    
    # Access and error logs
    access_log /var/log/nginx/reverse-proxy.access.log;
    error_log /var/log/nginx/reverse-proxy.error.log;
    
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
}
```

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 3: Enable the Site

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/reverse-proxy \
    /etc/nginx/sites-enabled/reverse-proxy

# Remove other sites if causing conflicts
sudo rm /etc/nginx/sites-enabled/default-site

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 4: Test the Reverse Proxy

```bash
# Assuming backend is running on port 3000

# Test with localhost
curl http://localhost
curl http://127.0.0.1

# Test with server IP
curl http://192.168.1.100

# Monitor logs
sudo tail -f /var/log/nginx/reverse-proxy.access.log

# Should see requests coming through
```

### Advanced: Multiple Backends - Different Routes (No DNS)

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
    
    # Route /api to Node.js
    location /api {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # Route /admin to Python
    location /admin {
        proxy_pass http://admin_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # Everything else to API
    location / {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Test different routes:**
```bash
curl http://localhost/api
curl http://localhost/admin
curl http://localhost/
```

---

## 5. Nginx as a Load Balancer (No DNS)

Distribute incoming traffic across multiple backend servers.

### Scenario:
3 backend servers on localhost with different ports:
- Server 1: 127.0.0.1:3000
- Server 2: 127.0.0.1:3001
- Server 3: 127.0.0.1:3002

### Step 1: Start Multiple Backend Servers

```bash
# Terminal 1: Start backend 1
cd /home/ubuntu/myapp
node app.js --port 3000 &

# Terminal 2: Start backend 2
cd /home/ubuntu/myapp
node app.js --port 3001 &

# Terminal 3: Start backend 3
cd /home/ubuntu/myapp
node app.js --port 3002 &

# Verify all are running
sudo lsof -i :3000
sudo lsof -i :3001
sudo lsof -i :3002

# Or test with curl
curl http://127.0.0.1:3000
curl http://127.0.0.1:3001
curl http://127.0.0.1:3002
```

### Step 2: Create Load Balancer Configuration

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
    
    # No server_name needed for testing
    
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

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 3: Load Balancing Methods (No DNS)

Different ways to distribute traffic:

```nginx
# Method 1: Round-robin (DEFAULT - each server gets equal requests)
upstream backend_servers {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
# Distribution: Request 1→3000, 2→3001, 3→3002, 4→3000, etc.

# Method 2: Least connections
upstream backend_servers {
    least_conn;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

# Method 3: IP Hash (same client always goes to same server)
upstream backend_servers {
    ip_hash;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

# Method 4: Weighted round-robin
upstream backend_servers {
    server 127.0.0.1:3000 weight=5;   # Gets 50%
    server 127.0.0.1:3001 weight=3;   # Gets 30%
    server 127.0.0.1:3002 weight=2;   # Gets 20%
}
```

### Step 4: Enable and Test Load Balancer

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/load-balancer \
    /etc/nginx/sites-enabled/load-balancer

# Remove conflicting sites
sudo rm /etc/nginx/sites-enabled/default-site
sudo rm /etc/nginx/sites-enabled/reverse-proxy

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 5: Monitor Load Balancing

```bash
# Watch requests going to each backend in real-time
sudo tail -f /var/log/nginx/loadbalancer.access.log

# Make requests in a loop to see load balancing
for i in {1..10}; do curl http://localhost; echo ""; done

# Count requests per backend
sudo awk '{print $11}' /var/log/nginx/loadbalancer.access.log | sort | uniq -c
```

---

## 6. SSL/TLS Certificate Setup (No DNS)

Self-signed certificates for local testing (no Let's Encrypt needed without DNS).

### Important Notes on SSL Without DNS:

- **Let's Encrypt requires DNS** (they verify domain ownership via DNS)
- **Self-signed certificates work for testing** but browser shows warning
- **Use for development/testing only**, not production

### Step 1: Generate Self-Signed Certificate

```bash
# Create directory for certificates
sudo mkdir -p /etc/nginx/ssl

# Generate private key (2048-bit RSA)
sudo openssl genrsa -out /etc/nginx/ssl/private.key 2048

# Generate certificate (valid for 365 days)
sudo openssl req -new -x509 -key /etc/nginx/ssl/private.key \
    -out /etc/nginx/ssl/certificate.crt -days 365

# When prompted, enter details (can press Enter to skip):
# Country Name (2 letter code) [AU]: US
# State or Province Name [Some-State]: California
# Locality Name (eg, city) []: San Francisco
# Organization Name (eg, company) [Internet Widgits Pty Ltd]: My Company
# Organizational Unit Name (eg, section) []: IT
# Common Name (eg, your name or your server hostname) []: localhost
# Email Address []: your@email.com

# Verify certificate was created
ls -la /etc/nginx/ssl/
```

### Step 2: Create HTTPS Configuration

```bash
# Create configuration with HTTPS
sudo nano /etc/nginx/sites-available/https-local
```

**Paste this configuration:**
```nginx
# Redirect HTTP to HTTPS (optional)
server {
    listen 80;
    listen [::]:80;
    
    # Redirect all traffic to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    
    # Can leave server_name empty for testing
    # server_name localhost 127.0.0.1;
    
    # Root directory
    root /var/www/mywebsite/html;
    index index.html index.htm;
    
    # SSL Certificate paths (self-signed)
    ssl_certificate /etc/nginx/ssl/certificate.crt;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Logging
    access_log /var/log/nginx/https.access.log;
    error_log /var/log/nginx/https.error.log;
    
    # Main location
    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 3: Enable HTTPS Configuration

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/https-local \
    /etc/nginx/sites-enabled/https-local

# Remove other conflicting sites
sudo rm /etc/nginx/sites-enabled/* || true

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 4: Test HTTPS (Ignore Certificate Warning)

```bash
# Test with curl (ignore self-signed warning)
curl -k https://localhost
curl -k https://127.0.0.1
curl -k https://192.168.1.100

# In web browser: https://localhost
# Browser will show warning (click "Advanced" → "Proceed")

# Or use curl with more details
curl -kv https://localhost

# Check certificate details
sudo openssl x509 -in /etc/nginx/ssl/certificate.crt -text -noout
```

### Step 5: SSL + Reverse Proxy Combined (No DNS)

```nginx
# Backend running on localhost:3000
upstream backend {
    server 127.0.0.1:3000;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    return 301 https://$server_name$request_uri;
}

# HTTPS with reverse proxy
server {
    listen 443 ssl;
    
    ssl_certificate /etc/nginx/ssl/certificate.crt;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Test:**
```bash
curl -k https://localhost
curl -k https://127.0.0.1
```

---

## 7. Using /etc/hosts for Local Testing

The `/etc/hosts` file acts like a local DNS server on your machine.

### Step 1: Edit /etc/hosts File

```bash
# Open hosts file with sudo
sudo nano /etc/hosts
```

**You'll see something like:**
```
127.0.0.1       localhost
::1             localhost

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
```

### Step 2: Add Your Custom Entries

**Add these lines at the end:**
```
# Local testing entries
127.0.0.1       mywebsite.local
127.0.0.1       api.local
127.0.0.1       admin.local
192.168.1.100   myserver.local
192.168.1.100   app.local
```

**Complete example:**
```
127.0.0.1       localhost
::1             localhost

# Local development entries
127.0.0.1       mywebsite.local
127.0.0.1       api.local
127.0.0.1       admin.local
127.0.0.1       app.test
192.168.1.100   myserver.local
192.168.1.100   app.local
```

**Save file:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 3: Test /etc/hosts Entries

```bash
# Test with ping
ping mywebsite.local
ping api.local

# Test with curl
curl http://mywebsite.local
curl http://api.local
curl http://192.168.1.100

# Test with nslookup (local resolution)
nslookup mywebsite.local

# In web browser
# http://mywebsite.local
# http://api.local
```

### Step 4: Update Nginx Config to Use /etc/hosts Names

```bash
# Create configuration
sudo nano /etc/nginx/sites-available/with-hosts
```

**Use domain names from /etc/hosts:**
```nginx
upstream backend {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    
    # Use names from /etc/hosts
    server_name mywebsite.local api.local app.test localhost 127.0.0.1;
    
    root /var/www/mywebsite/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}

# API proxy
server {
    listen 80;
    server_name api.local;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Enable and test:**
```bash
sudo ln -s /etc/nginx/sites-available/with-hosts /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# Test
curl http://mywebsite.local
curl http://api.local
```

### Step 5: /etc/hosts with HTTPS + Self-Signed Certs

```nginx
server {
    listen 443 ssl;
    server_name mywebsite.local www.mywebsite.local;
    
    ssl_certificate /etc/nginx/ssl/certificate.crt;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    
    root /var/www/mywebsite/html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 80;
    server_name mywebsite.local www.mywebsite.local;
    return 301 https://$server_name$request_uri;
}
```

**Test:**
```bash
curl -k https://mywebsite.local
# Browser: https://mywebsite.local (click through SSL warning)
```

---

## 8. Common Commands & Troubleshooting

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

### Checking What's Listening on Ports

```bash
# Check all listening ports
sudo ss -tlnp

# Check specific port
sudo lsof -i :80
sudo lsof -i :8080
sudo lsof -i :3000

# Find what's using a port
sudo netstat -tlnp | grep :80
```

### Site Configuration Management

```bash
# Enable a site
sudo ln -s /etc/nginx/sites-available/mysite \
    /etc/nginx/sites-enabled/mysite

# Disable a site
sudo rm /etc/nginx/sites-enabled/mysite

# List enabled sites
ls -la /etc/nginx/sites-enabled/

# List available sites
ls -la /etc/nginx/sites-available/

# Edit site configuration
sudo nano /etc/nginx/sites-available/mysite
```

### Viewing Logs

```bash
# Real-time access log
sudo tail -f /var/log/nginx/access.log

# Real-time error log
sudo tail -f /var/log/nginx/error.log

# Last 50 lines
sudo tail -50 /var/log/nginx/access.log

# View entire file
sudo cat /var/log/nginx/access.log

# Search logs
sudo grep "error" /var/log/nginx/error.log
sudo grep "127.0.0.1" /var/log/nginx/access.log

# Count HTTP status codes
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Count requests per IP
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
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
# - Typos in directives

# Edit config
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

# Kill the process if it's not Nginx
sudo kill -9 <PID>

# Or use different port temporarily
# Edit config, change: listen 8080;
# Then test on http://localhost:8080
```

#### Issue 3: "502 Bad Gateway" Error

**Cause:** Backend server not responding

**Solution:**
```bash
# Check if backend is running
sudo lsof -i :3000

# Test backend directly
curl http://127.0.0.1:3000

# If backend not running, start it
cd /home/ubuntu/myapp
node app.js &

# Check Nginx error log
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
# Check if backend is listening
sudo lsof -i :3000

# If not, start backend:
cd /home/ubuntu/myapp
node app.js &

# Verify it's listening
curl http://127.0.0.1:3000

# Test Nginx config
sudo nginx -t

# Reload
sudo systemctl reload nginx

# Check error log
sudo tail -f /var/log/nginx/error.log
```

#### Issue 6: SSL Certificate Errors

**Problem:** Browser shows SSL error

**Solution:**
```bash
# Check certificate exists
ls -la /etc/nginx/ssl/

# Check certificate details
sudo openssl x509 -in /etc/nginx/ssl/certificate.crt -text -noout

# Check expiration
sudo openssl x509 -in /etc/nginx/ssl/certificate.crt -noout -dates

# Verify paths in Nginx config
sudo grep ssl_certificate /etc/nginx/sites-enabled/yoursite

# Regenerate if expired
sudo openssl req -new -x509 -key /etc/nginx/ssl/private.key \
    -out /etc/nginx/ssl/certificate.crt -days 365
```

#### Issue 7: Hosts File Not Working

**Problem:** Can't access mysite.local even after adding to /etc/hosts

**Solution:**
```bash
# Verify entry is in /etc/hosts
cat /etc/hosts

# Flush DNS cache
sudo systemd-resolve --flush-caches

# Or restart DNS resolver
sudo systemctl restart systemd-resolved

# Test resolution
nslookup mysite.local
ping mysite.local

# Try accessing again
curl http://mysite.local
```

#### Issue 8: Can't Access from Different Machine

**Problem:** Can access from localhost but not from other machine

**Solution:**
```bash
# Check if Nginx is listening on all interfaces
sudo ss -tlnp | grep nginx

# Should show:
# LISTEN    0    128    *:80    # Listening on all IPs
# NOT:
# LISTEN    0    128    127.0.0.1:80    # Only localhost

# If only on localhost, edit config:
sudo nano /etc/nginx/sites-available/yoursite

# Change:
# listen 127.0.0.1:80;    # Wrong (only localhost)
# To:
# listen 80;              # Correct (all IPs)

# Test and reload
sudo nginx -t
sudo systemctl reload nginx

# Then access from other machine using server IP
curl http://192.168.1.100
```

---

## Quick Start Guide (No DNS)

### Quick Start: Just Want to Serve a Website

```bash
# 1. Install Nginx
sudo apt install nginx
sudo systemctl start nginx

# 2. Create website
sudo mkdir -p /var/www/mysite/html
echo "<h1>Hello</h1>" | sudo tee /var/www/mysite/html/index.html

# 3. Create config
sudo nano /etc/nginx/sites-available/default-site

# Paste:
server {
    listen 80;
    root /var/www/mysite/html;
    index index.html;
}

# 4. Enable and test
sudo ln -s /etc/nginx/sites-available/default-site /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 5. Access
curl http://localhost      # or http://127.0.0.1 or http://YOUR_IP
```

### Quick Start: Reverse Proxy to Backend

```bash
# 1. Start backend on port 3000
cd /path/to/app
node app.js &

# 2. Create reverse proxy config
sudo nano /etc/nginx/sites-available/proxy

# Paste:
upstream backend {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# 3. Enable
sudo ln -s /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 4. Test
curl http://localhost
```

### Quick Start: Load Balancing 3 Backends

```bash
# 1. Start 3 backends
cd /path/to/app
node app.js --port 3000 &
node app.js --port 3001 &
node app.js --port 3002 &

# 2. Create load balancer config
sudo nano /etc/nginx/sites-available/lb

# Paste:
upstream backend {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# 3. Enable
sudo ln -s /etc/nginx/sites-available/lb /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 4. Test
for i in {1..10}; do curl http://localhost; done
```

### Quick Start: HTTPS with Self-Signed Cert

```bash
# 1. Generate certificate
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/private.key \
    -out /etc/nginx/ssl/certificate.crt

# 2. Create HTTPS config
sudo nano /etc/nginx/sites-available/secure

# Paste:
server {
    listen 443 ssl;
    root /var/www/mysite/html;
    
    ssl_certificate /etc/nginx/ssl/certificate.crt;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    
    location / {
        try_files $uri =404;
    }
}

server {
    listen 80;
    return 301 https://$server_name$request_uri;
}

# 3. Enable
sudo ln -s /etc/nginx/sites-available/secure /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 4. Test
curl -k https://localhost
```

---

## Important Notes for Testing Without DNS

### Accessing from localhost:
```bash
curl http://localhost
curl http://127.0.0.1
# Works only on the server machine
```

### Accessing from another machine:
```bash
# On different machine, use server IP
curl http://192.168.1.100    # Replace with your server IP
# Requires: listen 80; (not listen 127.0.0.1:80;)
```

### Using /etc/hosts:
```bash
# Add to /etc/hosts on each machine that needs it
# Only works on that specific machine
sudo nano /etc/hosts
# Add: 127.0.0.1    mysite.local
```

### Checking Your IP:
```bash
# Find your server's IP
hostname -I

# Example output: 192.168.1.100
# Then access from other machine: http://192.168.1.100
```

### Difference Between:
```
listen 80;              # Listens on ALL IPs (0.0.0.0)
listen 127.0.0.1:80;   # Listens ONLY on localhost
listen 192.168.1.100;  # Listens ONLY on that specific IP

# For testing across network, always use:
listen 80;             # Without specifying an IP
```

---

## Commands Summary for No-DNS Testing

```bash
# Find your IP
hostname -I

# Test from same machine
curl http://localhost
curl http://127.0.0.1

# Test from other machine (use your IP)
curl http://192.168.1.100

# Add to /etc/hosts (on each machine)
sudo nano /etc/hosts
# Add: 127.0.0.1    mysite.local

# Check what's listening
sudo lsof -i :80
sudo ss -tlnp

# Check if backend running
sudo lsof -i :3000
curl http://127.0.0.1:3000

# Test Nginx config
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# View logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

---

## Key Takeaways for No-DNS Testing

| Method | Use Case | Command |
|--------|----------|---------|
| **localhost** | Same machine only | `curl http://localhost` |
| **127.0.0.1** | Same machine only | `curl http://127.0.0.1` |
| **Server IP** | Any machine on network | `curl http://192.168.1.100` |
| **/etc/hosts** | Local domain testing | Add to `/etc/hosts`, then `curl http://mysite.local` |
| **Port numbers** | Multiple services | Different ports: 80, 8080, 8081, etc. |
| **URL paths** | Same port, different routes | `/api`, `/admin`, `/static` |
| **Self-signed SSL** | HTTPS testing | Generate cert, use `-k` flag with curl |

---

## Next Steps

1. **Start simple** - Serve a static website on localhost
2. **Test reverse proxy** - Forward to a backend on port 3000
3. **Try load balancing** - Distribute across multiple backends
4. **Add HTTPS** - Use self-signed certificates for testing
5. **Use /etc/hosts** - Create local domain names for testing
6. **When ready** - Set up real DNS and Let's Encrypt certificates

Good luck! 🚀
