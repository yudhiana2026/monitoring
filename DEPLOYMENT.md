# Monitoring Stack Deployment and Release Notes

Status: pre-deployment guide for `docker-compose.monitoring.yml`.

## Release summary

- Deploys Beszel Hub, Beszel Agent, Dozzle, and a restricted Docker socket proxy.
- Does not deploy Dockge or any Compose-management UI.
- Publishes no Docker host ports. Public access is only through the existing Nginx container over HTTPS.
- Persists Beszel and Dozzle data in named volumes.
- Keeps Dozzle actions and shell access disabled, and restricts Docker API access to the socket proxy.

## Architecture and limits

Nginx must join the external `monitoring-proxy` network. It then proxies to `beszel:8090` and `dozzle:8080`; the socket proxy and Agent remain on `monitoring-internal` only.

The Agent intentionally does not use host networking, so the stack needs no host port. Docker CPU, memory, network, and container-state metrics remain available through the Docker API. Host network-interface statistics are less complete than with `network_mode: host`; use `docker stats` when you need an immediate container Block I/O snapshot.

Normal idle use is expected to be roughly 43-75 MiB. The Compose memory limits are safety ceilings, not expected consumption.

## Prerequisites

- Docker Engine and Docker Compose plugin are installed on the server.
- The public Nginx instance runs as a Docker container. If Nginx is a native host process, this no-port design does not apply.
- DNS and TLS already route `beszel.<your-domain>` and `dozzle.<your-domain>` to Nginx.
- An Nginx authentication method is available. The provided example uses Basic Auth; SSO is also suitable.
- You have a backup or snapshot of the current Nginx configuration before changing it.

## 1. Create the shared proxy network

Inspect the network first:

```bash
docker network inspect monitoring-proxy
```

If it does not exist, create it once:

```bash
docker network create monitoring-proxy
```

Do not create this network through a temporary Compose project. It is external because both the Nginx and monitoring projects use it.

## 2. Connect Nginx to the shared network

In the existing Nginx Compose file, retain every current network and add `monitoring-proxy` to the Nginx service:

```yaml
services:
  nginx:
    networks:
      # Keep existing application networks here.
      - monitoring-proxy

networks:
  monitoring-proxy:
    external: true
    name: monitoring-proxy
```

Copy the relevant server blocks from `nginx-monitoring.conf.example` into the live Nginx configuration. Replace both example domain names and use the existing TLS-certificate directives.

The example requires this protected file in the Nginx container:

```text
/etc/nginx/secrets/monitoring.htpasswd
```

Create the password file using the existing server secret-management procedure, then mount it read-only into Nginx. Do not remove `auth_basic` from the Dozzle virtual host unless equivalent SSO/forward authentication is already enforced.

Validate Nginx before applying it. Replace placeholders with the actual Nginx Compose file and service name:

```bash
docker compose -f <nginx-compose-file> exec -T <nginx-service> nginx -t
docker compose -f <nginx-compose-file> up -d <nginx-service>
```

## 3. Configure monitoring environment

```bash
cd /home/yudhiana/projects/personal/monitoring
cp .env.example .env
chmod 600 .env
```

Edit `.env` and set the actual public URL:

```dotenv
BESZEL_APP_URL=https://beszel.<your-domain>
```

Leave `BESZEL_AGENT_KEY` and `BESZEL_AGENT_TOKEN` empty for now. They are created from the Beszel UI after the Hub starts.

## 4. Validate and deploy the core stack

```bash
docker compose -f docker-compose.monitoring.yml config --quiet
docker compose -f docker-compose.monitoring.yml up -d
docker compose -f docker-compose.monitoring.yml ps
docker compose -f docker-compose.monitoring.yml logs --tail=100 beszel dozzle docker-socket-proxy
```

The default deployment starts Beszel, Dozzle, and the socket proxy. It does not start `beszel-agent` yet.

Acceptance criteria:

- Nginx can resolve `beszel` and `dozzle` on `monitoring-proxy`.
- `https://beszel.<your-domain>` renders the Beszel login/setup screen.
- `https://dozzle.<your-domain>` requests Nginx authentication before showing logs.
- `docker compose ... ps` shows the three core services as running; Beszel and Dozzle become healthy after their healthcheck interval.

## 5. Configure and start Beszel Agent

1. Sign in to Beszel through its HTTPS domain and create the initial administrator account.
2. In Beszel, add a local system. Use `/beszel_socket/beszel.sock` as the local Agent host/path and copy the generated `KEY` and `TOKEN`.
3. Put those values in `.env`. Treat the token as a secret.
4. Validate and start the Agent profile:

```bash
docker compose -f docker-compose.monitoring.yml --profile agent config --quiet
docker compose -f docker-compose.monitoring.yml --profile agent up -d beszel-agent
docker compose -f docker-compose.monitoring.yml --profile agent ps
docker compose -f docker-compose.monitoring.yml logs --tail=100 beszel-agent
```

The system should turn green in Beszel and begin showing Docker containers and historical metrics after its first collection interval.

## 6. Post-deployment checks

```bash
docker compose -f docker-compose.monitoring.yml stats --no-stream
docker compose -f docker-compose.monitoring.yml ps
```

Confirm all of the following:

- No monitoring host port is listed by `docker ps`.
- No monitoring service has a public port binding; public dashboard traffic terminates at Nginx.
- Dozzle cannot start, stop, recreate, or open shells in containers.
- Beszel displays the existing Docker containers and their CPU, memory, and network history; `docker stats --no-stream` reports the immediate PostgreSQL Block I/O snapshot.
- Nginx access logs contain dashboard traffic, and no upstream connection failures.

## Rollback

1. Restore the previous Nginx virtual-host configuration and redeploy Nginx.
2. Stop the monitoring services without deleting persistent data:

```bash
docker compose -f docker-compose.monitoring.yml --profile agent down
```

Do not append `-v`; that would delete the Beszel and Dozzle named volumes. The external `monitoring-proxy` network is intentionally preserved.

## Updating images

Change one pinned image version at a time, validate the Compose model, then deploy it:

```bash
docker compose -f docker-compose.monitoring.yml config --quiet
docker compose -f docker-compose.monitoring.yml pull
docker compose -f docker-compose.monitoring.yml --profile agent up -d
```

Check logs and dashboard health before updating the next image. Never replace a pinned image with `latest` in an unattended production deployment.

## Runtime boundary

This guide and the Compose file have passed syntax/interpolation validation. They do not prove the external network, Nginx TLS/authentication, image pulls, Docker permissions, or Beszel agent connection on the target server. Verify every acceptance criterion during the server deployment.
