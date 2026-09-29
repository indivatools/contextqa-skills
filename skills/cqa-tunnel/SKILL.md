---
name: cqa-tunnel
description: Use when the app under test is not reachable from the internet — it runs on localhost, a build agent, a VM or inside a corporate network — and ContextQA's runners need to reach it. Publishes a local service as a public HTTPS URL with the ContextQA tunnel agent, wires that URL into a ContextQA environment as the base URL, and optionally hosts a browser on the machine so runs execute inside the network. Triggers on "test my local app", "run tests against localhost", "expose port 3000 to contextqa", "run the browser on my machine", "set up the contextqa tunnel", "contextqa login", "add-route", "/cqa-tunnel".
---

# ContextQA Tunnel — testing what is only running locally

ContextQA's runners live in the cloud. Your app often does not. The tunnel
closes that gap: a small agent on the machine **dials out** and holds a
WireGuard tunnel from your side, so `http://localhost:3000` becomes a real
HTTPS URL the runners can reach. **Nothing inbound is opened.**

The same agent can host a **browser** on that machine, so the run itself
executes inside the network rather than reaching into it.

Two surfaces, and it matters which one you are on:

| Surface | Who drives it | What it is |
|---|---|---|
| `contextqa` CLI | **you, on the machine** | install, enrol, publish routes, diagnose |
| `/api/tunnel/*` HTTP API | the portal's Tunnels page | mint tokens, list machines and routes, remove them |

**There are no MCP tools for the tunnel.** The CLI is your surface. Treat the
HTTP API as read-mostly context and as the thing the portal page is doing.

## Step 0 — Decide whether you need it

You need a tunnel when the target is `localhost`, a private hostname, a VM, a
dev container or an app behind a corporate network. You do **not** need one for
anything already on the public internet — put that URL straight into an
environment and skip this skill.

## Step 1 — Get an install token (the user's step)

Send the user to the portal: **Tunnels → New tunnel**. They choose a machine
name and get back two things to copy: a **command** and a **token**.

If you are driving the API directly with a portal user session (cookie, or a
portal JWT as a bearer):

```
POST /api/tunnel/enroll-tokens     # on the portal origin, e.g. https://server.contextqa.com
Headers: X-C: 1                    # CSRF — presence only, value unchecked; omit it and you get 403
Body:    { "siteName": "build-agent-01", "platform": "linux", "enableBrowser": false }
```

- `platform` is `linux` | `macos` | `windows`.
- **`siteName` must match `^[a-z]([a-z0-9]|-[a-z0-9])*$` and be at most 21
  characters.** The cap is low because the name becomes part of every hostname
  this machine publishes. Validate before sending; 22 characters comes back as a
  `validation` error.
- The response carries `command`, `token` and `expiresAt` (default ~15 minutes).

**A portal API token (`token_…`) will not work here.** Those are limited to
`POST /api/tunnel/agent/enroll` — the machine's own first contact. Every
tenant-facing route needs a portal *user* session. So if you do not have one,
the honest answer is "open the Tunnels page", not a workaround.

**The command never contains the token, deliberately** — a command line ends up
in shell history, CI logs and screenshots. They are two separate things to copy.

## Step 2 — Install and enrol (ask first: this is a root install)

Installing runs a downloaded script **as root** and registers a background
service. Get explicit permission before running it, and say what it does.

```sh
# 1. install — identical for every machine and tenant
curl -fsSL https://get.contextqa.com/agent -o contextqa-install.sh
sudo sh contextqa-install.sh

# 2. enrol this machine
sudo contextqa login --site-name build-agent-01
```

`login` prompts for the token on the machine's own keyboard, which is why the
token does not travel through a command line. Windows enrols during install
instead (elevated PowerShell), and has no separate login step.

Two steps on purpose: the gap between them is where a cautious user reads the
script before running it as root. **Do not join them with `&&`.** Run them from
a directory only that user can write to — in `/tmp` another user could replace
the file between the two lines.

For CI, where nothing can be typed, pass the token through the environment
rather than a flag — `--token` leaves it visible in `ps` for the life of the
command:

```sh
sudo CONTEXTQA_TOKEN="$TOKEN" contextqa login --site-name ci-01
```

### Verify the install, and take exit 4 seriously

A successful install prints `verified … (minisign + sha256)` before installing
anything. If you do not see that line, nothing was installed.

| Exit | Meaning |
|---|---|
| 1 | bad usage — unknown flag, or an invalid site name / channel / version |
| 2 | refused before downloading — no token and no terminal, or a name conflict |
| 3 | installed but not enrolled — token rejected; mint a fresh one |
| **4** | **verification failed — signature or checksum mismatch. Nothing installed. Do not retry; escalate.** |
| 5 | the installer failed; the previous version is left in place |
| 6 | could not reach `get.contextqa.com`, or that version does not exist |

Exit 4 means what arrived on the wire was not what was published. Stop.

## Step 3 — Publish the app

```sh
sudo contextqa add-route web http://localhost:3000
contextqa routes
contextqa status
```

`add-route` publishes `https://web-<org>.<baseDomain>` and then polls until the
URL answers. Read the output honestly:

- `✓ Reachable (200)` — done.
- `! Published, but the URL isn't answering yet` — the route exists; the app is
  not up on that port. Start it, then `contextqa doctor`.
- `· still checking` — the probe worker has not reached a verdict inside the
  deadline. Not a failure.

Reachability is `pending` | `reachable` | `unreachable` | `unknown`, filled in
by a probe worker on a short interval. A brand-new route sitting at `pending` is
"checking", not broken.

**Do not build the URL yourself.** Read `urlShape` from `GET /api/tunnel/org`
(or the CLI output) — the `<route>-<slug>.<domain>` flattening is a hub-side
decision and has already changed once.

Other route commands: `contextqa remove-route <name>`, and the portal can delete
a route (`DELETE /api/tunnel/routes/{id}`) but **cannot add one** — creating
routes is the machine's job.

## Step 4 — Wire the URL into ContextQA (this is the point)

The tunnel is only useful once a test uses it. Put the URL in an **environment
variable**, never in a step:

```
manage_environment(action="create", name="Local — build-agent-01", parameters={
  "base_url": {"type": "text", "value": "https://web-acme.tunl.example.com"}
})
```

Then bind that environment on the case or the plan, and write steps against
`*|base_url|`. The same cases now run against local, staging and production by
changing one binding. See `cqa-environments`.

This is also why a literal URL in a step is so costly: swap the deployment and
the step still points at the old one, and the test passes.

## Step 5 — Optional: run the browser inside the network

Some apps can never be reached from outside — SSO bound to the corporate IdP,
an allowlisted internal API, a VPN-only backend. For those, host the browser on
the machine instead:

```sh
sudo contextqa browser install chromium      # ~260 MB
contextqa browser list                       # what is installed vs detected
sudo contextqa browser-host enable
contextqa browser-host status
```

What to know before promising it:

- **The browser host publishes a *private* route**, reachable only from
  ContextQA's egress addresses, with a deny-all catch-all behind the allowlist.
  **It refuses to publish anything if that allowlist is missing or empty** —
  an empty allowlist would mean an unauthenticated browser on the public
  internet. That refusal is the feature working.
- **Playwright-managed browsers** (chromium, firefox, webkit) exist only
  because Playwright downloaded them; the install default is **chromium only**.
  Telling a user to install Firefox themselves does not work — Playwright drives
  its own build. `contextqa browser install firefox|webkit`, elevated, is the
  way.
- **System channels** (chrome, msedge) are resolved from an existing vendor
  install and are never downloaded or removed. There, "install it yourself" is
  the correct answer.
- Downloads use `cdn.playwright.dev`, `registry.npmjs.org` and `nodejs.org` — a
  network already allowlisted for chromium needs no further change for the
  others.
- On Windows, `ENABLEBROWSER=1` on the MSI **does not enable the browser host**.
  It writes a flag and nothing else: no service started, nothing downloaded,
  nothing published. The two commands above are still required. Do not tell a
  customer the property sets it up.
- `browserHostAllowed` on `GET /api/tunnel/org` says whether the tenant may use
  it at all.

## Step 6 — When it does not work

```sh
contextqa doctor     # six checks: Agent · Service · Tunnel · Routes · Browser · App
contextqa status
contextqa logs 200
contextqa show-config   # secrets redacted
```

`doctor` is the first call, always — it separates "the agent is not running"
from "the tunnel is down" from "your app is not listening on that port", and
those have completely different fixes.

From the portal side: `GET /api/tunnel/routes/health` gives `{routeId, fqdn,
siteName, state, lastCheckCode, message}` per route, and `GET
/api/tunnel/sites` shows each machine's `state` — `enrolled` (registered, never
heartbeated), `up` (heartbeating) or `revoked`. Switch on the value and keep a
default branch: that set is explicitly expected to grow.

**Revocation stops management, not traffic.** A revoked machine's routes keep
serving. If the intent is "turn it off", delete the routes or the site — say
this plainly when confirming, because "revoke" reads like "turn off" and is not.

## Errors — branch on `code`, never on `message`

Every failure has the same body: `{ "code": …, "message": …, "detail": {…} }`.
`code` is contract; `message` is human text and gets reworded.

| code | What it means for you |
|---|---|
| `tunnel_disabled` (503) | the feature is off on this server — render a clean "not available", not an error |
| `forbidden` (403) | `detail.csrf: true` means you omitted the `X-C` header |
| `org_not_ready` (409) | still provisioning — **poll, do not error** |
| `slug_locked` (409) | too late to rename; the first machine has enrolled |
| `name_in_use` (409) | `detail.reason: "pending"` means a waiting enrollment holds it; `detail.enrollmentId` is the one to cancel |
| `token_used` (409) | install token already consumed — mint a fresh one |
| `quota_exceeded` (403) | plan limit — see quotas below |
| `lock_busy` (503), `rate_limited` (429), `hub_unavailable` (502) | the three worth an automatic retry. `hub_unavailable` guarantees nothing was created, so retrying is safe |

## Quotas — read them, never hardcode them

`GET /api/tunnel/org` returns `siteCount`/`siteQuota` and
`routeCount`/`routeQuota` on every call, so you can stop before the user hits
`quota_exceeded`. The documented tier defaults (free: 1 site / 3 routes;
verified: 5 / 15) are **already not what every environment serves** — quota is
per-environment configuration, not a property of the tier name. A page or a
script that hardcodes "free means 1" tells someone they are at their limit with
four machines left.

## Limits that are real today

- **Six endpoints are in the contract with full schemas and return `501`:**
  `PATCH /alerts`, `POST /alerts/test`, `POST /agent/diagnostics`, and the three
  `internal/**` ops routes. A generated client will happily offer them. **There
  is no alerting surface yet** — do not promise one.
- **There is no live docs endpoint.** `/api/tunnel/docs` looks like one and
  404s. The published OpenAPI 3.1 contract is the only source — ask ContextQA
  for it rather than probing paths, and generate any client from it.
- `/api/tunnel/health` and `/api/tunnel/readiness` answer `200` regardless of
  whether the feature is enabled — **pod health is not a signal that tunnels are
  live.** Call `GET /api/tunnel/org` and read the code.
- Several contract fields (`status`, `tier`, `state`, `reachability`,
  `visibility`) are typed as plain `string`, so a generated client will not
  narrow them. Keep a default branch in every switch.
- **The binaries are not yet code-signed** for Windows or Apple, so SmartScreen
  and Gatekeeper warn on first run. The minisign signature check is what
  actually protects the download — say so rather than telling someone to ignore
  a warning.

## Cleaning up

```sh
sudo contextqa uninstall     # stops the tunnel, removes the install, deletes this machine's routes
```

Public URLs stop resolving immediately. From the portal: `DELETE
/api/tunnel/sites/{id}` removes a machine and all its routes; `DELETE
/api/tunnel/enrollments/{id}` cancels a pending install and frees the name.

Upgrading is just re-running the install command — site and credentials are
kept, and once a machine is enrolled an upgrade needs no token and no
`--site-name`.
