---
name: cqa-integrations
description: Use when connecting ContextQA to GitHub, GitLab, Slack, Linear or Jira, diagnosing a connection that says "connected" but does nothing, enrolling a repository for PR impact analysis, or configuring where run notifications land. Covers the OAuth handoff, the settings that actually gate the pipeline, and the fields the platform stores but does not enforce. Triggers on "connect github", "set up slack notifications", "why isn't the PR check running", "enrol this repo", "disconnect jira", "/cqa-integrations".
---

# ContextQA Integrations, Notifications and PR Impact

Three things live here: **connections** (a provider is linked to the tenant),
**enrolments** (a specific repository is watched), and **settings** (what the
pipeline is allowed to do). A failure at any layer looks identical from the
portal — "connected", and nothing happens — so always diagnose in order.

## Step 0 — Read the current state

```
get_current_workspace
manage_integrations(action="list")                 # every provider, whole tenant
manage_integrations(action="registry", provider="GITHUB")
manage_pr_impact(action="list_repos")
manage_notifications(action="get")
```

**Prefer `registry` over `list` when something is wrong.** An integration row
can outlive the provider-side install — someone uninstalls the GitHub App and
`list` still says connected. Only the registry sees that.

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

- `github`, `gitlab`, `slack`, `linear` and **Jira Cloud** all connect by
  one-click OAuth (Jira Cloud is Atlassian 3LO).
- **Jira Data Center** uses a site URL + username + API token form, as do the
  other token-based trackers (Azure DevOps, ClickUp, YouTrack, Bugzilla, Mantis,
  Zepel, Backlog, Freshrelease, Trello, MS Teams, Google Chat). `list` returns
  them even though they never appear in a connect link.

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

## Step 3 — Enrol a repository for PR impact

```
manage_pr_impact(action="register_repo", integration_id=N,
                 name="acme-web", repo_full_name="acme/acme-web",
                 default_branch="develop")
```

Registration always binds the current workspace — a repo with no workspace
blocks every analysis.

**Read `syncStatus` on each row.** It is the single most useful field and the
answer to "the settings page says connected but nothing happens":

| `syncStatus` | Meaning |
|---|---|
| `PENDING_SETUP` → `TESTS_SYNCED` → `FEATURES_CONFIGURED` | stages on the way to ready |
| `READY` | working |
| `FAILED` | setup did not finish |
| `ACCESS_REMOVED` (with `isActive: false`) | the provider grant is gone — the usual cause |

Diagnose in this order: the repo's `syncStatus`, then
`manage_integrations(action="registry")`, then the connection itself. The repo
row is the most specific of the three, and the only one that distinguishes "we
lost access to this repository" from "the whole installation is revoked".

## Step 4 — Settings, and the one that silently drops everything

```
manage_pr_impact(action="get_settings")
manage_pr_impact(action="update_settings", changes={"baseBranchFilter": "develop"})
```

Settings are an **org-wide singleton**, not per workspace — a change reaches
every workspace. `update_settings` takes only the keys you want changed and
merges them, because the underlying PUT is a full replace that would otherwise
blank the rest of the org's configuration.

**`baseBranchFilter` defaults to the literal string `main`.** In an organisation
that merges into `develop` or `qa`, that default silently drops every pull
request and nothing anywhere reports it. Set it to the repo's real default
branch — this is the first thing to check when a correctly-connected repo
analyses nothing.

`testRunTiming` decides whether anything actually runs:

| Value | Behaviour |
|---|---|
| `OFF` (default) | analyse and report, run nothing |
| `POST_MERGE` | run after merge, once `deploymentWaitMinutes` (0–720) elapses or a deployment webhook arrives early |
| `PRE_MERGE` | run against the PR's own build, matched on the head commit |

**Stored but not enforced by the platform:** `preventSelfApproval`,
`requiredApproversQuorum`, `archiveApproversQuorum`, `autoDemoteOnRejections`,
`pathScope`, `suppressPaths`, `maxFilesPerAnalysis`. They come back in the
object and read like working controls. Never describe them to a user as active —
`pathScope` in particular looks like a working filter and matches nothing.

One more surprise: setting `autonomyTier` to `AUTONOMOUS` **also forces
`autoPromoteOnMerge` true server-side**, so the object can come back changed in
a field you did not send. Re-read after writing.

## Step 5 — Run and review an analysis

```
manage_pr_impact(action="trigger", repo_id=R, pr_number=142)   # 202: dispatched, not done
manage_pr_impact(action="find_by_pr", repo_id=R, pr_number=142)
manage_pr_impact(action="get_analysis", analysis_id=A)
manage_pr_impact(action="list_items", analysis_id=A, item_class="UPDATE")
manage_pr_impact(action="set_item_status", analysis_id=A, item_id=I, status="APPROVED")
manage_pr_impact(action="run", analysis_id=A, environment_id=E)
```

- **A PR is identified by `(repo_id, pr_number)`.** The head sha plays no part:
  there is one analysis row per PR forever, and each new commit overwrites it in
  place.
- The PR number is the **entire** request body for `trigger` — title, author,
  base branch and SHAs are read server-side from the integration, so a PR cannot
  be analysed against metadata that is not its own.
- **Poll on `lifecycle`, never on `status`.**
- `RERUN` items imply **no test-case change** — branch on `analysisClass` before
  acting. `AUTO_APPROVED` is server-set and always rejected.
- `set_item_status` fails once the analysis is finalized.

### Four shapes that read as success and are not

1. **`preflight: null` means "not checked", not "clean."** A real report with
   `casesWithFindings == 0` is the earned all-clear.
2. **A `COMPLETED` run with 0 passed and 0 failed executed nothing.**
3. **`FAILED_TO_START` is infrastructure, not a red build.**
4. **`decisionCarry: null` means nothing carried**, not that everything survived.

And a naming trap: the run block is on the wire as `postMergeRun` **even for a
pre-merge run** — the class was renamed and the JSON key deliberately was not.
Read its `timing` field.

The preflight itself is a static advisory scan asking "would a run against this
environment actually exercise the PR's build?". Its findings are ordered by how
badly they mislead: **`FIXED_URL` — a literal address in a step — is worse than
`MISSING_ENV_KEY`**, because the case *passes*, having tested the old
deployment. That is the argument for `cqa-environments` in one finding.

## `analyze_impact` is a different tool

`analyze_impact(title, description, diff, source)` is an **ad-hoc semantic
query**: give it a change and it reasons over the case repository for 1–2
minutes, returning `MUST_UPDATE` / `MUST_RERUN` / `SHOULD_RERUN` per case. It
runs nothing, stores nothing, and needs no repository enrolled. It returns
individual **test cases** only — never suites or plans.

Use `analyze_impact` for "what would this change break?" and `manage_pr_impact`
for the actual pipeline. `/cqa-impact` is the workflow that wraps both.

## Always go through the portal

Connection data is reachable two ways, and only one of them is yours to use.
The portal exposes every integration read and write **scoped to your tenant
session**, and that is the surface the MCP, the browser UI and you should all
use. The underlying connection service is a server-to-server component with its
own credential, kept server-side deliberately.

So: if you find yourself reaching for a raw integrations-service URL or a shared
key, stop — the tenant-scoped portal route is the supported path, and the
`manage_integrations` / `manage_notifications` tools above are already on it.
