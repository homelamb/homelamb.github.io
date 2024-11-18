This tutorial will walk you through how to use Cloudflare Tunnel with Traefik and Google OAuth. You may find individual tutorials elsehwere for how to use each of these applications for your Docker home lab set up. But here we will focus specifically on how to have these services play together nicely with each other. I used the [Traefik](https://www.smarthomebeginner.com/traefik-v3-docker-compose-guide-2024/) and [Google OAuth](https://www.smarthomebeginner.com/google-oauth-traefik-forward-auth-2024/) tutorials at smarthomebeginner.com to set up Traefik and Google OAuth. I recommend that you follow their super helpful Traefik tutorial and Google OAuth tutorial for the full setup. I will focus here on the changes that you would need for using Cloudflare Tunnel.

To keep this guide self contained, let's start with a simple example of a Traefik setup from the office [guide](https://doc.traefik.io/traefik/getting-started/quick-start/). The steps here can then be adapted to your actual setup.

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
      - "traefik.http.routers.whoami.rule=Host(`whoami.$TODO_YOUR_DOMAIN`)"
```

I have used the `TODO` prefix for environment variables which you would need to populate. You can create a `.env` file in the same directory as your docker compose file to specify the environment variables.

You can test the set up by running

```shell
curl -H Host:whoami.$TODO_YOUR_DOMAIN http://127.0.0.1
```

The output should show your IP address and other metadata.

### Setting up cloudflare tunnel

Follow Cloudflare's guide to set up a cloudflare tunnel using the dashboard.

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

Insert directions for how to set it up for a single subdomain using cloudflare UI.

You can test the setup using

```shell
curl -L whoami.$TODO_DOMAIN_NAME
```

The `-L` flag allows curl to follow the redirects used by Cloudflare tunnels. You should also be able to use a device outside your local network to visit your website.

## Part 2: Coming soon
