# Deploy V2Ray Server

base on CentOS 7

## Step 1.
```bash
# Env
firewall-cmd --zone=public --add-port=443/tcp --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-service=https --permanent
yum install socat nginx -y
```

## Step 2.
```bash
# Get your access key `https://ram.console.aliyun.com/#/user/list?guide` if your domain host on aliyun
# Install cert
curl https://get.acme.sh | sh
acme.sh --upgrade --auto-upgrade
export Ali_Key="yourKey"
export Ali_Secret="yourSecret"
acme.sh --force --issue --dns dns_ali -d *.yourdomain -d yourdomain --dnssleep 0
mkdir -p /etc/nginx/ssl
acme.sh --install-cert -d "*.yourdomain" --key-file "/etc/nginx/ssl/*.yourdomain.key" --fullchain-file "/etc/nginx/ssl/fullchain.cer" --reloadcmd "systemctl force-reload nginx"
```

## Step 3.
```bash
# Install v2ray
bash <(curl -L -s https://install.direct/go.sh)
```
### vi /etc/v2ray/config.json
```bash
{
  "log" : {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },
  "inbound": {
    "port": 9999,
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "c1eees937-d28a-4db7-8a5c-easdf61435a",
          "alterId": 64
        }
      ]
    },
    "streamSettings": {
      "network": "ws",
      "wsSettings": {
        "path": "/ws",
        "headers": {
          "host": "yourdomain"
        }
      }
    }
  },
  "outbound": {
    "protocol": "freedom",
    "settings": {}
  }
}
```

## Step 4.
### vi /etc/nginx/conf.d/yourdomain.conf
```nginx
server {
  server_name *.yourdomain yourdomain;
  listen 80;
  return 301 https://$host$request_uri;
}
server {
  server_name *.yourdomain yourdomain;
  listen 443 ssl http2;

  ssl_certificate /etc/nginx/ssl/fullchain.cer;
  ssl_certificate_key /etc/nginx/ssl/*.yourdomain.key;

  add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
  gzip on;

  location / {
    root /srv/yourdomain/;
    index index.html;
  }

  location /ws {
    proxy_pass http://0.0.0.0:3311;

    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_intercept_errors on;
    error_page 400 /404.html;

    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $http_host;
  }
}
```
### vi /etc/nginx/nginx.conf
```nginx
user nginx;
worker_processes auto;
error_log /dev/null;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
  use epoll;
  worker_connections 2048;
}

http {
    access_log off;
    error_log /dev/null;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
```

## Step 5.
```bash
systemctl start v2ray && systemctl enable v2ray && systemctl start nginx && systemctl enable nginx
```
