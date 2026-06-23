# Running Mattermost Locally + Playwright E2E

A practical, verified guide to running the Mattermost server from source and driving the
Playwright E2E suite against it — full run or smoke-only.

> All commands below were verified on macOS (Apple Silicon) with the server + tests run from
> source. The repo root is `mattermost/` (this file lives there). Paths are relative to it.

---

## 1. Prerequisites

| Tool   | Version           | Check                |
|--------|-------------------|----------------------|
| Go     | ≥ 1.26.3 (`go.mod`) | `go version`        |
| Node   | ^24               | `node --version`     |
| npm    | ^11               | `npm --version`      |
| Docker | running daemon    | `docker info`        |

Install Go if missing: `brew install go`.

---

## 2. Start the server + webapp (from source)

From the `server/` directory:

```bash
cd server

RUN_SERVER_IN_BACKGROUND=true \
ENABLED_DOCKER_SERVICES="postgres" \
MM_TEAMSETTINGS_ENABLEOPENSERVER=true \
MM_PLUGINSETTINGS_ENABLE=true \
MM_PLUGINSETTINGS_ENABLEUPLOADS=true \
make run
```

What `make run` does:
- starts the Docker dependencies (here limited to **Postgres** — see note below),
- builds + runs the Go server on **http://localhost:8065**,
- builds the React webapp (webpack) into `webapp/channels/dist`, symlinked at `server/client`,
- keeps a webpack watcher running for hot rebuilds.

> **First run is slow** (~several minutes): Go modules download/compile and the webapp does a
> full webpack build. Subsequent runs are fast (caches warm).

### Why these env vars?

- `ENABLED_DOCKER_SERVICES="postgres"` — the default pulls **7** service images (redis,
  prometheus, grafana, loki, otel, inbucket, postgres) and Docker Hub pulls can flakily
  TLS-timeout. The E2E smoke tests only need Postgres. (Add `inbucket` if you need email tests.)
- `MM_PLUGINSETTINGS_ENABLE=true` + `MM_PLUGINSETTINGS_ENABLEUPLOADS=true` — **required** for
  the Playwright suite. See [Gotchas](#gotchas).
- `MM_TEAMSETTINGS_ENABLEOPENSERVER=true` — lets the test harness create teams/users freely.
- `RUN_SERVER_IN_BACKGROUND=true` — server runs detached; the webpack watcher stays in the
  foreground of the `make` process.

### Wait until it's ready

```bash
# API up:
curl -fsS http://localhost:8065/api/v4/system/ping        # -> {"status":"OK", ...}

# Webapp bundle served (only after the first webpack build finishes):
curl -fsS -o /dev/null -w "%{http_code}\n" http://localhost:8065/   # -> 200
```

---

## 3. Install the Playwright dependencies (once)

```bash
cd e2e-tests/playwright
npm install                 # also builds @mattermost/playwright-lib + downloads browsers
npx playwright install chromium   # ensure the Chromium binary is present
```

---

## 4. Run the tests

All commands run from `e2e-tests/playwright`. Point the suite at the local server with
`PW_BASE_URL`.

### a) Smoke test (fastest signal)

A single representative `@smoke` spec — confirms the whole stack works end to end:

```bash
PW_BASE_URL=http://localhost:8065 PW_HEADLESS=true \
  npx playwright test specs/functional/channels/search/find_channels.spec.ts --project=chrome
```

All `@smoke`-tagged specs (Chrome, with a retry):

```bash
PW_BASE_URL=http://localhost:8065 \
  npm run test:smoke
```

### b) Full suite

CI-style run (Chrome, excludes `@visual` snapshot tests):

```bash
PW_BASE_URL=http://localhost:8065 \
  npm run test:ci
```

Everything (Chrome + Firefox + iPad, including visual snapshots — needs the Playwright Docker
image for stable snapshots, so prefer `test:ci` locally):

```bash
PW_BASE_URL=http://localhost:8065 \
  npm run test
```

### c) Useful variants

```bash
# A single spec / pattern
PW_BASE_URL=http://localhost:8065 npx playwright test <path-or-pattern> --project=chrome

# Watch it run (headed) or slow it down
PW_BASE_URL=http://localhost:8065 PW_HEADLESS=false PW_SLOWMO=500 npx playwright test <spec>

# Interactive UI mode
PW_BASE_URL=http://localhost:8065 npm run playwright-ui

# Step-through debug
PW_BASE_URL=http://localhost:8065 npm run test:e2e:debug   # (if defined) or: --debug
```

### View the report

```bash
npx playwright show-report
```

Artifacts (screenshots / video / trace on failure) land in
`e2e-tests/playwright/results/output/`.

---

## 5. Stop everything

```bash
cd server
make stop          # stops server + webapp watcher + Docker containers
```

Postgres data persists in a Docker volume across restarts. To wipe it: `make nuke`.

---

## Key environment variables (Playwright)

| Var                 | Default                          | Purpose                                  |
|---------------------|----------------------------------|------------------------------------------|
| `PW_BASE_URL`       | `http://localhost:8065`          | Server URL under test                    |
| `PW_HEADLESS`       | headless                         | `false` to watch the browser             |
| `PW_SLOWMO`         | `0`                              | ms delay per action (debugging)          |
| `PW_WORKERS`        | `1`                              | Parallel workers                         |
| `PW_ADMIN_USERNAME` | `sysadmin`                       | Admin account (auto-created on first run)|
| `PW_ADMIN_PASSWORD` | `Sys@dmin-sample1`               | Admin password                           |
| `PW_ADMIN_EMAIL`    | `sysadmin@sample.mattermost.com` | Admin email                              |

Full list: `e2e-tests/playwright/sample.env`.

The `setup` project (`specs/test_setup.ts`) runs first automatically and `baseGlobalSetup`
(`lib/src/global_setup.ts`) **creates the `sysadmin` account as the first user** — no license
needed for functional/smoke specs.

---

## Gotchas

### `Changing PluginSettings.EnableUploads is not allowed due to security reasons`

The test harness's `pw.initSetup()` pushes a config with `PluginSettings.EnableUploads: true`
via the API. The server (`server/channels/api4/config.go`) **refuses to change `EnableUploads`
through the API** when the new value differs from the current one. Fix: start the server with
`MM_PLUGINSETTINGS_ENABLEUPLOADS=true` (as in §2) so the value already matches → no change →
allowed. (CI does the equivalent via `mmctl --local config set` in
`e2e-tests/.ci/server.prepare.sh`.)

### Docker Hub TLS handshake timeouts while pulling images

Transient. Re-running the pull usually succeeds. Limiting `ENABLED_DOCKER_SERVICES="postgres"`
minimizes the number of images pulled.

### Webapp returns HTTP 500 at `/`

The first webpack build hasn't produced `server/client/root.html` yet. Wait for the webpack
"compiled successfully" line (watch the `make run` output) and retry.

---

## All-in-one Docker alternative (no local Go/Node)

If you'd rather not build from source, the CI orchestration runs a prebuilt server image +
Playwright entirely in Docker (defaults to `@smoke`):

```bash
cd e2e-tests
TEST=playwright make          # generates compose, starts server, runs Playwright smoke
make stop                     # tear down
```

Pin the server image via `e2e-tests/.ci/env` (e.g.
`SERVER_IMAGE="mattermostdevelopment/mattermost-enterprise-edition:master"`) since the default
tag tracks the current commit hash. This path pulls several large images on first run.
```
