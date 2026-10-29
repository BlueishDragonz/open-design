# Self-hosting the OpenDesign web frontend on Vercel

This fork carries a small set of **local patches** that make the upstream
web frontend work as a self-hosted deployment. They are listed below so they
can be reviewed, re-applied after an upstream merge, and dropped as soon as
upstream provides an equivalent.

Scope: this document covers the **web frontend deployment** only. The
OpenDesign **daemon** runs separately on `hermes-oci` (Oracle Cloud), reached
over a Tailscale funnel.

## Architecture

```
browser
  │
  ├─ https://blueishdragonz-opendesign.vercel.app   (Vercel static export)
  │     └── /api/*, /artifacts/*, /frames/*  ──rewritten──▶  Tailscale funnel
  │                                                                │
  │                                                                ▼
  │                                                   nginx (Basic Auth) on hermes-oci
  │                                                                │
  │                                                                ▼
  │                                                   OpenDesign daemon (:7456)
  │                                                   + shared volume open_design_data
  ▼
all navigation is client-side (src/router.ts, pushState)
```

The app is a **client-routed SPA**: `apps/web/app/[[...slug]]/page.tsx` is a
single optional catch-all route, and `src/router.ts` reads `window.location`
at runtime to decide what to render. Only three route shells are ever
requested by the browser — the document, its JS chunks, and the RSC payload
for the current path.

## Local patches

### 1. `vercel.json` — SPA fallback + API proxy  (the important one)

`generateStaticParams()` returns only `[{ slug: [] }]`, so a static export
contains **exactly one** HTML shell (`out/index.html`) plus `404.html` — there
is no per-route HTML. The daemon has its own SPA fallback
(`apps/daemon/src/server.ts`) that serves the shell for any unknown non-API
path. **Vercel has no such fallback by default**, so every deep link
(`/projects/<id>`, `/settings`, `/oc/<slug>`, …) served Next's own
"404: This page could not be found." page even though the app was working
perfectly.

The patch adds:

- three rewrites sending `/api/*`, `/artifacts/*`, `/frames/*` to the
  Tailscale funnel that fronts the daemon, and
- a catch-all rewrite `/:path*` → `/index.html` so deep links serve the SPA
  shell, and
- a `redirects` entry that strips a trailing slash (a rewrite never sees
  trailing-slash paths: Vercel resolves the filesystem first and 404s before
  the rewrite phase runs).

**Rewrite order matters.** Vercel applies rewrites in array order, first
match wins, so the three funnel rewrites must stay ahead of the catch-all.

The `redirects` source uses a negative lookahead
(`(?!api/|artifacts/|frames/|_next/)`) so the API proxy and static assets
are never caught by the trailing-slash redirect.

> **Before editing the rewrite destinations:** the funnel hostname is
> `hermes-vnic-1.taila8a61a.ts.net` and it must match the live Tailscale
> funnel on `hermes-oci`. If the funnel hostname changes, update all three
> destinations or every `/api` call 404s.

### 2. `apps/web/src/components/project-actions/useProjectDeleteFlow.ts`

In self-hosted mode the daemon exposes **no workspace**, so
`/api/projects/:id/share-state` returns `400 WORKSPACE_CONTEXT_REQUIRED`.
The hook mapped that to `shareReadStatus = 'error'`, so `commit()` returned
early forever and the project-delete button stayed disabled ("Something went
went wrong") — even though the daemon's `DELETE /api/projects/:id` returns
`200 {"ok":true}`.

The patch treats "no workspace context" as "no active shares"
(`'ready'` and `0`).

### 3. `.gitignore`

Ignores `.vercel/` (Vercel CLI's local project-link state) so it is not
treated as repository content.

### Known upstream bug NOT patched here

`apps/web/src/components/project-actions/activeFileSharesBeforeDelete.ts`
throws `workspace identity unavailable for shared HTML deletion` when
`context` is null, which in self-hosted mode is always. The daemon's
`DELETE /api/projects/:id/files/:name` works (200), so the bug is purely a
frontend gate — but the correct fix (`if (!context) return 0;`) touches
upstream-shared share-flow logic and is left for a deliberate upstream PR
rather than a local patch.

## Topology: one origin, as upstream documents it

Upstream ships the web UI and the API from **one** origin. The published
container serves both: `deploy/docker-compose.yml` sets `OD_PORT=7456` and
`OD_WEB_PORT=7456`, and `deploy/README.md` states it outright — *"The daemon
serves both the API and the built Next.js static export, so there is no
separate nginx container."* The daemon answers `/` with the SPA shell and
`/_next/static/...` with the assets, and `apps/daemon/src/server.ts`
challenges the **document navigation** with
`WWW-Authenticate: Basic realm="OpenDesign"` before it serves shell bytes.
The browser prompts once, caches those credentials for the origin, and every
later same-origin `/api` call carries the header. The whole design depends
on the document and the API sharing an origin.

This deployment now does the same, in two hops:

```
browser ─▶ blueishdragonz-opendesign.vercel.app        (Vercel, pure reverse proxy)
             │  every path ─▶ funnel :8443 ─▶ nginx :8009 ─▶ daemon :7456
             │                  (auth_basic "OpenDesign" on ALL paths)
             └─ one origin, one 401 challenge, browser caches the credentials
```

- **Vercel** is a pure proxy: `vercel.json` has a single `routes` entry
  (`/(.*)` → the funnel on `:8443`) and builds nothing. It must be `routes`,
  not `rewrites` — with `rewrites` Vercel serves its own filesystem first,
  so `/` returns `404` before the rewrite is ever considered.
- **nginx** has a dedicated server block on `:8009`
  (`/etc/nginx/conf.d/opendesign-proxy.conf`) that proxies every path to the
  daemon with `auth_basic "OpenDesign"`. It is a separate block because
  nginx's `location /` on `:8080` belongs to **n8n** (the editor), and
  sharing that server block would put the document and the API on two
  origins again — the exact failure this topology avoids.
- **The daemon** serves the SPA shell, the assets and the API, and its own
  SPA fallback handles deep links (`/projects/<id>`, `/settings`,
  `/oc/<slug>`) and trailing slashes with no extra rewrite.

n8n keeps the funnel on `:443` → nginx `:8080` → n8n on `127.0.0.1:5678`.
The two services share the VM, the nginx instance and the tailnet node
without sharing a port, a path or a credential realm.

### Why the daemon image is built from this fork

Under this topology the browser receives the UI **built into the daemon
image**, so the `useProjectDeleteFlow.ts` fix is only live if the image is
built from this fork. The image on hermes-oci is
`open-design:fork-chromium-2026-10-29`: `deploy/Dockerfile` built from
`BlueishDragonz/open-design`, with the host's Chromium and CLI-tooling
layers applied on top (`/home/opc/open-design-build/Dockerfile.local`, whose
`FROM` line points at the fork build instead of
`ghcr.io/nexu-io/od:latest`).

To rebuild after an upstream merge:

```bash
ssh hermes-oci
cd /home/opc/src && curl -fsSL -o od.tar.gz \
  https://codeload.github.com/BlueishDragonz/open-design/tar.gz/refs/heads/main
tar xzf od.tar.gz && mv open-design-main open-design && rm od.tar.gz
cd open-design && docker build -f deploy/Dockerfile -t open-design:fork-2026-10-29 .
cd /home/opc/open-design-build
sed 's|^FROM ghcr.io/nexu-io/od:latest$|FROM open-design:fork-2026-10-29|' \
  Dockerfile.local > Dockerfile.fork
docker build -f Dockerfile.fork -t open-design:fork-chromium-2026-10-29 .
cd /home/opc/open-design && docker compose up -d --force-recreate --no-deps open-design
```

There is **no git on hermes-oci** — fetch the source as a codeload tarball,
as above.

### The earlier two-origin failure, for the record

The first attempt at this deployment served the SPA as a **Vercel static
export** and rewrote only `/api`, `/artifacts` and `/frames` to the funnel.
The document came back `200` with no `WWW-Authenticate`, so the browser
never received a challenge for the origin and never cached credentials.
Every API call then reached nginx without an `Authorization` header and was
answered `401`: the app rendered its onboarding/sign-in screen, projects
did not load, and files could not be opened. With credentials supplied the
whole API surface answered `200` through the same URL — the daemon and the
proxy were never at fault, only the missing handshake.

The symptom to watch for is a `200` document with `401` API calls. The fix
is always the same: the document must be challenged on the same origin the
API calls use.

### Unused: the old two-origin arrangement

The earlier text in this file described a Vercel-served static export with
selective funnel rewrites. That arrangement is gone, and the notes below are
kept only as a record of the failure mode.

The failure was: the document came back `200` with no
`WWW-Authenticate`, so the browser never received a challenge for the origin
and never cached credentials; every API call then reached nginx without an
`Authorization` header and was answered `401`. The daemon and the proxy were
never at fault — with credentials supplied the whole API surface answered
`200` through the same URL.

## Verifying a deployment

The deployed site is the source of truth; never trust that a build did what
it was supposed to.

```bash
B=https://blueishdragonz-opendesign.vercel.app
PW=$(ssh hermes-oci 'grep -E "^OD_API_TOKEN=" /home/opc/open-design/.env | cut -d= -f2-')

# 1. THE HANDSHAKE. The document must be challenged with the OpenDesign
#    realm, on the same origin the API uses. A 200 here means the handshake
#    is broken and every feature will fail with 401s.
curl -sSI "$B/" | grep -iE "^HTTP|www-authenticate"
#   expect: HTTP/2 401 + www-authenticate: Basic realm="OpenDesign"

# 2. With credentials, the document and the API are all served.
for p in / /projects /oc/foo /api/health /api/projects; do
  printf "%-16s " "$p"
  curl -sS -u "blueishdragonz:$PW" -o /dev/null -w "%{http_code} %{size_download}\n" "$B$p"
done

# 3. Static assets must serve their REAL bytes, not the 14 KB HTML shell.
curl -sS -u "blueishdragonz:$PW" -o /dev/null \
  -w "%{http_code} %{size_download} %{content_type}\n" "$B/_next/static/chunks/<hash>.js"

# 4. SSE must stream (the app's live channels).
curl -sS -u "blueishdragonz:$PW" -N --max-time 8 "$B/api/memory/events"
#   expect: "event: connected"
```

The delete-flow patch lives in a **lazy-loaded** chunk, so it is not in the
page HTML. Crawl the chunk map (it is in the `turbopack-<hash>.js` runtime,
not in the initial `<script>` tags) and look for the minified branch
`y=r&&!i?"ready"` (fixed) versus `y=r&&!i?"error"` (broken).

Because the UI now ships inside the daemon image, verify the patch in the
**running container**, not in a local build:

```bash
ssh hermes-oci 'docker exec open-design sh -c \
  "grep -rhoE \".{0,60}delete-confirmation.{0,90}\" /app/apps/web/out/_next/static/chunks | head -2"'
#   expect: ...y=r&&!n?"ready":f,k=r&&!n?0:h?h.publications...
```

## Keeping this fork in step with upstream

Upstream: `https://github.com/nexu-io/open-design` (remote `upstream`).
This fork: `https://github.com/BlueishDragonz/open-design` (remote `origin`).

```bash
git fetch upstream
git log --oneline HEAD..upstream/main          # what upstream added
git log --oneline HEAD..upstream/main -- \
  vercel.json \
  .gitignore \
  apps/web/src/components/project-actions/useProjectDeleteFlow.ts
```

If upstream has **not** touched the patched files, a rebase/merge is
conflict-free:

```bash
git rebase upstream/main      # or: git merge upstream/main
git push --force-with-lease origin main
```

If upstream **has** touched them, review each patch in turn — the local fix
may now be redundant (drop it) or may have been reimplemented upstream
(rename the patch to a no-op and keep only what is still needed).

**Upgrade only when it buys something.** Upstream lands changes daily;
pulling it in does not make the self-hosted deployment better by itself. It
is worth the merge when upstream fixes a bug you are hitting, when a
security fix lands, or when a patch here can be dropped because upstream
now provides the equivalent. Otherwise a stale fork is cheaper and less
risky than a merge.

## Deploying

The Vercel project is a **pure reverse proxy** — it builds nothing, so
deploying it is just a config push:

```bash
cd <this checkout>
vercel deploy --prod --yes --archive=tgz
```

`--archive=tgz` is required: this repo has more than 15000 files and Vercel
rejects a plain upload with `missing_archive`.

The Vercel project is `blueishdragonz-opendesign` (team `homelab-143b`),
linked via `.vercel/`.

The **application** is deployed separately, on hermes-oci, as a Docker image
built from this fork — see "Why the daemon image is built from this fork"
above for the rebuild recipe. Changing `apps/web` or `apps/daemon` here
requires rebuilding that image and recreating the container; the Vercel
deploy alone does not ship application code.

## Deployment protection

The project is private and Vercel Authentication is enabled. The
**per-deployment** URLs (`blueishdragonz-opendesign-<hash>-homelab-143b.vercel.app`)
redirect to `vercel.com/sso-api` — that is expected and is **not** a sign
that the app is broken. Verify through the project alias
(`blueishdragonz-opendesign.vercel.app`), which is publicly reachable.

## n8n on the same VM

n8n and OpenDesign share hermes-oci, the nginx instance and the tailnet
node, but nothing else:

| | n8n | OpenDesign |
|---|---|---|
| Funnel port | 443 | 8443 |
| nginx server block | `:8080` in `n8n-proxy.conf` | `:8009` in `opendesign-proxy.conf` |
| Upstream | `127.0.0.1:5678` | `127.0.0.1:7456` |
| Auth realm | `n8n` | `OpenDesign` |
| htpasswd | `/etc/nginx/.htpasswd` | `/etc/nginx/.htpasswd-opendesign` |

Both blocks live in `/etc/nginx/conf.d/`, so `nginx -t && nginx -s reload`
picks up a change to either. Do not merge the two blocks: nginx picks a
server block by `listen` + `server_name`, and sharing one would put two
services' documents behind one credential realm.
