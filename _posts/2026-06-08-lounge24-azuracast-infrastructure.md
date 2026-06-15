---
layout: post
toc: true
title: "Lounge24 Radio - AzuraCast Infrastructure Deep Dive"
promote_kleindrops: true
date: 2026-06-08
author: Anthony Klein
description: >
  Deep operational reference for the Lounge24 Radio broadcast stack. AzuraCast
  on Docker, pfSense NAT for broadcast ingest, Cloudflare DNS split between proxied
  listener traffic and grey-cloud DJ ingest, troubleshooting 522s, and MariaDB
  recovery after broken updates.
tags:
  - azuracast
  - lounge24
  - internet-radio
  - web-radio
  - docker
  - homelab
  - pfsense
  - cloudflare
  - cloudflare-tunnel
  - self-hosted
  - networking
  - icecast
  - nat
  - dns
  - reverse-proxy
  - dockhand
  - mariadb
  - troubleshooting
  - broadcast
  - broadcast-infrastructure
  - radio-server
  - internet-radio-station
---

The [previous post on Lounge24 Radio](/2026/03/31/lounge24-azuracast) covered the move from vanilla Icecast2 and MPD to AzuraCast. This is the deeper operational reference covering architecture, port routing, DNS, broadcaster connection settings, and the recovery procedures I've had to run.

## Architecture

```
[Listener / Browser]
        │
        ▼
  Cloudflare (proxied)
        │  HTTPS 443
        ▼
  NPM reverse proxy or Cloudflare Tunnel → 192.168.70.4:8090
        │
        ▼
  AzuraCast container on dockerlab

[Live DJ Broadcaster]
        │
        ▼
  dj.aklein.studio (grey-cloud A record → WAN IP)
        │  TCP 8005 Icecast
        ▼
  pfSense NAT → 192.168.70.4:8005
        │
        ▼
  AzuraCast container on dockerlab
```

Web UI and listener traffic flows through Cloudflare. Broadcast ingest bypasses Cloudflare entirely because Cloudflare cannot proxy arbitrary TCP on port 8005. The broadcaster connects direct via a grey-cloud DNS record and a pfSense NAT forward.

## Key Config Decisions

**Use `stable` not `latest` for the AzuraCast image.** The rolling `latest` tag can push breaking changes multiple times per day with no testing gate. The `stable` tag is a tested release that only updates when AzuraCast cuts a new version. The `updater` container has no `stable` tag so leave that one on `latest`.

**`dj.aklein.studio` must stay grey-cloud.** If it were orange-cloud proxied, Cloudflare would intercept the TCP connection on port 8005 and the broadcaster cannot connect. Grey-cloud resolves directly to WAN IP and pfSense forwards to the container.

## DNS Records

| Hostname | Type | Proxied |
|---|---|---|
| radio.aklein.studio | A → WAN IP | ✅ Orange cloud |
| dj.aklein.studio | A → WAN IP | ❌ Grey cloud |

pfSense Unbound has a host override mapping both hostnames to 192.168.70.4 internally so LAN clients never hairpin through WAN.

## pfSense NAT Rule

Forwards TCP 8000 to 8005 from WAN to 192.168.70.4. Covers both the Icecast listener port and the DJ broadcast ingest port. This rule must stay in place regardless of whether Cloudflare Tunnel is handling the web UI since the tunnel only carries HTTP and HTTPS.

## Port Reference

| Port | Protocol | Purpose |
|---|---|---|
| 8090 | HTTP | AzuraCast Web UI |
| 8005 | TCP | Icecast broadcast ingest (DJ input) |
| 8006 | TCP | SHOUTcast broadcast ingest |
| 8000 to 8004 | TCP | Icecast listener streams |
| 2222 | TCP | SFTP media upload |

## Broadcaster Connection Settings

| Field | Value |
|---|---|
| Type | Icecast |
| Address | dj.aklein.studio |
| Port | 8005 |
| Mountpoint | / |
| User | dj |
| SSL/TLS | Disabled |

SSL is disabled on port 8005. The broadcast connection is authenticated via password and the data is audio rather than anything sensitive, so plain TCP on 8005 is acceptable. TLS is supported on port 8006 if needed later.

## Cloudflare Tunnel

A Cloudflare Tunnel via `cloudflared` can replace the NPM reverse proxy for serving `radio.aklein.studio` publicly. The main benefit is eliminating the inbound 80/443 port forward dependency and removing reliance on a stable public IP for web traffic.

The tunnel does not replace the NAT rule. Broadcast ingest on port 8005 must still flow through pfSense NAT directly.

Adding `cloudflared` to the AzuraCast Docker Compose stack:

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token <YOUR_TUNNEL_TOKEN>
    networks:
      - azuracast_network

networks:
  azuracast_network:
    external: true
```

Point the tunnel ingress at `http://192.168.70.4:8090` and configure the tunnel hostname as `radio.aklein.studio` in the Cloudflare dashboard.

## Troubleshooting

### Cloudflare 522

TCP connection timed out at origin. Check in order:

1. AzuraCast health: `curl -sI http://192.168.70.4:8090` should return a 302 redirect to `/login`
2. WAN IP change: `curl -s ifconfig.me` vs the A record in Cloudflare DNS
3. NPM container status: `docker ps | grep npm`
4. cloudflared status if using tunnel: `docker ps | grep cloudflared`
5. Restart Unbound in pfSense under Services > DNS Resolver > Restart since stale internal DNS state has caused spurious 522s before

### Broadcaster Cannot Connect

1. Confirm `dj.aklein.studio` is grey-cloud in Cloudflare
2. Confirm pfSense NAT rule for 8000 to 8005 pointing to 192.168.70.4 is enabled
3. Test from inside LAN: `nc -zv 192.168.70.4 8005`
4. Check AzuraCast station is running and mount point is active in the web UI

### Post-Update 502 / Startup Timeout Loop

After a Dockhand update, AzuraCast may show 502 with `startup` cycling every 3 minutes and `[ERROR] Timed out waiting for services to start` in logs. There are two root causes: stale container environment or MariaDB auth regression.

**Stale container:** Run `docker stop azuracast && docker rm azuracast` then redeploy from Dockhand. Never just restart since a simple restart does not recreate the container and env var changes will not take effect.

**MariaDB auth regression:** Start MariaDB in recovery mode with `--skip-grant-tables`, recreate the azuracast user with full grants on the azuracast database, kill the recovery instance, restart via `supervisorctl start mariadb`, then run `supervisorctl start startup`. Watch logs for `exited: startup (exit status 0; expected)` followed by all services spawning.

Do not add `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_ROOT_PASSWORD`, `REDIS_HOST`, or `REDIS_PORT` to the compose environment. AzuraCast's unified container manages all internal service connections itself. These vars were only needed in the old multi-container setup and conflict with startup logic in current versions.

## Dockhand Compose Location

```
/var/lib/docker/volumes/dockhand_data/_data/stacks/Main-Prod-01/azuracast/compose.yaml
```

Edit directly when CLI access is needed. Always stop, remove the container, and redeploy after any compose change.
