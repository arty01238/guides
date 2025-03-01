---
title: Manual installation
summary: A guide to understand how to set up chibisafe manually without Docker
sidebar_position: 2
---

# Running chibisafe manually

> This document will guide you in setting up chibisafe manually in a server. Keep in mind we strongly recommend you use the [Docker guide](/docs/installation/running-with-docker) instead as it's simpler and tested thoroughly.

In order to install chibisafe and run it without containers there are a few things you need to set up on your system beforehand. This guide assumes you are using Ubuntu/Debian so feel free to adjust commands as you see fit based on your distro.

### Pre-requisites
Before running chibisafe make sure you install the following dependencies:

- To install ffmpeg you can run 
```bash
sudo apt install ffmpeg
```
- To get node v20 we recommend you install it via [volta.sh](https://volta.sh/)
- For the reverse proxy we recommend using [Caddy](https://caddyserver.com/)
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```
- But if you rather use [NGINX](https://www.nginx.com/) instead of Caddy then:
```bash
sudo apt update
sudo apt install nginx
```
:::tip
  Choose either Caddy or NGINX, don't install both!
:::

### Installing
The first things you should do is clone the repository to get everything necessary, so run the following commands:
```bash
git clone https://github.com/chibisafe/chibisafe.git
cd chibisafe
yarn install
yarn workspace @chibisafe/backend generate
yarn workspace @chibisafe/backend migrate
yarn build
```

With chibisafe now built you need to run both the backend and the frontend process to get the whole system working. Before starting the chibisafe stack you need to set up 2 environment variables in order for chibisafe to understand where it needs to connect. In order to do so first run the following command:
```bash
echo "BASE_API_URL=http://127.0.0.1:8000" >> ./packages/next/.env
```

You can start both processes by running the following 2 commands in either separate terminals or tmux sessions.
```bash
yarn start:backend
yarn start:frontend
```
Now the backend should be running in port `8000` and the frontend in port `8001`.

#### With PM2

If you rather use PM2 what you can do is create a file called `chibisafe.json` in the root directory of the project with the following content:
```json
{
  "apps": [
    {
      "name": "chibisafe-frontend",
      "cwd": "/path/to/your/chibisafe",
      "script": "yarn",
      "args": ["workspace", "@chibisafe/next", "start"],
      "env": {
            "BASE_API_URL": "http://127.0.0.1:8000",
            "NODE_ENV": "production"
        }
    },
    {
      "name": "chibisafe-backend",
      "cwd": "/path/to/your/chibisafe",
      "script": "yarn",
      "args": ["workspace", "@chibisafe/backend", "start"],
      "env": {
        "NODE_ENV": "production"
      }
    }
  ]
}
```

:::tip
  Make sure to change chibi.domain for your own domain name.
:::

Now you can run the following command and chibisafe will run in the background
```bash
pm2 start chibisafe.json
```


### Reverse proxy
In order to make chibisafe's 2 processes usable and attach a domain name, you need a reverse proxy. This guide assumes you are using Caddy since we like Caddy.
Once you have Caddy installed locally you can add this to your Caddyfile in order reverse proxy the exposed ports from chibisafe:

```caddy title="/etc/caddy/Caddyfile"
chibi.domain {
	route {
		file_server * {
			root /app/uploads
			pass_thru
		}
		@api path /api/*
		reverse_proxy @api http://127.0.0.1:8000 {
			header_up Host {http.reverse_proxy.upstream.hostport}
			header_up X-Real-IP {http.request.header.X-Real-IP}
		}
		@docs path /docs*
		reverse_proxy @docs http://127.0.0.1:8000 {
			header_up Host {http.reverse_proxy.upstream.hostport}
			header_up X-Real-IP {http.request.header.X-Real-IP}
		}
		reverse_proxy http://127.0.0.1:8001 {
			header_up Host {http.reverse_proxy.upstream.hostport}
			header_up X-Real-IP {http.request.header.X-Real-IP}
		}
	}
}
```
:::tip
  Make sure to change chibi.domain for your own domain name, and `root /app/uploads` for the actual path where your upload folder is.
:::

After correctly setting your Caddyfile and restarting the caddy process in your system, you should be able to visit your instance with the domain name and start using it.
