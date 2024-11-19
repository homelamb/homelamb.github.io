---
title: Using Cloudflare Tunnel with Traefik
date: 2024-11-17 18:00:00 -0800
categories: [Docker]
tags: [docker, traefik, cloudflare tunnel]
---
This tutorial will walk you through how to use Cloudflare Tunnel with Traefik and Google OAuth. You may find individual tutorials elsehwere for how to use each of these applications for your Docker home lab set up. But here we will focus specifically on how to have these services play together nicely with each other. I used the [Traefik](https://www.smarthomebeginner.com/traefik-v3-docker-compose-guide-2024/) and [Google OAuth](https://www.smarthomebeginner.com/google-oauth-traefik-forward-auth-2024/) tutorials at smarthomebeginner.com to set up Traefik and Google OAuth. I recommend that you follow their super helpful Traefik tutorial and Google OAuth tutorial for the full setup. I will focus here on the changes that you would need for using Cloudflare Tunnel.

To keep this guide self contained, let's start with a simple example of a Traefik setup from the official [guide](https://doc.traefik.io/traefik/getting-started/quick-start/). The steps here can then be adapted to your actual setup.

## Part 1: Basic Traefik setup with Cloudflare Tunnel

```yaml
services:
  reverse-proxy:
    # The official v3 Traefik docker image
    image: traefik:v3.2
    # Enables the web UI and tells Traefik to listen to docker
    command: 
      - --api.insecure=true
      - --providers.docker
      - --providers.docker.exposedbydefault=false
    ports:
      # The HTTP port
      - "80:80"
      # The Web UI (enabled by --api.insecure=true)
      - "8080:8080"
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock
      
  whoami:
    # A container that exposes an API to show its IP address
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.$TODO_YOUR_DOMAIN`)"
```

I have used the `TODO` prefix for environment variables which you would need to populate. You can create a `.env` file in the same directory as your docker compose file to specify the environment variables.

You can test the set up by running

```shell
curl -H Host:whoami.$TODO_YOUR_DOMAIN http://127.0.0.1
```

The output should show your IP address and other metadata.

### Setting up the Cloudflare tunnel

Follow Cloudflare's [guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/) to set up a Cloudflare tunnel using the dashboard.

![Cloudflare Tunnel configuration page showing mapping from the public domain name to the whoami container present behind the Traefik proxy](/assets/img/posts/cloudflare-tunnel-http.png)
_Adding a public hostname that maps to the new Cloudflare tunnel. Notice that in the URL field we put `reverse-proxy`, which is the container name of the Traefik container we defined. This works because docker containers within the same network can reach each other using the container names._

```yaml
  cloudflared:
    image: cloudflare/cloudflared
    container_name: cloudflared
    security_opt:
      - no-new-privileges:true
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=$TODO_CLOUDFLARE_TOKEN
    restart: unless-stopped
```

You can test the setup using

```shell
curl -L whoami.$TODO_DOMAIN_NAME
```

The `-L` flag allows curl to follow the redirects used by Cloudflare tunnels. You should also be able to use a device outside your local network to visit your website.

## Part 2: Using wildcard for subdomains

To minimize configuration overhead, you may wish to avoid having to update the Cloudflare Tunnel configuration every time you want to expose a new Docker container via the tunnel. We can achieve this by using a wildcard DNS entry. 

Start by adding another public hostname for your tunnel.

![Cloudflare Tunnel configuration page showing adding a new hostname using wildcard* for the subdomain](/assets/img/posts/cloudflare-tunnel-http-wildcard.png)
_Adding a wildcard public hostname for the Cloudflare tunnel_

Notice the warning about DNS records. We need to manually create a wildcard DNS record. This is where our existing CNAME DNS record for `whoami` will come in handy. We can copy the `Target` field from the existing record and use it for the new DNS record. This works because we would like the new record to point to the same tunnel. The `Target` should be of the form `TODO_YOUR_TUNNEL_ID.cfargotunnel.com`.

![Cloudflare DNS configuration page showing creating a new CNAME DNS record](/assets/img/posts/cloudflare-dns-wildcard.png)
_Adding a wildcard DNS entry with the same `Target` as the existing `whoami` entry._

You can now go ahead and remove the `whoami` hostname entry from the tunnel configuration page. This will also remove the DNS entry automatically.

You can confirm that the tunnel still works with the new wildcard entry by running

```shell
curl -L whoami.$TODO_DOMAIN_NAME
```

Adding a new container behind the proxy will now only require modifying the Treafik configuration.

## Part 3: Coming soon
