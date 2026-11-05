---
name: cqa-bug-repro
description: Use when a bug ticket should become a running reproduction — either on demand from a pasted or fetched ticket, or automatically from Linear, Jira or Slack when a label, command or reaction fires. Covers the trigger rules and settings, the production-write approval gate, what gets posted back on the ticket, and the handoff into a fix. Triggers on "reproduce this bug", "turn this ticket into a test", "set up auto-reproduce", "why didn't the reproduction trigger", "/reproduce", "/cqa-bug-repro".
---

# ContextQA Bug Reproduction

A bug report becomes a real, executable test case that either reproduces the
bug or does not — and the answer is posted back on the ticket. That is the whole
product surface. There are two ways in:

| Path | Who starts it | Use when |
|---|---|---|
| **Automatic** | a label, comment command, reaction or agent mention on a Linear / Jira ticket, or `/reproduce` in Slack | the team wants every bug triaged without asking |
| **On demand** | you, via the MCP | you have a ticket in hand right now |

This skill owns **ticket → reproduction**. Once you have a failing run and want
to fix it, that is `/cqa-debug`.

## On demand — you have the ticket

```
bug_fix_from_ticket(ticket_text=<the body>, url=<app_url>,
                    repo_url=<optional>, deployment_info=<optional>)
```

Returns `test_case_id`, `session_rules[]`, `fix_guide[]` and a note. It creates
the reproduction case, executes it, optionally locates the responsible code, and
hands back a structured bundle.

- **Fetch the body first.** Matching MCP → `gh issue view` / `glab issue view` →
  `curl`. **Never pass a raw URL as `ticket_text`** — the tool reads the text,
  not the link. Plain pasted text is a first-class input.
- **`deployment_info` defaults to assuming local changes are immediately live**
  at `url` (hot reload, or a tunnel — see `/cqa-tunnel`). If a build or deploy
  step stands between the fix and the app, say so here, or a verification run
  will fail for a reason that has nothing to do with the fix.
- **Treat the returned `session_rules` as authoritative** — where they conflict
  with this skill, they win.
- `reproduce_from_ticket` is the lower-level half (create and run, no fix
  guide). Prefer `bug_fix_from_ticket` unless you only want the case.

Then: `verify_bug_fix(test_case_id=N)` after a fix,
`post_run_comment(...)` per attempt, `report_fix_to_ticket(...)` at the end.
**Never call `report_fix_to_ticket` without a real verification result**, and if
verification still fails, call `investigate_failure(result_id)` for the actual
evidence before concluding anything.

## On demand — the platform run that reports on the ticket

`bug_fix_from_ticket` runs inside your session. To start the same reproduction
the automatic triggers start — the one that comments on the ticket, keeps a
regression test and re-validates later — trigger it by ticket:

```
manage_bug_reproduction(action="trigger", ticket_id="CON-123", source="linear", confirm=True)
manage_bug_reproduction(action="list_by_ticket", ticket_id="CON-123")   # poll
```

- **One run costs credits and posts on the ticket.** Being asked to reproduce
  this ticket is consent for one run; ask before another.
- **Check first.** If `list_by_ticket` already has a finished row and nobody
  asked for a fresh run, show it. A row still `IN_PROGRESS` → wait for it,
  never start a second.
- **`trigger` returns no id.** Poll `list_by_ticket` about once a minute; an
  empty list for the first minutes is queued, not failed. Runs take roughly
  3–12 minutes. Stop at 20 and report what you saw.
- **Empty for a ticket that did run?** Rows are matched on the ticket reference
  exactly. Older runs may be stored under the tracker's internal issue id (the
  Linear issue UUID, the Jira numeric id) instead of the key. If you can read
  the ticket with a tracker tool, retry with that id before concluding there is
  no run.
- **Report the verdict as written** — `REPRODUCED`, `NOT_REPRODUCED`,
  `NEEDS_INFO`, `INTERNAL_ERROR` — with the failed step, expected vs actual in
  the run's own words, the evidence link and the regression `test_case_id`.
- **`NEEDS_INFO`** → `action="submit_info"`, `analysis_id` = the row's
  `uploadRequirementId`, `details` = the missing facts **from the user**, never
  invented. It comments on the ticket and runs again (credits).

## Automatic — how a ticket fires a run

```
manage_pr_impact(action="reproduction_settings")
```

Org-wide, and the shape is worth reading before you debug a trigger:

```jsonc
{
  "triggerMode": "LABELLED",                 // or "ALL_ISSUES"
  "triggerLabels": ["auto-reproduce"],
  "triggerEvents": { "issueCreated": true, "labelAdded": true,
                     "contentEdited": true, "commentCommand": true, "reaction": true },
  "defaultEnvironmentId": 712,
  "defaultDataProfileId": null,
  "envLabelPrefixes": ["env:", "environment:"],
  "dataProfileLabelPrefixes": ["dataprofile:"],
  "maxRunsPerTicket": null,                  // null = no cap
  "maxRunsPerDay": null
}
```

### The triggers

| Trigger | Default |
|---|---|
| A trigger **label** added to an issue | `auto-reproduce` |
| A **comment command** | `/reproduce` to start, `/stop` to stop |
| A **reaction** emoji | 🔁 to start, 🛑 to stop |
| **@mentioning or assigning** the ContextQA agent in Linear | always |
| **`/reproduce`** as a Slack slash command, or @mentioning the app in a thread | always |

`triggerMode` decides whether labels gate anything at all: `LABELLED` (the
default) requires one of `triggerLabels`; `ALL_ISSUES` ignores labels and runs
on every issue that gets past the event switches.

### Why a reproduction did not fire

Work down this list — the first two account for most of it:

1. **`triggerMode: "LABELLED"` with an empty `triggerLabels` means nothing ever
   triggers.** It is a valid configuration that silently does nothing.
2. **The team's label is not the configured one.** A workspace that labels its
   bugs `Bug` and never changed the default from `auto-reproduce` gets silence.
   Check the actual labels against `triggerLabels`, not against what feels
   obvious.
3. **An event switch is off.** `contentEdited` is on by default so a reporter
   filling in missing detail re-runs the reproduction — and it is also the
   switch people turn off when every description tweak causes a run. Turning it
   off does not stop label-added triggering.
4. **The connection is not `active`.** `needs_reauth` or `revoked` on the
   provider row means no webhook is being acted on. See `/cqa-integrations`.
5. **A run cap.** `maxRunsPerTicket` / `maxRunsPerDay`, when set.

### Choosing the environment per ticket

The run uses `defaultEnvironmentId` unless the ticket carries a label with one
of `envLabelPrefixes` — `env:staging` or `environment:staging`. The same applies
to test data via `dataprofile:<name>`. That is how one team reproduces against
staging and another against a preview, without two configurations.

The environment must actually define the variables the reproduction's steps use
— see `/cqa-environments`. A reproduction that fails on step 1 with a missing
`*\|var\|` is a configuration problem, not a reproduced bug.

## The production-write approval gate

**A reproduction that needs to write to production stops and asks.** It posts a
comment asking the reporter to reply with the approval word — `APPROVE` by
default — and reads that grant back from the ticket's comments on the **next**
run.

Three consequences:

- **The grant is matched case-sensitively on a word boundary.** "approved the
  design" grants nothing, deliberately.
- **Replying `APPROVE` does not itself resume the run.** Something must start
  the next one — the reply, a re-label, or an explicit `/reproduce`. If a
  reporter says "I approved it and nothing happened", this is almost always why.
- Treat the gate as the safety feature it is. Do not work around it by
  re-pointing the reproduction at production with a different environment.

## What lands on the ticket

The agent posts progress and a verdict back as comments, with a root-cause
analysis when it reaches one. Two things to know:

- **Its own comments carry a hidden marker** so they cannot re-trigger a
  reproduction. Do not strip or reformat them when you copy them somewhere else,
  and do not hand-write a comment that imitates one.
- The verdict is the executor's own — read it, don't infer it. A reproduction
  that fails to reproduce is a real and useful answer, not a broken run.

## Configuring it

Settings are **org-wide**, and the write is a full replace, so change them by
merging rather than sending a partial object. Sensible first configuration:

1. Set `triggerLabels` to the label the team *actually* uses.
2. Leave `triggerMode: "LABELLED"` — `ALL_ISSUES` on a busy tracker spends
   credits on every issue, including ones that are not bugs.
3. Set `defaultEnvironmentId` to a **non-production** environment, and confirm
   it defines the variables the app's login needs.
4. Set `maxRunsPerDay` before turning it on for a large workspace, not after.

## Where this sits

| | |
|---|---|
| Connect Linear / Jira / Slack so triggers arrive | `/cqa-integrations` |
| Fix the bug once a reproduction is red | `/cqa-debug` |
| Environments and the variables a reproduction needs | `/cqa-environments` |
| What a change is likely to break, before it ships | `/cqa-impact` |
