---
title: Reduce Treafik and CrowdSec logging
date: 2025-01-05 19:00:00 -0800
categories: [Docker]
tags: [crowdsec, traefik, docker]
---

I set up Traefik and the [CrowdSec plugin](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin) for Traefik on my Docker server. If your set up is similar, and you would like to reduce the amount of logging to `STDOUT`, `STDERR` and the log file for Traefik, this guide may be helfpul.

## Reducing Traefik logs

My Docker Compose file had the log level set to `DEBUG`. This results in a lot of logs being written to your configured `log.filePath` when the Traefik hosts are being accessed.

```terminal
2025-01-02T17:26:57-08:00 DBG github.com/traefik/traefik/v3/pkg/server/service/loadbalancer/wrr/wrr.go:196 > Service selected by WRR: cd65da17a5fcd4a4
2025-01-02T17:26:58-08:00 DBG github.com/traefik/traefik/v3/pkg/server/service/loadbalancer/wrr/wrr.go:196 > Service selected by WRR: cd65da17a5fcd4a4
2025-01-02T17:27:28-08:00 DBG github.com/traefik/traefik/v3/pkg/server/service/loadbalancer/wrr/wrr.go:196 > Service selected by WRR: cd65da17a5fcd4a4
```

In your Docker Compose file, reduce the log level to `INFO` to get rid of these logs. You can also remove the entire line if you would like to fallback to the default level of `ERROR`.

```yaml
      - --log.level=INFO # (Default: error)
```

## Reducing CrowdSec logs

For CrowdSec, logs were being written to `STDERR` every minute with a heartbeat request.

```terminal
crowdsec       | time="2025-01-06T19:45:23-08:00" level=info msg="127.0.0.1 - [Mon, 06 Jan 2025 19:45:23 PST] \"GET /v1/heartbeat HTTP/1.1 200 21.432928ms \"crowdsec/v1.6.3-4851945a-docker\" \""
crowdsec       | time="2025-01-06T19:46:23-08:00" level=info msg="127.0.0.1 - [Mon, 06 Jan 2025 19:46:23 PST] \"GET /v1/heartbeat HTTP/1.1 200 24.638082ms \"crowdsec/v1.6.3-4851945a-docker\" \""
crowdsec       | time="2025-01-06T19:47:23-08:00" level=info msg="127.0.0.1 - [Mon, 06 Jan 2025 19:47:23 PST] \"GET /v1/heartbeat HTTP/1.1 200 22.23079ms \"crowdsec/v1.6.3-4851945a-docker\" \""
```

The log level configuration can be found inside the `/etc/crowdsec/config.yaml` file within the container. The location on your system will depend on the directory mapping you configured in the Compose file.

```yaml
      - $DOCKERDIR/crowdsec/etc:/etc/crowdsec
```

Edit the `config.yaml` file to change the `log_level` for the `server`. The default level is `info`.

```yaml
api:
...
  server:
    log_level: error
```






