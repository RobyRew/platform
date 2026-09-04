# platform

Shared build and UI plumbing for the RobyRew app fleet.

Exists so that a change is made **once** rather than copied into every repo — while
each app repo stays independently forkable and buildable on its own. Nothing here
is vendored into the apps; they reference it by version.

## What lives here

| Path | Purpose |
|---|---|
| `.github/workflows/build-image.yml` | Reusable workflow: build the app's Dockerfile on GitHub's runners, push to GHCR, then ask Dokploy to pull it. |
| `images/static-web/` | `ghcr.io/robyrew/static-web` — pinned, non-root nginx runtime with one maintained security-header set. |

## static-web

```dockerfile
FROM ghcr.io/robyrew/static-web:1
ENV WEB_FALLBACK=/index.html                     # SPA; omit for multi-page
ENV CSP_CONNECT_EXTRA="https://stats.cosmincalin.es"
COPY --from=build --chown=nginx:nginx /app/dist /usr/share/nginx/html
```

Runs as uid 101 on **8080** — a Dokploy domain pointing at port 80 must be changed.

| Variable | Default | Purpose |
|---|---|---|
| `WEB_PORT` | `8080` | listen port |
| `WEB_FALLBACK` | `=404` | `try_files` fallback. `/index.html` for an SPA; `$uri/index.html $uri.html =404` for Astro-style extensionless URLs |
| `CSP_SCRIPT_EXTRA` etc. | empty | widen one CSP directive instead of restating the policy |

App-specific redirects and cache rules go in `/etc/nginx/robyrew/app.d/*.conf`.

**nginx `add_header` does not merge.** Any block that sets its own header discards
every inherited one, so an app.d block that sets `Cache-Control` must also
`include /etc/nginx/robyrew/security-headers.conf;` or that response ships with no
CSP at all.

Planned (see the infrastructure repo's plan file):

- `@robyrew/ui` — shared navbar, footer and the liquid-glass design tokens
- `@robyrew/web-config` — shared Astro / Tailwind / tsconfig preset and one CSP policy
- `ghcr.io/robyrew/static-web` — pinned, non-root nginx base image

## Using the build workflow

In an app repo, `.github/workflows/build-image.yml`:

```yaml
name: Build and publish image

on:
  push:
    branches: [main]
    paths-ignore: ['**.md', 'docs/**', '.vscode/**']
  workflow_dispatch:

jobs:
  build:
    permissions:
      contents: read
      packages: write
    uses: RobyRew/platform/.github/workflows/build-image.yml@main
    secrets:
      DOKPLOY_DEPLOY_WEBHOOK: ${{ secrets.DOKPLOY_DEPLOY_WEBHOOK }}
```

`permissions` belongs in the **caller**, not only in the reusable workflow. A called
workflow can only reduce the permissions it is invoked with, never escalate, and a
repo whose `default_workflow_permissions` is `read` will fail at startup without
this block.

The image is published to `ghcr.io/<owner>/<repo>` as `:latest` and `:<sha>`.
It is pushed with the workflow's `GITHUB_TOKEN`, so no registry PAT is stored in
the repo or on the VPS. For a **public** repo the resulting package is publicly
pullable and Dokploy needs no credentials; a **private** repo produces a private
package, and Dokploy has to be given a token with `read:packages`.

Set `DOKPLOY_DEPLOY_WEBHOOK` as a repo secret to the app's Dokploy deploy URL, and
switch that Dokploy app's source to the Docker provider pointing at the image.
