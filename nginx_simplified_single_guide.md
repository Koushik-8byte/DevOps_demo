# Nginx Simplified Single Guide

This file combines the useful parts from:

- `nginx_ubuntu_guide.md`
- `nginx_ubuntu_no_dns_guide.md`
- `Nginx_Guide.txt`

Goal: understand Nginx clearly and use it for common DevOps tasks on Ubuntu.

---

## 1. What Nginx Does

Nginx is a web server and traffic manager.

It can do four common jobs:

1. Serve static websites: HTML, CSS, JS, images.
2. Reverse proxy: receive public traffic and forward it to a backend app.
3. Load balance: distribute traffic across multiple backend apps.
4. Handle HTTPS/SSL: encrypt public traffic.

Simple traffic flow:

```text
Browser
  -> Nginx on port 80 or 443
  -> Backend app on localhost:3000 or static files in /var/www
  -> Response back to browser
```

Important idea:

Your backend apps should usually run on private local ports like `127.0.0.1:3000`.
The public internet talks to Nginx, not directly to your app.

---

## 2. Install and Check Nginx

```bash
sudo apt update
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

Check if Nginx is listening:

```bash
sudo ss -tlnp | grep nginx
nginx -v
curl http://localhost
```

Main Ubuntu paths:

```text
/etc/nginx/nginx.conf              Main config
/etc/nginx/sites-available/        Site configs you create
/etc/nginx/sites-enabled/          Active site configs
/var/www/                          Website files
/var/log/nginx/access.log          Request logs
/var/log/nginx/error.log           Error logs
```

Useful commands:

```bash
sudo nginx -t                 # test config before reload
sudo systemctl reload nginx   # apply config without full restart
sudo systemctl restart nginx  # full restart
sudo systemctl status nginx
```

Always run `sudo nginx -t` before reload or restart.

---

## 3. Testing Without DNS

You do not need a domain name while learning.

Use one of these:

```bash
curl http://localhost
curl http://127.0.0.1
curl http://YOUR_SERVER_IP
```

Find your server IP:

```bash
hostname -I
```

If you want a fake local domain, edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add:

```text
127.0.0.1 mywebsite.local
```

Then test:

```bash
curl http://mywebsite.local
```

This fake domain works only on the machine where you edited `/etc/hosts`.

---

## 4. Static Website Setup

Use this when you want Nginx to directly serve HTML/CSS/JS files.

Create website folder:

```bash
sudo mkdir -p /var/www/mywebsite/html
sudo nano /var/www/mywebsite/html/index.html
```

Example HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Nginx Test</title>
</head>
<body>
  <h1>Nginx is working</h1>
  <p>This page is served from /var/www/mywebsite/html.</p>
</body>
</html>
```

Set permissions:

```bash
sudo chown -R www-data:www-data /var/www/mywebsite
sudo chmod -R 755 /var/www/mywebsite
```

Create config:

```bash
sudo nano /etc/nginx/sites-available/mywebsite
```

Paste:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/mywebsite/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/mywebsite /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Test:

```bash
curl http://localhost
```

---

## 5. Reverse Proxy Setup

Use this when your app runs on a port like `3000`, but users should access it through Nginx on port `80`.

Example backend:

```bash
cd /path/to/your/app
npm start
```

Assume the app listens on:

```text
127.0.0.1:3000
```

Create config:

```bash
sudo nano /etc/nginx/sites-available/reverse-proxy
```

Paste:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Test:

```bash
curl http://localhost
```

Why proxy headers matter:

```text
Host                 Keeps original domain/IP.
X-Real-IP            Sends real visitor IP to backend.
X-Forwarded-For      Keeps full proxy IP chain.
X-Forwarded-Proto    Tells backend if request was HTTP or HTTPS.
```

---

## 6. Load Balancer Setup

Use this when you have multiple backend apps and want Nginx to distribute traffic.

Example backend servers:

```bash
mkdir -p ~/nginx-project/server{1,2,3}

echo "<h1>Node 1 on port 3001</h1>" > ~/nginx-project/server1/index.html
echo "<h1>Node 2 on port 3002</h1>" > ~/nginx-project/server2/index.html
echo "<h1>Node 3 on port 3003</h1>" > ~/nginx-project/server3/index.html

cd ~/nginx-project/server1 && python3 -m http.server 3001 --bind 127.0.0.1 &
cd ~/nginx-project/server2 && python3 -m http.server 3002 --bind 127.0.0.1 &
cd ~/nginx-project/server3 && python3 -m http.server 3003 --bind 127.0.0.1 &
```

Create config:

```bash
sudo nano /etc/nginx/sites-available/loadbalancer
```

Paste:

```nginx
upstream backend_pool {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    listen [::]:80;

    server_name _;

    location / {
        proxy_pass http://backend_pool;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/loadbalancer /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Test round-robin balancing:

```bash
for i in {1..6}; do curl http://localhost; done
```

Expected pattern:

```text
Node 1
Node 2
Node 3
Node 1
Node 2
Node 3
```

Common load balancing methods:

```nginx
upstream backend_pool {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

```nginx
upstream backend_pool {
    ip_hash;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

```nginx
upstream backend_pool {
    server 127.0.0.1:3001 weight=3;
    server 127.0.0.1:3002 weight=1;
}
```

---

## 7. HTTPS Without a Domain

Without a real domain, use a self-signed certificate.

Important:

Browsers will show a warning because the certificate is not trusted by a public certificate authority. This is normal for local testing.

Generate certificate:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/CN=localhost"
```

HTTPS reverse proxy config:

```bash
sudo nano /etc/nginx/sites-available/https-proxy
```

Paste:

```nginx
upstream backend_pool {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    listen [::]:80;

    server_name _;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name _;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    location / {
        proxy_pass http://backend_pool;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/https-proxy /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Test:

```bash
curl -I http://localhost
curl -k https://localhost
```

`-k` tells curl to ignore the self-signed certificate warning.

---

## 8. HTTPS With a Real Domain

Use this after you buy or configure a domain.

DNS setup:

```text
Type: A
Name: @
Value: YOUR_SERVER_PUBLIC_IP
```

Optional www record:

```text
Type: CNAME
Name: www
Value: yourdomain.com
```

Nginx config should use your domain:

```nginx
server_name yourdomain.com www.yourdomain.com;
```

Install Certbot:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
```

Generate trusted SSL certificate:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Test auto-renewal:

```bash
sudo certbot renew --dry-run
```

---

## 9. Example for Server IP 32.192.235.131

This is the focused version of the IP-based load balancer setup from the old `Nginx_Guide.txt`.

Start three backend nodes:

```bash
mkdir -p ~/nginx-project/server{1,2,3}

echo "<h1>Node 1 (Port 3001)</h1>" > ~/nginx-project/server1/index.html
echo "<h1>Node 2 (Port 3002)</h1>" > ~/nginx-project/server2/index.html
echo "<h1>Node 3 (Port 3003)</h1>" > ~/nginx-project/server3/index.html

cd ~/nginx-project/server1 && python3 -m http.server 3001 --bind 127.0.0.1 &
cd ~/nginx-project/server2 && python3 -m http.server 3002 --bind 127.0.0.1 &
cd ~/nginx-project/server3 && python3 -m http.server 3003 --bind 127.0.0.1 &
```

Generate certificate for the IP:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/CN=32.192.235.131"
```

Create config:

```bash
sudo nano /etc/nginx/sites-available/loadbalancer.conf
```

Paste:

```nginx
upstream backend_pool {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    server_name 32.192.235.131;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name 32.192.235.131;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    location / {
        proxy_pass http://backend_pool;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/loadbalancer.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Verify HTTP redirects to HTTPS:

```bash
curl -I http://32.192.235.131/
```

Expected:

```text
HTTP/1.1 301 Moved Permanently
Location: https://32.192.235.131/
```

Verify load balancing:

```bash
for i in {1..6}; do curl -k https://32.192.235.131/; done
```

---

## 10. Troubleshooting

### Config syntax error

Check:

```bash
sudo nginx -t
```

Common causes:

```text
Missing semicolon ;
Missing closing brace }
Wrong file path
Wrong directive name
```

### Port already in use

```bash
sudo ss -tlnp | grep ':80'
sudo ss -tlnp | grep ':443'
```

### 502 Bad Gateway

Nginx is working, but the backend is not reachable.

Check backend:

```bash
sudo ss -tlnp | grep ':3000'
curl http://127.0.0.1:3000
sudo tail -f /var/log/nginx/error.log
```

### 403 Forbidden

Usually file permission problem.

Fix static site permissions:

```bash
sudo chown -R www-data:www-data /var/www/mywebsite
sudo chmod -R 755 /var/www/mywebsite
```

### Cannot access from another machine

Check Nginx is listening on all interfaces:

```bash
sudo ss -tlnp | grep nginx
```

Good:

```text
0.0.0.0:80
*:80
```

Bad for public access:

```text
127.0.0.1:80
```

Also check cloud firewall/security group:

```text
Allow port 80 for HTTP
Allow port 443 for HTTPS
```

### SSL warning in browser

If using self-signed certificate, browser warning is expected.

For no warning, use a real domain plus Certbot/Let's Encrypt.

---

## 11. Quick Cheat Sheet

```bash
# Install
sudo apt update
sudo apt install nginx

# Test and reload
sudo nginx -t
sudo systemctl reload nginx

# Logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Enabled sites
ls -la /etc/nginx/sites-enabled/

# Available sites
ls -la /etc/nginx/sites-available/

# Check ports
sudo ss -tlnp

# Test local site
curl http://localhost

# Test self-signed HTTPS
curl -k https://localhost
```

---

## 12. What to Remember

1. `sites-available` stores configs.
2. `sites-enabled` activates configs through symlinks.
3. Always run `sudo nginx -t` before reload.
4. Static site uses `root`.
5. Reverse proxy uses `proxy_pass`.
6. Load balancer uses `upstream`.
7. Real HTTPS needs a domain and Certbot.
8. Self-signed HTTPS is fine for learning, but browsers will warn.

