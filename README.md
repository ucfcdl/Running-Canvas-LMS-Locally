# Local Canvas Dev Server Setup

Instructions for running a local development server for Canvas LMS. This guide assumes you are running Ubuntu 24.04.1 LTS and have a domain under which to host the application and Rich Content Editor (RCE). If you do not have a domain handy, you may use the default http://canvas.docker/

**These steps are not intended for use in a production environment.**

## Table of Contents

- [Setup Docker](#setup-docker)
  - [Install Docker](#install-docker)
  - [Set Docker permissions](#set-docker-permissions)
- [Setup Swapfile](#setup-swapfile)
- [Install Canvas-LMS](#install-canvas-lms)
- [Run initial setup](#run-initial-setup)
- [Configure Canvas](#configure-canvas)
  - [Docker Compose Override](#docker-compose-override)
  - [Domains](#domains)
  - [Session Store](#session-store)
  - [Dynamic Settings](#dynamic-settings)
  - [Vault Contents](#vault-contents)
  - [Environment Variables](#environment-variables)
- [Install and configure Rich Content Editor API](#install-and-configure-rich-content-editor-api)
- [Start Docker/Canvas on instance startup](#start-dockercanvas-on-instance-boot)
- [NGINX](#nginx)

## Setup Docker

### Install Docker

You will want to update the list of avialable packages, then upgrade the installed packages. Install the necessary dependencies. Add the Docker GPG key and add the Docker repository to the list of sources. Install Docker, add the current user to the Docker group, and install an older version of docker-compose to work with Instucture's compose V1 commands.

```sh
sudo apt-get update
sudo apt-get -y upgrade
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker ${USER}

# The canvas startup script currently uses 'docker-compose' V1 commands, requiring the additional installation of an older version of docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

### Set Docker permissions

Set access control lists on the current directory and all its contents, giving user ID and group ID `9999` read/write/execute access. Set default ACL for any new contents in this directory. Create a new group `docker-instructure` with group ID `9999` and add the current user to this group. Set an environment variable to skip the Docker usermod command.

```sh
sudo apt install acl
sudo setfacl -Rm u:9999:rwX,g:9999:rwX .
sudo setfacl -dRm u:9999:rwX,g:9999:rwX .
sudo addgroup --gid 9999 docker-instructure
sudo usermod -a -G docker-instructure $USER

export CANVAS_SKIP_DOCKER_USERMOD=1
```

## Setup Swapfile

Enable swap memory and allocate 4GB of swap space, with the proper permissions to read/write with root. This is especially helpful for running the Canvas LMS on an instnce with low RAM allocation, which can be memory intensive.

```sh
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

## Install Canvas-LMS

```sh
git clone https://github.com/instructure/canvas-lms
cd canvas-lms/
```

Checkout latest the latest stable branch (check dates), e.g.:

```sh
git checkout stable/2025-01-29
```

## Run initial Canvas setup

Skip dory when prompted (y). For first time setup you will want to copy the default yml configs and .env. Upon subsequent runs of this script be sure to say no to these prompts, as they will overwrite your configs. You may be prompted to DROP/MIGRATE the databse. If you are running this for the first time, you will want to DROP the database. If you are running this after initial setup, you will want to MIGRATE the database. The script will also prompt you to name your LMS and to set up a site admin account. Be sure to save these credentials in a secure location.

```sh
./script/docker_dev_setup.sh
```

## Configure Canvas

### Docker Compose Override

Add the domain you want to use to the `VIRTUAL_HOST` env var and set ports to Web Service

`nano docker-compose.override.yml`

```
...
web:
  <<: *BASE
  ports:
    - "9100:80"
  environment:
  <<: *BASE-ENV
  VIRTUAL_HOST: your-canvas-domain-here.com
...
```

### Domains

Setup domains in `domains.yml`. Replace your-canvas-domain-here.com with your domain

`nano config/domain.yml`

```
production:
  domain: "canvas.docker"
test:
  domain: "localhost"
development:
  domain: "your-canvas-domain-here.com"
  ssl: true
```

### Session Store

Setup the session store in `session_store.yml`

`nano config/session_store.yml`

```
development:
  session_store: encrypted_cookie_store
  expire_after: 86400 # 1 day in seconds
  secure: true
production:
  session_store: encrypted_cookie_store
  expire_after: 86400 # 1 day in seconds
  # uncomment this option if your canvas install is over HTTPS, and the cookies
  # will be SSL-only
  #
  secure: true
  #
  # change the time that "stay logged in" tokens are valid for, defaults to 1 month
  #
  expire_remember_me_after: 2592000
test:
  session_store: encrypted_cookie_store
  expire_after: 86400 # 1 day in seconds
```

### Dynamic Settings

Modify the dynamic settings to include the RCE API host. Here we are running under the same domain as the canvas domain, but with the path `/rce`. Replace your-canvas-domain-here.com with your domain.

`nano config/dynamic_settings.yml`

```
development:
  config:
    canvas:
      ...
      rich-content-service:
        app-host: 'your-canvas-domain-here.com/rce'
      ...
```

### Vault Contents

Create a `vault_contents.yml` from the example one

`cp /config/vault_contents.yml.example config/vault_contents.yml`

### Environment Variables

`nano .env`

```
COMPOSE_FILE=docker-compose.yml:docker-compose.override.yml:docker-compose/mailcatcher.override.yml:docker-compose/rce-api.override.yml
```

This is where you map the compose files for any required services. The above config includes the base settings, mailcatcher, and RCE. Add any additional services you may need.

## Install and configure Rich Content Editor API

Modify the `rce-api.override.yml` to include the domain you want to use. Replace your-canvas-domain-here.com with your domain.

`nano docker-compose/rce-api.override.yml`

```
# to use this add docker-compose/rce-api.override.yml
# to your COMPOSE_FILE var in .env

services:
  web:
    links:
      - canvasrceapi

  canvasrceapi:
    image: instructure/canvas-rce-api
    ports:
      - "3000:80"
    environment:
      VIRTUAL_HOST: your-canvas-domain-here.com
      VIRTUAL_PORT: 80
      HTTP_PROTOCOL_OVERRIDE: "https"
      PASSENGER_MIN_INSTANCES: 2
      PASSENGER_MAX_POOL_SIZE: 6
      NGINX_WORKER_CONNECTIONS: 2048
      STATSD_HOST: 127.0.0.1
      STATSD_PORT: 8125
      ECOSYSTEM_SECRET: "astringthatisactually32byteslong"
      ECOSYSTEM_KEY: "astringthatisactually32byteslong"
      CIPHER_PASSWORD: TEMP_PASSWORD
    init: true

```

## Start Docker/Canvas on instance boot

When the server restarts, you will need to start the docker containers. You can do this automatically on boot by creating a systemd service that starts the compose app. This will start both the Canvas LMS and the RCE API containers.

```sh
sudo tee /etc/systemd/system/docker-compose-app.service << 'EOF'
[Unit]
Description=Docker Compose Application Service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ubuntu/canvas-lms
ExecStartPre=/bin/sleep 5
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
User=ubuntu
Group=docker
TimeoutStartSec=180

[Install]
WantedBy=multi-user.target
EOF
```

Enable the service and start it

```sh
sudo systemctl enable docker-compose-app.service
sudo systemctl daemon-reload
sudo systemctl restart docker-compose-app
```

## NGINX

```sh
sudo apt-get install nginx
sudo mkdir -p /etc/nginx/sites-available/
sudo mkdir -p /etc/nginx/sites-enabled
sudo nano /etc/nginx/sites-available/canvas
```

Paste the following to create the canvas site config. Be sure to replace your-canvas-domain-here.com with your domain. Again, note that this runs the RCE API under the same domain as the canvas app, but with the path `/rce`.

```
# /etc/nginx/sites-available/canvas
server {
    listen 80;
    server_name your-canvas-domain-here.com;

    proxy_set_header X-Forwarded-Proto "https";
    # Main Canvas app
    location / {
        proxy_pass http://127.0.0.1:9100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
    }

    # RCE API service
    location /rce/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
    }
}
```

Enable the site and restart nginx

```sh
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/canvas /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```
