---
name: cqa-integrations
description: Use when connecting ContextQA to GitHub, GitLab, Slack, Linear or Jira, diagnosing a connection that says "connected" but does nothing, or choosing where Slack notifications land. Covers the one-click handoff, how to tell a one-click row from a legacy token row, what `connection_status` means, and why disconnecting takes enrolled repositories with it. Triggers on "connect github", "set up slack notifications", "why isn't my integration working", "reconnect gitlab", "disconnect jira", "/cqa-integrations".
---

# ContextQA Integrations and Notifications

This skill owns **connections** — linking a provider to the tenant, telling
whether that link actually works, and taking it away — plus the Slack channel
that notifications go to.

It does **not** own the PR impact pipeline. Enrolling a repository, the
analysis settings, triggering a run and reviewing the items all live in
**`/cqa-impact`**.

## Step 0 — Read the current state

```
get_current_workspace
manage_integrations(action="list")                 # every provider, whole tenant
manage_integrations(action="registry", provider="GITHUB")
manage_pr_impact(action="list_repos")
manage_notifications(action="get")
```

### Read `connection_status` first

Every one-click row carries `metadata.connection_status`, and it is exactly
three values. This is one call and it usually *is* the answer:

| `connection_status` | What it means |
|---|---|
| `active` | working |
| `needs_reauth` | the stored credential no longer reaches the provider — the user must reconnect |
| `revoked` | the grant is gone provider-side (app uninstalled, token revoked) |

`registry` is the cross-check: it returns the global mapping with its own
`status` (e.g. `REVOKED`) and the `externalConnectionId`, which is what proves a
row is wired to *this* org rather than merely present. A connection that reads
`active` in `list` but has no registry row is wired to nothing.

### Telling a one-click row from a legacy one

`list` returns every integration the tenant has, of both kinds, and they are
easy to confuse:

- **One-click (OAuth / App):** named `GitHub App`, `GitLab App`, `Slack App`,
  `Linear App`; `username` is `oauth-app`; `token` is null; the real state is in
  `metadata` (`connection_status`, `installation_id`, `account_login`, `teamId`).
- **Legacy token rows:** a real `username` and a stored, masked `password` /
  `token`; `metadata` is usually `null`. Azure DevOps, Jira Data Center, the
  vault entries and the older trackers all look like this.

Only the first kind has a connect link, a registry row, or a
`connection_status`.

## Step 1 — Connect a provider

```
manage_integrations(action="connect_url", provider="github")
```

This returns the **portal's integrations page**, not a raw OAuth link. That is
deliberate: the portal mints a signed, single-use connect ticket at the moment
the user clicks Connect, so the link never expires and cannot be replayed
against another org. Hand the link over and tell the user which workspace to
have selected.

**You cannot complete OAuth headlessly.** The provider shows a consent screen,
and the handshake is consumed exactly once. Building the link and handing it
over is the whole of what an agent can do here.

On **Ship** (`ship.contextqa.com`, the self-serve product) these are part of
onboarding, and the source-control connection is limited to **exactly one
repository, read-only**. Plan accordingly — there is no second repo to enrol.

- `github`, `gitlab`, `slack`, `linear` and **Jira Cloud** are the five
  one-click providers — those are exactly the ids `connect_url` accepts, and
  exactly the five registered in the connection service.
- **GitHub is a GitHub *App install*, not an OAuth authorize.** The consent
  screen is an app installation, and API calls use short-lived installation
  tokens rather than a stored one — which is why an uninstall on GitHub's side
  shows up as `revoked` rather than as an expired token. GitLab and Slack are
  OAuth 2 (GitLab with PKCE), Linear is plain authorization-code, and Jira Cloud
  is Atlassian 3LO.
- **Jira Data Center keeps the site URL + username + API token form,
  permanently** — it has no 3LO, and that is a decision on record, not a gap.
  The same form covers the other token-based trackers (Azure DevOps, ClickUp,
  YouTrack, Bugzilla, Mantis, Zepel, Backlog, Freshrelease, Trello, MS Teams,
  Google Chat). `list` returns all of them even though they never appear in a
  connect link.
- **Expect to find existing Jira rows in the legacy form.** Jira Cloud only
  became one-click recently, so a tenant connected before then still carries a
  Basic-auth row with a real username and a stored token. Read the row before
  assuming which kind you are looking at, and reconnect through `connect_url` if
  you want it on 3LO.

**A connection binds to one workspace version at connect time.** `connect_url`
always sends the current workspace, because omitting it makes the service bind
the tenant's *oldest* version — almost never the one the user is looking at.
Reconnecting from the right workspace heals a wrongly-bound connection.

### Disconnecting is worse than it looks

`disconnect` requires `confirm=True` because it **cascade-deletes every enrolled
repository** under the integration, with no warning and no 409. Every PR those
repos were enrolled in stops being analysed, and reconnecting does not bring the
enrolments back. Read `list` and `repositories` and tell the user exactly what
will be lost before you pass the flag.

## Step 2 — Slack notifications

```
manage_notifications(action="channels")    # read live from Slack; check `truncated`
manage_notifications(action="set", channel_id="C0BS12345", channel_name="qa-alerts")
```

- `channel_id` is **Slack's own id**, never `#name`.
- For a private channel the bot must already be a member, or Slack answers
  `private_channel_invite_required`.
- Requires a **one-click** Slack connection. A legacy pasted-token Slack row
  reports `connected: false` here even though it shows up in
  `manage_integrations(action="list")`.
- The choice is stored **per workspace, not per workspace version** — two
  versions share one channel.
- `clear` mutes the workspace rather than deleting the setting, so it does not
  fall back to an org default.
- The **run report by email** is a separate, simpler mechanism: `notify_emails`
  on a test plan. See `cqa-suites-and-plans`.
## When a connection is live but nothing happens

Three layers can each break while the one above looks fine. Diagnose in this
order — most specific first:

1. **The enrolled repository row** — `manage_pr_impact(action="list_repos")`,
   read `syncStatus`. `ACCESS_REMOVED` (with `isActive: false`) is the usual
   cause and the only signal that distinguishes "we lost access to *this*
   repository" from "the whole installation is revoked". `PENDING_SETUP` →
   `TESTS_SYNCED` → `FEATURES_CONFIGURED` are stages on the way to `READY`;
   `FAILED` means setup did not finish.
2. **The registry** — `manage_integrations(action="registry", provider=...)`.
3. **The connection row** — `metadata.connection_status`, above.

Everything past the repository row — enrolment, settings, triggering and
reviewing an analysis — belongs to the impact pipeline. **See `/cqa-impact`**,
which owns it end to end, including the `baseBranchFilter` default that
silently drops every pull request in an organisation that does not merge into
`main`.

## Always go through the portal

Connection data is reachable two ways, and only one of them is yours to use.
The portal exposes every integration read and write **scoped to your tenant
session**, and that is the surface the MCP, the browser UI and you should all
use. The underlying connection service is a server-to-server component with its
own credential, kept server-side deliberately.

So: if you find yourself reaching for a raw integrations-service URL or a shared
key, stop — the tenant-scoped portal route is the supported path, and the
`manage_integrations` / `manage_notifications` tools above are already on it.
