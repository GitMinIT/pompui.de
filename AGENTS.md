# Agent Guide for pompui.de

## Development status — activity subscription wheel

The activity subscription wheel incorporates the Sites version 19 geometry and visibility fixes. The active tile has a hard visibility guarantee, edge fading is position-based, the wheel radius grows from a minimum mathematical tile distance, and the visual wheel depth remains constant while the tilt adapts to the radius.

The previous blocking issue in which the active tile could disappear is resolved in this implementation. Desktop and mobile behavior still need to pass the normal repository review and CI checks before deployment.

## Environment
- **Project Root**: `/var/www/pompui.de`
- **Testing Host**: `/home/daniel/Projects/pompui.de` (HomeGate home server — implement, test and debug only; nothing is published from here, HTTP-only workflow)
- **Domain**: `pompui.de` (+ `www.pompui.de`)
- **Architecture**: Dockerized. Static nginx containers; production traffic enters via the **shared global-proxy** (`global-proxy` container, owned by the daniel-hettich.de project) which terminates SSL on ports 80/443 and routes by `server_name`.
- See `ENVIRONMENT.md` for full server details.

## Testing Host (HomeGate)
- The stack runs via a compose override: `docker compose -f docker-compose.yml -f infrastructure/compose/homegate-testing.yml up -d --build`
- Containers are exposed **directly** (HTTP, no proxy, no TLS), one port per site:
  - `pompui.de` (landing) → **6010**
  - `punctum.pompui.de` → **6012** (6011 is taken by open-webui on this host)
  - `gj.pompui.de` → **6013**
  - `snapotter.pompui.de` → **6014** (embedded PostgreSQL/Redis; first boot takes a while; needs `secrets/snapotter-password`)
  - `vtracer.pompui.de` → **6015** (heavy one-time build: Rust→wasm + webpack)
  - `metronome.pompui.de` → **6022** (static Vite build; 6016–6018 are taken by other services on this host)
  - `peercalls.pompui.de` → **6023** (Peer Calls Go server, container port 3000; build context is the sibling clone `../PeerCalls`)
  - **STUN/TURN (`pompui-coturn`)**: **3478 udp/tcp** + **49160–49200 udp** published directly on the host (UDP cannot be reverse-proxied — the one exception to the "only 80/443" rule). Internet use needs firewall rules for those ports and `COTURN_EXTERNAL_IP` (public IP) in `.env`; the homegate override forces the LAN IP. TURN REST auth via `COTURN_SECRET` in `.env` (git-ignored, chmod 600).
- The external `web-network` join is overridden away — the daniel-hettich.de stack does not need to be running for testing this repo.
- daniel-hettich.de testing runs directly on **6009** from the daniel-hettich.de repo (see its AGENTS.md).
- Verify: `curl http://192.168.178.60:6010/` (or `http://<LAN-IP>:<port>` from any device).

## Credentials & Secrets
- **GitHub Repository**: `https://github.com/GitMinIT/pompui.de.git`
- **GitHub Token**: store in `secrets/github.token` (git-ignored) or let git use a credential helper; never commit it.
- **SSL Certificates**: `/etc/letsencrypt` on host → `/etc/nginx/ssl` inside `global-proxy`. Current cert only covers `*.daniel-hettich.de` — a pompui.de cert is still needed.

## Repo Layout
- `sites/landing-page` — the landing page (this repo's core product).
- `sites/landing-page/html/sites/snapotter/` — served AGPL licence-compliance docs for SnapOtter (image lives upstream).
- `repos/` — **hosted apps as separate git repositories**, cloned into this tree; each is built and routed like any other container. See `repos/README.md` for the full how-to (add an app: clone repo, compose service, carousel entry, subdomain, proxy conf).
- Garden Journal lives in its own repo (`https://github.com/DaScoob/Garden-Journal.git`); Punctum in `https://github.com/GitMinIT/Punctum`; SnapOtter upstream is `https://github.com/snapotter-hq/SnapOtter` (AGPL — clone pinned at the running tag into `repos/snapotter`, image digest-pinned in compose).

## Infrastructure Map
- **global-proxy** (external, from the daniel-hettich.de compose): nginx:alpine, SSL termination, routes:
  - `pompui.de` / `www.pompui.de` → `pompui-landing:8080`
  - `punctum.pompui.de` → `pompui-punctum:8080`
  - `snapotter.pompui.de` → `pompui-snapotter:1349`
  - `vtracer.pompui.de` → `pompui-vtracer:8080`
  - `metronome.pompui.de` → `pompui-metronome:8080`
  - `peercalls.pompui.de` → `pompui-peercalls:3000` (WebSocket signalling; media is P2P)
- **pompui-coturn** (not proxied): STUN/TURN on host UDP/TCP 3478 + UDP 49160–49200; `turn.pompui.de` resolves to the host via the wildcard DNS record — no nginx conf needed.
- **pompui-landing**: static landing page (nginx-unprivileged:alpine, non-root, port 8080).
- **pompui-punctum**: Punctum app (separate repo in `repos/punctum`, static nginx, port 8080).
- **pompui-snapotter**: SnapOtter 2.2.0 file-processing suite (AGPL-3.0, embedded PostgreSQL/Redis, port 1349). **Licence compliance is mandatory** — see `sites/landing-page/html/sites/snapotter/COMPLIANCE.md` before updating it: pin image digest, update source offer, keep `repos/snapotter` checkout at the running tag. Auth enabled, telemetry off, password in `secrets/snapotter-password`.
- **pompui-vtracer**: VTracer webapp (raster→vector, MIT/Apache-2.0, built from source via wasm-pack+webpack, port 8080). Clone at `repos/vtracer` currently tracks **DaScoob/vtracer:feature/pompui-ui** (POMPUI UI fork; fork added as remote `fork`), NOT upstream master. Client-side processing only (wasm in browser).
- **pompui-metronome**: Metronome webapp (browser metronome, Web Audio, speed trainer) from `repos/metronome` (github.com/mackushev/metronome). Static Vite build (`--base=/`) served by nginx-unprivileged, port 8080. Fully client-side; settings in localStorage.
- **pompui-peercalls**: Peer Calls (WebRTC group video calls, Apache-2.0) from the sibling clone `../PeerCalls` — our fork `github.com/GitMinIT/peer-calls`, remotes `fork` (push) / `upstream` (read-only), currently on branch **`feature/pompui-dark-theme`** (POMPUI dark theme; workflow for app changes: branch off `upstream/master` → commit → `git push fork <branch>` → checkout here → rebuild). Build via the upstream multi-stage Dockerfile (webpack frontend + Go server in a scratch image), port 3000. Mesh mode + in-memory store (no Redis); the container only handles HTTP + `/ws` signalling — media flows P2P. Runs as uid 65534, all caps dropped. ICE servers point at our own **pompui-coturn** (`PEERCALLS_ICE_SERVER_URLS`, TURN REST auth via `PEERCALLS_ICE_SERVER_SECRET` = `COTURN_SECRET` from `.env`) — no Google/Twilio STUN, no third-country transfer (datenschutz §12/§15). Carousel `repo:` and Impressum link the fork since it diverges from upstream.
- **pompui-coturn**: coturn 4.19.2-r0 (MIT), STUN/TURN for Peer Calls. `use-auth-secret` + `static-auth-secret=${COTURN_SECRET}` (pompui-peercalls mints 24h HMAC credentials; NOT an open relay). Needs `cap_add: NET_BIND_SERVICE` (turnserver exits EPERM without it). Host-published 3478 udp/tcp + 49160–49200 udp; `--external-ip=${COTURN_EXTERNAL_IP}` (public IP in production; homegate override forces 192.168.178.60). Secrets in `.env` (git-ignored): `COTURN_SECRET`, `COTURN_EXTERNAL_IP`.

## Deployment
1. `docker compose up -d --build` in this directory (builds `pompui-landing`, joins external `web-network`).
2. Copy `infrastructure/nginx/conf.d/pompui-landing.conf` into `/var/www/daniel-hettich.de/infrastructure/nginx/conf.d/` (this is the directory volume-mounted into `global-proxy`).
3. **Restart** the proxy whenever containers were **recreated** (new IPs): `docker restart global-proxy`.
   - ⚠️ Gotcha: `nginx -s reload` alone does NOT re-resolve upstream container names — recreated containers get new IPs and the proxy keeps connecting to stale ones (502s with `Connection refused` to wrong IPs). A full restart re-resolves.
   - `nginx -s reload` is fine after conf *text* changes when no container was recreated.
4. Ensure a valid pompui.de certificate exists in `/etc/letsencrypt` and fix the cert paths in the conf file.
5. ⚠️ Gotcha: the proxy-wide `X-Frame-Options: DENY` also applies to **non-HTML responses**. Anything a page embeds as `<object>`/`<iframe>` (e.g. `<object data="*.svg">`) is a nested navigation and gets blocked by Chrome with `net::ERR_BLOCKED_BY_RESPONSE` — the element renders empty with no obvious error on the page. The pompui-landing conf therefore carries a `location ~* \.svg$` block that re-declares the headers without XFO (nginx `add_header` does not inherit into nested locations) and adds CSP `frame-ancestors 'self'` instead. If you embed other subresources as documents (PDFs, XML…), extend that pattern to their extensions.

## Scaling & Extensibility
- To add a new hosted app, follow `repos/README.md` (clone repo → compose service → carousel entry → nginx conf → proxy mount).
