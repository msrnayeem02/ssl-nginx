# Nginx & SSL Setup Guide for Cartlar.com

This guide walks you through configuring Nginx and enabling SSL with Let's Encrypt for `cartlar.com` and `www.cartlar.com` on Ubuntu.

---

## Requirements

* Ubuntu/Debian server with root access
* Domains `cartlar.com` and `www.cartlar.com` pointing to your server IP
* Installed: Nginx, Certbot, PHP 8.3 FPM

---

## Step 1: Prepare Secure Nginx Configuration

Create the config file:

```bash
sudo nano /etc/nginx/sites-available/cartlar.com
```

Add:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name cartlar.com www.cartlar.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name cartlar.com www.cartlar.com;

    ssl_certificate /etc/letsencrypt/live/cartlar.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cartlar.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    root /var/www/metastore/public;
    index index.php index.html;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_param HTTPS on;
        include fastcgi_params;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    location ~* \.(env|log|sql|bak|backup|old|tmp)$ {
        deny all;
        access_log off;
        log_not_found off;
    }

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;
}
```

---

## Step 2: Temporary HTTP Config for SSL Challenge

```bash
sudo nano /etc/nginx/sites-available/cartlar-temp.com
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name cartlar.com www.cartlar.com;

    root /var/www/metastore/public;
    index index.php index.html;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
        allow all;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

Enable temporary config:

```bash
sudo ln -s /etc/nginx/sites-available/cartlar-temp.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## Step 3: Issue SSL Certificate

```bash
sudo certbot --nginx -d cartlar.com -d www.cartlar.com
```

Select: **Redirect** when prompted.

---

## Step 4: Switch to Secure Config

```bash
sudo rm /etc/nginx/sites-enabled/cartlar-temp.com
sudo ln -s /etc/nginx/sites-available/cartlar.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## Step 5: Test Everything

```bash
curl -I http://cartlar.com
curl -I https://cartlar.com
curl -I https://www.cartlar.com
curl https://cartlar.com/index.php
```

---

## Optional: Firewall & Security

```bash
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 'Nginx Full'
sudo ufw status
```

Test SSL: [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)

---

## Maintenance & Logs

```bash
sudo certbot renew --dry-run
sudo certbot certificates

sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx
sudo systemctl status php8.3-fpm

sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

---

## File Paths

* Configs: `/etc/nginx/sites-available/`, `/etc/nginx/sites-enabled/`
* SSL: `/etc/letsencrypt/live/cartlar.com/`
* Root: `/var/www/metastore/public`
* Logs: `/var/log/nginx/`
* PHP socket: `/var/run/php/php8.3-fpm.sock`

---

## You’re Done ✅

* HTTPS loads for `cartlar.com` and `www.cartlar.com`
* HTTP redirects to HTTPS
* SSL green lock in browser
* PHP renders correctly
* Static files are cached efficiently
