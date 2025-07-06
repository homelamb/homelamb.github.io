---
title: Using Crowdsec with Traefik and Cloudflare Tunnels
date: 2025-07-05 20:00:00 -0800
categories: [Docker]
tags: [crowdsec, docker, traefik, cloudflare tunnel]
---

In this tutorial we will cover adding Crowdsec to a home lab that uses Traefik and Cloudflare Tunnels. This is a follow-up of my [previous post]({% post_url 2024-11-17-using-cloudflare-tunnel-with-traefik %}) where we covered setting up Traefik with Cloudflare tunnels. If you are getting started with your set up, I would recommend starting there as we will build upon it here. If you already have an existing setup, it might be helpful to read through the previous post to see how it maps to your current setup.

We will be using the Crowdsec Traefik [plugin](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin) for our setup. I used the sample [compose](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin/blob/abae855d9e82e248806f110beb956499e55f9394/docker-compose.yml) file from the plugin's repo and [this](https://www.reddit.com/r/selfhosted/comments/1dcn19v/standing_up_the_crowdsec_bouncer_plugin_in_traefik/) Reddit post as reference for my own setup.

## Part 1: Adding Crowdsec container

To your docker compose file add an entry for `crowdsec`.

```yaml
  crowdsec:
    image: crowdsecurity/crowdsec:latest
    container_name: crowdsec
    restart: unless-stopped
    environment:
      - BOUNCER_KEY_TRAEFIK=$TODO_CROWDSEC_BOUNCER_API_KEY
      - COLLECTIONS=crowdsecurity/traefik crowdsecurity/appsec-virtual-patching crowdsecurity/appsec-generic-rules
    volumes:
      - $TODO_DOCKER_DIR/crowdsec/data:/var/lib/crowdsec/data
      - $TODO_DOCKER_DIR/crowdsec/etc:/etc/crowdsec
      - $TODO_DOCKER_DIR/logs/traefik:/var/log/traefik:ro
      - $TODO_DOCKER_DIR/crowdsec/acquis.yaml:/etc/crowdsec/acquis.yaml:ro
```
{: file="docker-compose.yml"}

### Generate Crowdsec API key

With the Crowdsec container added to your config, spin up your docker containers(`docker compose up --detach`). Now execute the following command using your terminal.

```shell
docker exec crowdsec cscli bouncers add crowdsecBouncer
```

This should provide you with the Crowdsec API key. As mentioned in the previous post, you can specify the `TODO_CROWDSEC_BOUNCER_API_KEY` environment variable in a `.env` file in the same directory as your docker compose file.

## Part 2: Setting up Crowdsec Traefik plugin

We will start by updating the Traefik container in our docker compose file.

```yaml
  reverse-proxy:
    image: traefik:v3.2
    command: 
      ...
      - --log.level=DEBUG
      - --log.filepath=/logs/traefik.log
      - "--experimental.plugins.crowdsec-bouncer.modulename=github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin"
      - "--experimental.plugins.crowdsec-bouncer.version=v1.4.4"
      - "--accesslog"
      - "--accesslog.filepath=/logs/access.log"
      - --entrypoints.websecure.forwardedHeaders.trustedIPs=$LOCAL_IPS
    ...
    volumes:
      ...
      - $TODO_DOCKER_DIR/logs/traefik:/logs # Traefik logs
    environment:
      ...
      - TODO_CROWDSEC_BOUNCER_API_KEY=$TODO_CROWDSEC_BOUNCER_API_KEY
    ...
    depends_on:
      - 'crowdsec'
```
{: file="docker-compose.yml" }

Adding `log.level` and `log.filepath` is not strictly needed, but I found it helpful to check that things are working correctly.

The `$LOCAL_IPS` environment variable defines all the standard local IPs. Here we are instructing Traefik to trust the `X-Forwarded-For` header for requests coming from local IPs. This is important because the requests coming from the cloudflare tunnel will have the local IP of the tunnel container and the IP of the actual visitor will be present in the `X-forwarded-for` header. Without this line, Traefik will drop the forwarded headers and Crowdsec won't be able to differentiate requests coming from different IPs.

```shell
LOCAL_IPS='127.0.0.1/32,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12'
```
{: file=".env" }

### Adding the Crowdsec middleware

Create a new file for the Crowdsec middleware: `middlewares-crowdsec-root.yml` in the same directory as your other middlewares.

```yaml
http:
  middlewares:
    middlewares-crowdsec-root:
      plugin:
        crowdsec-bouncer:
          enabled: true
          crowdseclapikey: {% raw %}{{ env "TODO_CROWDSEC_BOUNCER_API_KEY" }}{% endraw %}
          crowdsecappsecenabled: true
```
{: file="traefik/rules/middlewares-crowdsec-root.yml"}

Now let's add this middleware to our middleware chain.
```
http:
  middlewares:
    chain-oauth:
      chain:
        middlewares:
          - middlewares-rate-limit
          - middlewares-secure-headers
          - middlewares-crowdsec-root
          - middlewares-oauth
```
{: file="traefik/rules/chain-oauth.yml"}

### Create acquis.yaml file

Create a new folder called `crowdsec` in your docker root directory and create a new file named `acquis.yaml` within it. I used the same [config](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin/blob/84a5674b14aa982b8e60c21d1706c8e81ec95d40/acquis.yaml) as the plugin's repo.

```yaml
---
filenames:
 - /var/log/traefik/access.log
labels:
  type: traefik

---
listen_addr: 0.0.0.0:7422
appsec_config: crowdsecurity/virtual-patching
name: myAppSecComponent
source: appsec
labels:
  type: appsec
```
{: file="crowdsec/acquis.yaml"}

## Testing the setup

With all the configuration now in place, start your docker containers again using `docker compose up`.

You can use a device on another network to test whether Crowdsec is working. You should be able to access your containers using your public domain name. If you set up a `whoami` container as mentioned in the [previous post]({% post_url 2024-11-17-using-cloudflare-tunnel-with-traefik %}), you can use it for this test. On the other device, like your mobile phone, visit `https://whoami.TODO_YOUR_DOMAIN`. You should be able to load the page successfully.

Now let's test that blocking is working. First, get the IP address of your other device. You can find it in Traefik's `access.log` file. This will be the same as the `X-Forwarded-Host` entry on the whoami web page. To block the device execute the following command in your terminal

```shell
docker exec crowdsec cscli decisions add --ip TODO_OTHER_DEVICE_IP -d 10m
```

This will add your device's IP to the blocklist for 10 minutes. Trying to access your containers' web pages should now result in a `403` HTTP error response.

You can remove your IP from the blocklist by running

```shell
docker exec crowdsec cscli decisions remove --ip TODO_OTHER_DEVICE_IP
```

This should be it. Let me know if you have any suggestions for improving this setup or if you run into any issues.
