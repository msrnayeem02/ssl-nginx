# Cartlar.com SSL & Nginx Setup Guide

This guide explains how to set up Nginx with SSL for `cartlar.com` and `www.cartlar.com` domains, serving the Laravel project located at `/var/www/metastore/public`.

---

## Prerequisites

- Ubuntu/Debian server with root or sudo access
- Domain `cartlar.com` and `www.cartlar.com` pointed to your server IP
- Nginx installed and running
- PHP 8.3 FPM installed and running
- Certbot installed for Let's Encrypt SSL certificates

---

## Nginx Configuration

Create Nginx site config file:

```bash
sudo nano /etc/nginx/sites-available/cartlar.com
````

Add the following configuration:

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name cartlar.com www.cartlar.com;

    return 301 https://$host$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name cartlar.com www.cartlar.com;

    ssl_certificate /etc/letsencrypt/live/cartlar.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cartlar.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    root /var/www/metastore/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_param HTTP_PROXY "";
        fastcgi_param HTTPS on;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
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

Enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/cartlar.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## SSL Certificate Setup

Obtain SSL certificates:

```bash
sudo certbot --nginx -d cartlar.com -d www.cartlar.com
```

Follow prompts for email, terms acceptance, and redirect selection.

---

## Troubleshooting Tips

* Check nginx config syntax:

```bash
sudo nginx -t
```

* Reload nginx if syntax OK:

```bash
sudo systemctl reload nginx
```

---

## File Locations

* Nginx configs: `/etc/nginx/sites-available/cartlar.com`
* SSL certs: `/etc/letsencrypt/live/cartlar.com/`
* Laravel project root: `/var/www/metastore/public`
