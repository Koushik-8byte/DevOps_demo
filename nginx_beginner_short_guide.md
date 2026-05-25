# Nginx Beginner Quick Guide
Use this first. It keeps only the core Nginx ideas and commands.

## 1. Main Idea
Nginx sits in front of your website or app.
```text
User -> Nginx -> Website files or backend app
```
Common jobs:
```text
Static web server = serves HTML/CSS/JS
Reverse proxy     = sends traffic to one app
Load balancer     = sends traffic to many apps
HTTPS             = secure traffic
```
Most common DevOps setup:
```text
User -> Nginx on 80/443 -> App on 127.0.0.1:3000
```

## 2. Install Nginx
```bash
sudo apt update
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
curl http://localhost
```
Important paths:
```text
/etc/nginx/sites-available/   configs you create
/etc/nginx/sites-enabled/     active configs
/var/www/                     website files
/var/log/nginx/error.log      error log
/var/log/nginx/access.log     access log
```

## 3. Daily Commands
```bash
sudo nginx -t                         # test config
sudo systemctl reload nginx           # apply changes
sudo systemctl restart nginx          # full restart
sudo tail -f /var/log/nginx/error.log # live errors
```
After editing config:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 4. Static Website
Use this when Nginx directly serves HTML files.
```bash
sudo mkdir -p /var/www/mywebsite/html
echo "<h1>Nginx is working</h1>" | sudo tee /var/www/mywebsite/html/index.html
sudo chown -R www-data:www-data /var/www/mywebsite
sudo chmod -R 755 /var/www/mywebsite
sudo nano /etc/nginx/sites-available/mywebsite
```
Config:
```nginx
server {
    listen 80;
    server_name _;
    root /var/www/mywebsite/html;
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
```
Enable:
```bash
sudo ln -s /etc/nginx/sites-available/mywebsite /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
curl http://localhost
```
Remember: `root` means the folder where static website files live.

## 5. Reverse Proxy
Use this when your backend app runs on port `3000`.
```bash
sudo nano /etc/nginx/sites-available/reverse-proxy
```
Config:
```nginx
server {
    listen 80;
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
curl http://localhost
```
Remember: `proxy_pass` sends traffic to your backend app.

## 6. Load Balancer
Use this when you have multiple backend apps.
```nginx
upstream backend_pool {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
server {
    listen 80;
    server_name _;
    location / { proxy_pass http://backend_pool; }
}
```
Remember: `upstream` means a group of backend servers.

## 7. HTTPS
```text
HTTP  = port 80
HTTPS = port 443
```
For real HTTPS, use a domain and Certbot:
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```
Without a domain, self-signed SSL works for testing, but browser shows a warning.

## 8. Testing Without Domain
```bash
curl http://localhost
curl http://127.0.0.1
hostname -I
curl http://YOUR_SERVER_IP
```
Fake local domain:
```bash
sudo nano /etc/hosts
```
Add:
```text
127.0.0.1 mywebsite.local
```
Test:
```bash
curl http://mywebsite.local
```

## 9. Common Errors
```text
502 Bad Gateway = backend app is down or wrong port
403 Forbidden   = permission problem
Config error    = missing ; or missing }
Port conflict   = port 80/443 already used
```
Debug:
```bash
curl http://127.0.0.1:3000
sudo tail -f /var/log/nginx/error.log
sudo nginx -t
sudo ss -tlnp | grep :80
```

## 10. Remember These Words
```text
root        serves static files
proxy_pass  sends traffic to backend
upstream    groups backend servers
listen 80   HTTP
listen 443  HTTPS
nginx -t    test config
reload      apply config
```
