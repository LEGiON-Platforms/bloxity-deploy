# Bloxity Deploy action

Build a **Unity WebGL client** and a **Docker game backend**, and ship **both** to
Bloxity Legion in one step. The frontend and backend get the **same version**, so the
hosting dashboard always shows them matched.

It's opinionated: it assumes a Unity client and a server with a `Dockerfile` at a
known path. Everything is overridable, and you can skip either half.

## Quickstart

```yaml
# .github/workflows/deploy.yml
name: Deploy to Bloxity
on:
  push:
    branches: [dev, main]   # dev -> dev channel, main -> prod
  workflow_dispatch:

permissions:
  contents: read
  packages: write           # push the server image to ghcr

jobs:
  ship:
    runs-on: ubuntu-latest
    env:
      UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}   # Unity Personal .ulf contents
      # For Unity Pro instead: UNITY_SERIAL / UNITY_EMAIL / UNITY_PASSWORD
    steps:
      - uses: actions/checkout@v4
      - uses: LEGiON-Platforms/bloxity-deploy@v1
        with:
          deploy-token: ${{ secrets.LEGION_DEPLOY_TOKEN }}
          server-path: Server      # dir with your backend Dockerfile
          server-port: 2567        # port your backend listens on
```

> The action is public, but it grants no access on its own: every deploy requires a
> valid per-game token, which Bloxity issues only to whitelisted developers. Without
> a token the resolve / deploy / upload endpoints simply return 401.

The **deploy token is the only Bloxity input** — the game id, URLs and image name are
all resolved from it. On push it will:
1. build the Unity WebGL client, zip it, and upload it as your frontend,
2. build `Server/` into your repo's ghcr, and roll it on Legion.

Both halves get the same version, so the dashboard shows them matched.

## You provide two things

- **`LEGION_DEPLOY_TOKEN`** — your per-game token (re-viewable on hosting.bloxity.io). The only Bloxity secret.
- **A Unity license** — `UNITY_LICENSE` (Personal) or `UNITY_SERIAL`/`UNITY_EMAIL`/`UNITY_PASSWORD` (Pro), as env. See [game-ci activation](https://game.ci/docs/github/activation).
- The image is pushed to **your repo's own ghcr** with the built-in `GITHUB_TOKEN` (hence `packages: write`). One-time: make the `*-server` package **public** so Legion can pull it, or ask us to wire a private pull secret.

## Inputs

| Input | Default | Notes |
|---|---|---|
| `deploy-token` | (required) | `lgn_...` per-game token. **The only required input** — the game id is resolved from it. |
| `channel` | branch-derived | `prod` on `main`, else `dev`. Override to force one. |
| `version` | short SHA | Stamped on BOTH halves so they match. |
| `skip-client` | `false` | Deploy backend only. |
| `client-path` | `.` | Unity project root. |
| `unity-version` | `auto` | Read from `ProjectSettings/ProjectVersion.txt`. |
| `build-target` | `WebGL` | Unity build target. |
| `skip-server` | `false` | Deploy frontend only. |
| `server-path` | `Server` | Dir with the backend Dockerfile (build context). |
| `server-dockerfile` | `<server-path>/Dockerfile` | Override if elsewhere. |
| `server-port` | `2567` | Backend listen port (HTTP + WS). |
| `api-url` / `legion-url` | bloxity.io | Rarely changed. |

## Building the frontend yourself?

Set `skip-client: true` and upload your own build with the same token:

```bash
curl -fsS -X POST \
  "https://api.bloxity.io/v1/hosting/games/<game-id>/frontend?channel=dev&version=<sha>" \
  -H "Authorization: Bearer $LEGION_DEPLOY_TOKEN" \
  -H "Content-Type: application/zip" \
  --data-binary @build.zip
```

## Notes

- Frontend upload limit is 150 MB (the zipped WebGL output; Unity ships Brotli-compressed builds, so this fits most games — tell us if you need more).
- The action is public (token-gated — referencing it grants nothing without a valid per-game token). Canonical source is the Bloxity monorepo at `.github/actions/bloxity-deploy`; releases are published to this repo as `@v1`.
