<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-05-27 | Updated: 2026-05-27 -->

# stacks-traefik

## Purpose
Edge router for the home lab. Traefik v3 listens on a dedicated macvlan IP (`192.168.26.140`) on the `vlan-containers` network, terminates TLS using Let's Encrypt with the Route53 DNS-01 challenge, and routes to backend services attached to the `traefik_internal` Docker network. **This stack is the prerequisite for every other stack in the repo** — they all attach to `traefik_internal` and rely on its `route53resolver` cert resolver.

## Key Files
| File | Description |
|------|-------------|
| `docker-compose.yml` | Traefik v3.4.1 service: macvlan IP, ACME/Route53 config, dashboard, dynamic-file-provider mount |
| `README.md` | Title only — no prose docs |

## For AI Agents

### Working In This Directory
- **Don't change the macvlan IP (`192.168.26.140`).** It's referenced as the `entryPoints.web/websecure` bind address *and* as the IP other LAN devices and DNS records point at. Changing it breaks every published service simultaneously.
- The `vlan-containers` network is an external macvlan and is **host/router-specific**; recreating it has to be coordinated with the Synology + LAN config.
- ACME state lives at `/volume3/docker/traefik/hostconfig/acme.json` on the host. **Never delete it** — Let's Encrypt rate-limits cert issuance and a fresh start can lock you out for a week. Back it up before any destructive change.
- Dynamic file-provider configs live at `/volume3/docker/traefik/hostconfig/` (mounted at `/hostconfig/`, watched). Edits there are picked up live — they are **not in this repo** by design.
- `--api.insecure=true` is exposed only because the dashboard router is bound to a TLS host (`traefik.home.sea.nlea.ch`) and reachable only on `traefik_internal`. If the network topology changes, re-evaluate this flag.
- AWS credentials for the Route53 DNS-01 challenge come from `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` env vars set in Portainer's stack environment. They must have permission to write TXT records in the relevant hosted zone (`sea.nlea.ch`).
- Compose-v3 + Portainer Git-stack — no CLI-only features.

### Testing Requirements
- No tests. Validate by:
  1. `docker compose config` parses.
  2. After redeploy: `https://traefik.home.sea.nlea.ch` loads the dashboard with a valid cert.
  3. Other stacks' hosts (e.g. `paperless.home.sea.nlea.ch`, `zigbee.home.sea.nlea.ch`) still resolve and serve their backends.
  4. Container logs show no ACME errors and no "router has no service" warnings.

### Common Patterns
- All routers/services for backends live on the **backend container's** compose labels, not here. This stack only configures the router itself + its dashboard.
- Cert resolver name `route53resolver` is referenced by every backend stack — don't rename it without a coordinated rollout.

## Dependencies

### Internal
- None — this is the bottom of the stack. Other stacks (`stacks-paperless`, `stacks-home-automation-tooling`) depend on this one being up.

### External
- `traefik:v3.4.1` Docker image.
- External Docker networks: `vlan-containers` (macvlan), `traefik_internal` (overlay/bridge for backend wiring).
- AWS Route53 hosted zone for `sea.nlea.ch` (DNS-01 challenge target).
- Host paths: `/var/run/docker.sock` (provider), `/volume3/docker/traefik/hostconfig` (ACME state + file provider), `/volume3/docker/traefik/logs`.

<!-- MANUAL: -->
