# Orange Uptime Kuma — Fork Documentation

**Project:** Cloud S2 2526 — Orange Kuma
**Author:** Cas Emmens
**Upstream:** [louislam/uptime-kuma](https://github.com/louislam/uptime-kuma)

This repository is a **fork of Uptime Kuma**, customized to run as the
per-customer monitoring instance of the Orange Kuma hosting platform.
This document describes only the changes layered on top of upstream — the
core monitoring engine, UI, and feature set are unchanged. For the full
platform (Proxmox, k3s, GitOps provisioning), see the
[`project-cloud`](../project-cloud) repository.

---

## Table of Contents

1. [Role in the Platform](#1-role-in-the-platform)
2. [Branding Changes](#2-branding-changes)
3. [Customer Metadata](#3-customer-metadata)
4. [Hands-off Startup Behaviour](#4-hands-off-startup-behaviour)
5. [Environment Variables](#5-environment-variables)
6. [Container Image](#6-container-image)
7. [CI/CD Pipelines](#7-cicd-pipelines)
8. [Keeping in Sync with Upstream](#8-keeping-in-sync-with-upstream)

---

## 1. Role in the Platform

Each customer of the Orange Kuma platform gets their own isolated Uptime
Kuma instance, deployed into a dedicated `customer-<slug>` Kubernetes
namespace. The manifest is rendered and committed to Gitea by an Ansible
playbook (`provision-customer.yml` in `project-cloud`) and reconciled
into the cluster by Argo CD.

The whole flow is meant to be **hands-off** — a salesperson fills in a
form in Semaphore and a working, branded, pre-configured monitoring
instance appears. The customizations in this fork exist to make that
possible without anyone touching the instance after it boots.

---

## 2. Branding Changes

The application is rebranded from "Uptime Kuma" to **"Orange Uptime
Kuma"** with an orange colour scheme. Touched files:

| File | Change |
|------|--------|
| `index.html` | page `<title>` and meta description |
| `public/manifest.json` | PWA app name |
| `src/assets/vars.scss` | orange colour palette |
| `src/layouts/Layout.vue` | branding in the layout shell |
| `src/mixins/theme.js`, `src/util.ts` | theme colour wiring |

These are presentational only — no behavioural change to monitoring.

---

## 3. Customer Metadata

A customer instance is aware of who it belongs to. Four environment
variables carry customer identity into the container (see
[Section 5](#5-environment-variables)), and a small read-only API
endpoint surfaces them:

```
GET /api/customer-info
→ { "name": "...", "id": "...", "email": "...", "domain": "..." }
```

Defined in `server/routers/api-router.js`. The values come straight from
the `CUSTOMER_*` env vars (empty strings when unset). The UI
(`src/layouts/Layout.vue`) reads this to display which customer the
instance belongs to. The endpoint is read-only and exposes only what was
already injected at deploy time.

---

## 4. Hands-off Startup Behaviour

Two best-effort startup steps were added to `server/server.js`, called
from the server bootstrap right around `await server.start();`. Both are
wrapped in try/catch so a failure is logged and swallowed — they can
never block the server from coming up — and both are **idempotent** so
restarts never duplicate state.

### 4.1 Admin bootstrap — `autoCreateAdminUser()`

Stock Uptime Kuma only creates the admin user through the web setup
wizard. For a hands-off provisioning flow that won't do. On startup, if
`UPTIME_KUMA_ADMIN_PASSWORD` is set **and the user table is empty**, the
instance creates the admin account from env (username from
`UPTIME_KUMA_ADMIN_USER`, default `admin`), hashes the password the same
way the socket `setup` handler does, and flips the in-memory `needSetup`
flag off so clients get the login screen instead of the wizard.

Because it only runs when no user exists, a later manual password change
is never clobbered, and leaving the password env unset preserves the
normal web-setup flow.

### 4.2 Auto-monitor — `autoCreateCustomerMonitor()`

When `CUSTOMER_DOMAIN` is set, the instance ensures exactly one HTTP
monitor for `https://<CUSTOMER_DOMAIN>` exists, so the customer's
dashboard is useful out of the box. It:

1. Returns early if `CUSTOMER_DOMAIN` is empty.
2. Looks up the first user (a monitor needs an owning `user_id`); if no
   user exists yet, it logs and skips — retrying on the next boot once
   setup (or the admin bootstrap above) has run.
3. Checks for an existing monitor with that URL (idempotency key) and
   does nothing if one is present.
4. Otherwise creates an `http` GET monitor (60s interval, 1 retry,
   active), letting the remaining columns fall back to their schema
   defaults, and starts it on the live scheduler via `startMonitor()` so
   it begins checking immediately — no restart required.

With the admin bootstrap in place, a freshly provisioned instance with a
domain comes up already monitoring that domain, with a usable admin
login, and nobody had to click through any setup.

---

## 5. Environment Variables

These are declared (with empty defaults) in `docker/dockerfile` and set
at deploy time by the Kubernetes manifest:

| Variable | Purpose |
|----------|---------|
| `CUSTOMER_NAME` | Customer slug; surfaced via `/api/customer-info` and the UI. |
| `CUSTOMER_ID` | Optional customer identifier. |
| `CUSTOMER_EMAIL` | Contact email; surfaced via `/api/customer-info`. |
| `CUSTOMER_DOMAIN` | If set, an HTTPS monitor for it is auto-created on boot (§4.2). |
| `UPTIME_KUMA_ADMIN_PASSWORD` | If set, bootstraps the admin user on first boot (§4.1). |
| `UPTIME_KUMA_ADMIN_USER` | Admin username, default `admin`. |

`UPTIME_KUMA_PORT` (3001 in the platform manifest) and the standard
upstream Kuma env vars continue to work unchanged.

---

## 6. Container Image

Built from `docker/dockerfile` (multi-stage, based on the upstream
`louislam/uptime-kuma:base2` images). Orange Kuma additions:

- The `CUSTOMER_*` env declarations on the release stage.
- A `chown -R node:node /app` before `npm ci` in the build stage — the
  base image leaves `/app` owned by root, which made `npm ci` fail with
  `EACCES` once we switched to `USER node`.

The image listens on port **3001**, runs `node server/server.js` under
`dumb-init`, and keeps the upstream healthcheck. The platform mounts a
PersistentVolume at `/app/data` for the SQLite database.

---

## 7. CI/CD Pipelines

Two Drone pipelines are defined in `.drone.yml`.

### 7.1 PR check pipeline (on pull requests to `main`)

A DevSecOps gate with a shared `node_modules` cache volume. Steps:

| Step | What it does |
|------|--------------|
| `build` | `npm ci` — installs and warms the cache. |
| `test` | `npm ci && npm test`. |
| `sast-eslint` | Static analysis via ESLint (`--max-warnings=10`). |
| `npm-audit` | Dependency vulnerability audit (`--audit-level=high`). |
| `owasp-dependency-check` | OWASP Dependency-Check (HTML + JSON report, NVD API key from secret). |
| `secrets-scan` | greps for hardcoded `password=`/`secret=`/`api_key=`. |
| `sbom` | Generates a CycloneDX Software Bill of Materials. |
| `config-validation` | Validates `package.json` required fields + Node version. |
| `notify-discord` | Posts pass/fail to Discord (webhook from secret). |

The security steps are non-blocking (`|| true`) — they surface findings
in the logs and Discord without failing the build, suitable for a
coursework pipeline.

### 7.2 Docker build & push (on push to `main`)

`plugins/docker` builds `docker/dockerfile` and pushes to the Gitea
registry with two tags: `latest` and the 7-char commit SHA. Registry,
repo, and credentials come from Drone secrets (`GITEA_REGISTRY`,
`KUMA_IMAGE_REPO`, `GITEA_REGISTRY_USERNAME`, `GITEA_REGISTRY_PASSWORD`),
and the push is `insecure: true` (plain-HTTP internal registry). A
Discord notification reports the result.

The SHA tag is what Argo CD Image Updater keys on to roll new builds out
to every customer instance — see the auto-update strategy in
`project-cloud`.

---

## 8. Keeping in Sync with Upstream

All Orange Kuma changes are deliberately small and localized so the fork
can track upstream:

- New behaviour in `server/server.js` lives in two self-contained
  functions appended near the bottom, each prefixed with an
  "Orange Kuma customization" comment block.
- The `/api/customer-info` route is a single additive handler.
- Branding changes are confined to the files listed in §2.
- The Dockerfile only adds env declarations and one `chown`.

To pull upstream changes, rebase onto the upstream tag and re-apply these
isolated edits — there are no invasive changes to the monitoring core.
