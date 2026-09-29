# hangar-app

The deployment relay for **HANGAR**, live at **https://hangar.intech.studio**.

This repository contains no application code. It holds the credentials and the
workflow that build [`sabotond-dev/hangar`](https://github.com/sabotond-dev/hangar)
and publish it to Cloudflare.

## Why it exists

HANGAR is developed outside our production pipeline. The relay keeps a
credential boundary: the upstream repository stays public and unprivileged, and
this repository is the only thing that can reach the Intech Studio Cloudflare
account. Nothing upstream — no contributor, no agent — can deploy. The only
path to production is a person with write access here pressing **Run workflow**.

## Deploying

Actions → **Deploy HANGAR** → **Run workflow**, or:

```sh
gh workflow run deploy.yml --repo intechstudio/hangar-app -f ref=master
```

The `ref` input takes anything `git checkout` takes — a branch, a tag or a full
commit SHA. It defaults to `master`.

**Rollback** is the same workflow run against the SHA that was good. Cloudflare
also keeps previous versions of the Worker, so `wrangler rollback` is a faster
route if a bad deploy needs undoing before a build completes.

## How it works

1. `sabotond-dev/hangar` is cloned into `upstream/` at the chosen ref, with full
   history.
2. `npm ci && npm run build` (Node 24, from upstream's `.nvmrc`).
3. The build is checked for a clean tree and the presence of the source archive.
4. `_headers` is copied into `upstream/build/`.
5. `wrangler deploy` uploads `upstream/build/` using this repository's
   `wrangler.jsonc`.

### The site is static, and deliberately has no Worker code

Upstream ships its own `wrangler.jsonc` pointing at `worker/index.js` — a
Basic Auth gate used during preproduction. It is **fail-closed**: with no
`SITE_PASSWORD` secret configured it returns 401 for every request.

Our `wrangler.jsonc` declares no `main`, so that Worker is never uploaded and
the site is served entirely from the static asset store. This is what makes the
deployment public. Leaving a secret unset would not make it public — it would
make it unreachable. Anyone reinstating `main` must also set the secret.

### The tslib workaround

The workflow runs `npm install --no-save --no-package-lock tslib@2.8.1` between
install and build. This is a workaround, not configuration.

`@intechstudio/grid-protocol` imports `tslib` but does not declare it as a
dependency. On macOS the build works anyway, because `tslib` arrives as an
optional transitive dependency of `@img/sharp-wasm32` and
`@tailwindcss/oxide-wasm32-wasi` by way of `@emnapi/runtime`. On Linux, npm
installs the native builds of those packages instead, the wasm32 variants are
skipped, `tslib` is never installed, and the build fails with
`ERR_MODULE_NOT_FOUND`.

**The real fix is a `@intechstudio/grid-protocol` release that declares `tslib`
as a dependency** — it is our own package, and this will affect any Linux
consumer of it, not just this pipeline. Once that ships, delete the step.

The version matches what upstream's lockfile already pins for the optional
entry. `--no-save --no-package-lock` leaves `package.json` and
`package-lock.json` untouched, so the clean-tree check still means what it says.

### Response headers

`_headers` in this repository is copied into the build directory on every
deploy. Cloudflare reads it from there and does not serve it.

It currently carries one rule: `/dev/*` responds with
`X-Robots-Tag: noindex, nofollow`. Those seven pages are internal development
views — unlinked from the site, but publicly reachable, and `robots.txt` allows
crawling everything. Upstream relied on the preproduction Basic Auth worker's
`X-Robots-Tag` to keep them out of search results, and that worker is not
deployed here.

To keep another path out of search results, add a pattern to `_headers`. Prefer
that over a `robots.txt` Disallow: Disallow stops crawlers reading the page at
all, so they never see the noindex, and the URL can still be indexed from an
external link.

## Configuration

| Secret | Where it comes from |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | Cloudflare → My Profile → API Tokens, "Edit Cloudflare Workers" template, scoped to the Intech Studio account and the `intech.studio` zone |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare dashboard → Workers & Pages sidebar |

Both are set on the `production` GitHub environment. To rotate, create a new
token in Cloudflare, `gh secret set CLOUDFLARE_API_TOKEN --env production`, then
revoke the old one.

DNS needs no manual setup: `custom_domain: true` in `wrangler.jsonc` makes
Cloudflare create the `hangar.intech.studio` record and issue the certificate on
deploy.

## Licensing

Upstream is `Copyright (C) 2026 Botond Sandor`, **GPL-3.0-or-later**.

The HANGAR bundle runs in the visitor's browser, which makes serving the site
distribution of object code rather than mere network use. Corresponding Source
is therefore offered alongside it: `npm run build` writes
`build/source-<sha>.tar.gz` and the page footer links to it. That archive ships
as part of every deploy — do not strip it, and do not deploy a `build/`
directory that was assembled by anything other than upstream's own build.

The **GRIFTER** typeface is served by the site but excluded from the source
archive (`export-ignore` in upstream's `.gitattributes`), because the foundry
licence covers use, not redistribution. Intech Studio holds that licence, which
is what permits serving it from `intech.studio`.

## Project lifecycle

| Stage | State |
| --- | --- |
| Development | `sabotond-dev/hangar`, outside this pipeline |
| Preproduction | `hangar.sabotond.workers.dev`, Basic Auth gated, Botond's Cloudflare account |
| Production | `hangar.intech.studio`, public, Intech Studio's Cloudflare account, deployed from here |

Deploys are manual and on request. This repository does not watch upstream and
will not deploy anything by itself.
