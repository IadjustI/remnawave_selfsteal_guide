# remnawave_selfsteal_guide
A comprehensive step-by-step guide on how to deploy a SelfSteal node on Remnawave, including the complete setup process for the Remnawave platform itself. This guide covers prerequisites, installation, configuration, and troubleshooting tips to ensure a smooth and secure deployment.

This guide provides a step-by-step walkthrough for configuring the control panel and the node, each deployed on at least two separate VPS instances that meet the system requirements outlined here: https://remna.st/docs/install/requirements.

We will begin with the basic setup of the Remnawave control panel.

Step 0 — Updating and installing requirement tools

# Updating entire system and installing curl
`apt update && apt upgrade && apt install curl`

Step 1 — Install Docker

# Install Docker using the official convenience script
`curl -fsSL https://get.docker.com | sh`

Step 2 — Download required files

# Create the project directory and enter it
`mkdir -p /opt/remnawave && cd /opt/remnawave`

# Fetch the Docker Compose file
`curl -o docker-compose.yml https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/docker-compose-prod.yml`

# Fetch the sample environment file
`curl -o .env https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/.env.sample`

Step 3 — Configure .env

# Generate JWT secrets used for auth and API token signing
`sed -i "s/^JWT_AUTH_SECRET=.*/JWT_AUTH_SECRET=$(openssl rand -hex 64)/" .env && sed -i "s/^JWT_API_TOKENS_SECRET=.*/JWT_API_TOKENS_SECRET=$(openssl rand -hex 64)/" .env`

# Generate additional secrets for metrics and webhooks
`sed -i "s/^METRICS_PASS=.*/METRICS_PASS=$(openssl rand -hex 64)/" .env && sed -i "s/^WEBHOOK_SECRET_HEADER=.*/WEBHOOK_SECRET_HEADER=$(openssl rand -hex 64)/" .env`

# Rotate the Postgres password and keep DATABASE_URL in sync
`pw=$(openssl rand -hex 24) && sed -i "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$pw/" .env && sed -i "s|^\(DATABASE_URL=\"postgresql://postgres:\)[^\@]*\(@.*\)|\1$pw\2|" .env`

Set domains

`nano .env`
# Search in .env this values
FRONT_END_DOMAIN="panel.yourdomain.com"
SUB_PUBLIC_DOMAIN="sub.yourdomain.com"
# Change "panel.yourdomain.com" & "sub.yourdomain.com" on your domains

Step 4 — Start the stack

# Start containers and tail logs with timestamps
`docker compose up -d && docker compose logs -f -t`

Step 5 — Point domain names to your server via DNS provider or registar

Step 6 — Caddy configuration

# Create a file called Caddyfile in the /opt/remnawave/caddy directory.
`mkdir -p /opt/remnawave/caddy && cd /opt/remnawave/caddy && nano Caddyfile`

# Paste the following configuration

```
https://REPLACE_WITH_YOUR_DOMAIN {
        reverse_proxy * http://remnawave:3000
}
:443 {
    tls internal
    respond 204
}

https://SUBSCRIPTION_PAGE_DOMAIN {
        reverse_proxy * http://remnawave-subscription-page:3010
}
```

# Create docker-compose.yml

`cd /opt/remnawave/caddy && nano docker-compose.yml`

# Paste the following configuration.

```
services:
    caddy:
        image: caddy:2.9
        container_name: 'caddy'
        hostname: caddy
        restart: always
        ports:
            - '0.0.0.0:443:443'
            - '0.0.0.0:80:80'
        networks:
            - remnawave-network
        volumes:
            - ./Caddyfile:/etc/caddy/Caddyfile
            - caddy-ssl-data:/data

networks:
    remnawave-network:
        name: remnawave-network
        driver: bridge
        external: true

volumes:
    caddy-ssl-data:
        driver: local
        external: false
        name: caddy-ssl-data
```

# Start the container

`docker compose up -d && docker compose logs -f -t`

# Remnawave Subscription Page
Creating docker-compose.yml file
`mkdir -p /opt/remnawave/subscription && cd /opt/remnawave/subscription && nano docker-compose.yml`

# Paste docker-compose.yml file content
```
services:
    remnawave-subscription-page:
        image: remnawave/subscription-page:latest
        container_name: remnawave-subscription-page
        hostname: remnawave-subscription-page
        restart: always
        environment:
            - REMNAWAVE_PANEL_URL=https://panel.com
            - APP_PORT=3010
            - META_TITLE="Subscription Page Title"
            - META_DESCRIPTION="Subscription Page Description"
        ports:
            - '127.0.0.1:3010:3010'
        networks:
            - remnawave-network

networks:
    remnawave-network:
        driver: bridge
        external: true
```
# Start the container

`docker compose up -d && docker compose logs -f`

Step 7 — Remnawave Node configuration

# Open your second vps server, for node you created before
# Updating entire system and installing curl
`apt update && apt upgrade && apt install curl`

# Install Docker using the official convenience script
`curl -fsSL https://get.docker.com | sh`

# Create project directory

`mkdir -p /opt/remnanode && cd /opt/remnanode`

# Configure the .env file

`nano .env`

# .env file content

```
APP_PORT=2222
SSL_CERT=CERT_FROM_MAIN_PANEL
```

Create docker-compose.yml file

`nano docker-compose.yml`
# docker-compose.yml file content

```
services:
    remnanode:
        container_name: remnanode
        hostname: remnanode
        image: remnawave/node:latest
        restart: always
        network_mode: host
        env_file:
            - .env
```
# Pulling container
`docker compose up -d && docker compose logs -f`

Step 8 — Selfsteal (SNI) Setup
# Create the working directory and open Caddyfile for editing
`mkdir -p /opt/selfsteel && cd /opt/selfsteel && nano Caddyfile`

# Paste the following
```
{
    https_port {$SELF_STEAL_PORT}
    default_bind 127.0.0.1
    servers {
        listener_wrappers {
            proxy_protocol {
                allow 127.0.0.1/32
            }
            tls
        }
    }
    auto_https disable_redirects
}

http://{$SELF_STEAL_DOMAIN} {
    bind 0.0.0.0
    redir https://{$SELF_STEAL_DOMAIN}{uri} permanent
}

https://{$SELF_STEAL_DOMAIN} {
    root * /var/www/html
    try_files {path} /index.html
    file_server

}


:{$SELF_STEAL_PORT} {
    tls internal
    respond 204
}

:80 {
    bind 0.0.0.0
    respond 204
}
```
# Configure environment variables
`nano .env`
# Paste (replace steel.domain.com with your placeholder domain):
<table>
  <tbody>
    <tr>
      <td><code>SELF_STEAL_DOMAIN=steel.domain.com</code></td>
      <td>MUST match XRAY <code>realitySettings.serverNames</code></td>
    </tr>
    <tr>
      <td><code>SELF_STEAL_PORT=9443</code></td>
      <td>MUST match XRAY <code>realitySettings.dest</code></td>
    </tr>
  </tbody>
</table>

# Create docker-compose.yml
`nano docker-compose.yml`

# Paste:
```
services:
  caddy:
    image: caddy:2.9.1
    container_name: caddy-remnawave
    restart: unless-stopped
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ../html:/var/www/html
      - ./logs:/var/log/caddy
      - caddy_data_selfsteal:/data
      - caddy_config_selfsteal:/config
    env_file:
      - .env
    network_mode: "host"

volumes:
  caddy_data_selfsteal:
  caddy_config_selfsteal:
```

# Launch and verify
`docker compose up -d && docker compose logs -f -t`

# Create the placeholder site
```
mkdir -p /opt/html
printf '%s\n' '<!doctype html><meta charset="utf-8"><title>Selfsteal</title><h1>It works.</h1>' \
  > /opt/html/index.html
```

Step 9
# Xray config (Reality) — update via Panel 
Note: The Shadowsocks inbound is optional and may be safely removed in the future if not required.

```
{
  "log": {
    "loglevel": "info"
  },
  "inbounds": [
    {
      "tag": "Shadowsocks",
      "port": 1234,
      "protocol": "shadowsocks",
      "settings": {
        "clients": [],
        "network": "tcp,udp"
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      }
    },
    {
      "tag": "VLESS",
      "port": 443,
      "listen": "0.0.0.0",
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "sniffing": {
        "enabled": true,
        "routeOnly": false,
        "destOverride": [
          "http",
          "tls",
          "quic",
          "fakedns"
        ],
        "metadataOnly": false
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "tcpSettings": {
          "header": {
            "type": "none"
          },
          "acceptProxyProtocol": false
        },
        "realitySettings": {
          "dest": "9443",
          "show": false,
          "xver": 0,
          "spiderX": "/",
          "shortIds": [
            "CHANGE_ME_SHORTID"
          ],
          "publicKey": "CHANGE_ME_PUBLIC_KEY",
          "privateKey": "CHANGE_ME_PRIVATE_KEY",
          "fingerprint": "chrome",
          "serverNames": [
            "steel.domen.com"
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "DIRECT",
      "protocol": "freedom"
    },
    {
      "tag": "BLOCK",
      "protocol": "blackhole"
    }
  ],
  "routing": {
    "rules": [
      {
        "ip": [
          "geoip:private"
        ],
        "type": "field",
        "outboundTag": "BLOCK"
      },
      {
        "type": "field",
        "domain": [
          "geosite:private"
        ],
        "outboundTag": "BLOCK"
      },
      {
        "type": "field",
        "protocol": [
          "bittorrent"
        ],
        "outboundTag": "BLOCK"
      }
    ]
  }
}
```
