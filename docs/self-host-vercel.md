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

## Known gap: the frontend and the daemon are on two origins

**This deployment is not the topology upstream documents, and the split
breaks the browser's credential handshake.** Everything below describes the
current fork-local arrangement; upstream's documented arrangement is
described at the end of this section.

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

This fork splits them:

```
browser ─▶ blueishdragonz-opendesign.vercel.app   (Vercel static export, NO auth)
             │  document  → 200, no WWW-Authenticate   ← the handshake never starts
             └─ /api, /artifacts, /frames ─▶ funnel ─▶ nginx auth_basic ─▶ daemon
```

The document is served by Vercel with no authentication, so the browser
never receives a 401 challenge for the origin and never caches credentials.
Every `/api`, `/artifacts` and `/frames` request therefore arrives at nginx
without an `Authorization` header and is answered `401`. The app renders its
onboarding/sign-in screen instead of the project list, and files cannot be
opened.

This is **not** a daemon or proxy fault. With credentials supplied, the whole
API surface answers `200` through the same Vercel URL
(`/api/version`, `/api/projects`, `/api/app-config`, `/api/agents`,
`/api/plugins`, `/api/skills`, `/api/design-systems`), and Vercel does
forward the `Authorization` header upstream. Only the handshake is missing.

Two further consequences of the split:

- nginx's `location /` on the funnel proxies to **n8n** (`127.0.0.1:5678`),
  not to the daemon. OpenDesign's own UI is not reachable through the funnel
  at all — only the API paths are. The documented single-origin model would
  put it at the funnel root.
- Upstream's `vercel.json` has **no** rewrites. The catch-all SPA rewrite
  and the three funnel rewrites in this fork are local additions. Upstream's
  supported way to run a separate frontend against a remote daemon is the
  **web sidecar** (`apps/web/sidecar/server.ts`, `OD_WEB_OUTPUT_MODE=server`),
  a Node server that serves the SPA and proxies `/api`, `/artifacts` and
  `/frames` to the daemon — it forwards the browser's `Authorization` header
  unchanged and does not inject credentials of its own. A raw static export
  on Vercel has no equivalent.

### Closing the gap

Make the document and the API share an origin again, so the browser gets one
401 challenge and caches the credentials:

1. Point the Vercel deployment at the daemon for **all** paths (document,
   `/_next/...`, `/api`, `/artifacts`, `/frames`) instead of serving a static
   export — a `/:path*` rewrite to the funnel, no `destination:
   /index.html`.
2. Give nginx an OpenDesign server block that proxies those paths to
   `127.0.0.1:7456` with `auth_basic "OpenDesign"`, including the document.
   n8n currently owns `location /` on that server (it is paused, so it can
   move to a path prefix or to port 8443).
3. Keep `OD_DISABLE_API_AUTH=1` and `OD_API_TOKEN` as they are. That pair is
   upstream's documented "trusted reverse proxy" mode
   (`deploy/README.md`): the proxy authenticates every request, so the
   daemon does not enforce bearer tokens itself.
4. `OD_ALLOWED_ORIGINS` already lists both the Vercel origin and the funnel
   origin, which is what `isLocalSameOrigin` needs behind a proxy.

Note the trade-off: under this topology the browser receives the UI **built
into the daemon image**, not the patched `apps/web` build. To keep this
fork's `useProjectDeleteFlow.ts` fix live, build the image from this fork
(`deploy/Dockerfile`) and run that image on hermes-oci, rather than
`ghcr.io/nexu-io/od:latest`.

## Verifying a deployment

The deployed site is the source of truth; never trust that a build did what
it was supposed to.

```bash
B=https://blueishdragonz-opendesign.vercel.app

# Deep links must serve the SPA shell: 200, 14303 bytes.
# Next's own 404 page is 13316 bytes — a 13316 response means the fallback broke.
for p in / /projects /projects/p1 /settings /oc/foo; do
  printf "%-16s " "$p"; curl -sS -o /dev/null -w "%{http_code} %{size_download}\n" "$B$p"
done

# Trailing slash must redirect, then serve the shell.
curl -sSL -o /dev/null -w "%{url_effective} %{http_code}\n" "$B/projects/"

# Static assets must still serve their REAL bytes. A too-greedy catch-all
# returns the 14 KB HTML shell instead of the asset.
curl -sS -o /dev/null -w "%{http_code} %{size_download} %{content_type}\n" "$B/app-icon.png"

# API proxy (token lives on hermes-oci).
TOKEN=$(ssh hermes-oci 'grep -E "^OD_API_TOKEN=" /home/opc/open-design/.env | cut -d= -f2- | tr -d "\"'"'"'"')
curl -s -o /dev/null -w "%{http_code}\n" -u "blueishdragonz:$TOKEN" "$B/api/health"
```

The delete-flow patch lives in a **lazy-loaded** chunk, so it is not in the
page HTML. Crawl the chunk map (it is in the `turbopack-<hash>.js` runtime,
not in the initial `<script>` tags) and look for the minified branch
`y=r&&!i?"ready"` (fixed) versus `y=r&&!i?"error"` (broken).

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

```bash
cd <this checkout>
pnpm install --frozen-lockfile
pnpm guard && pnpm typecheck          # repo policy before publication
vercel --prod --yes
```

The Vercel project is `blueishdragonz-opendesign` (team `homelab-143b`),
linked via `.vercel/`. `vercel deploy` uploads the **working tree**, so the
local patches must be present (committed or uncommitted) at deploy time.

## Deployment protection

The project is private and Vercel Authentication is enabled. The
**per-deployment** URLs (`blueishdragonz-opendesign-<hash>-homelab-143b.vercel.app`)
redirect to `vercel.com/sso-api` — that is expected and is **not** a sign
that the app is broken. Verify through the project alias
(`blueishdragonz-opendesign.vercel.app`), which is publicly reachable.
