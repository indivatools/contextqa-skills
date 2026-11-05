---
name: cqa-verify-fix
description: Use when a bug that ContextQA reproduced has been fixed, merged and deployed, and the ticket should get a FIXED / NOT_FIXED verdict from the saved regression test. Moves the ticket into a re-validation status, waits for ContextQA's re-check, and falls back to re-running the regression test directly when the status change never arrives. Triggers on "verify the fix for CON-123", "is CON-123 fixed now", "re-validate this ticket", "/cqa-verify-fix".
---

# ContextQA Verify a Fix

A reproduction leaves a regression test behind. When the ticket moves into one
of the org's **re-validation statuses**, ContextQA re-runs that test and
comments the verdict on the ticket. This skill drives that, and reports it.

Verifying code that is not live proves nothing. Step 0 is not optional.

## Step 0 — Is the fix live?

The fix must be reachable at the environment the reproduction ran against:
merged and deployed, or the local app restarted on the fixed branch behind a
tunnel (`/cqa-tunnel`). If you cannot tell, ask.

## Step 1 — Read the settings

```
manage_bug_reproduction(action="get_settings")
```

Read `revalidation.enabled` and `revalidation.statuses`. Off → say so.
Turning it on is `update_settings` with a merged `changes` object, and only
after the user agrees (settings are org-wide).

## Step 2 — Note where things stand

`manage_bug_reproduction(action="list_by_ticket", ticket_id=...)` → keep the
newest row, its `status` and its time, and the regression `test_case_id`.
Nothing found → see the lookup note in `/cqa-bug-repro`.

## Step 3 — Move the ticket

Move it into one of `revalidation.statuses`.

- A Linear / Jira tool is available → move it. Being asked to verify is
  consent for this one status change, nothing more.
- No tracker tool → ask the user to move it and wait for "done".

## Step 4 — Wait for the re-check

Poll `list_by_ticket` about once a minute, up to 20 minutes, until the row
gains a re-validation verdict or a newer row finishes.

**Nothing started after about 10 minutes** → the status change most likely
never reached this ContextQA environment (the tracker's webhook points at a
different one, or the status is not in the list). Say so, and offer the direct
check:

```
verify_bug_fix(test_case_id=N)
```

It re-runs the regression test against the environment (credits) but posts
nothing on the ticket. Report it as **"regression test passed / failed"**, never
as the ticket's verdict.

## Step 5 — Report

| Verdict | Means | Next |
|---|---|---|
| `FIXED` | the regression test passed on the ticket's re-check | stop; leave the ticket status to the team |
| `NOT_FIXED` | still broken | show the failed step and evidence; offer `/cqa-debug` |
| `COULD_NOT_VERIFY` | the test never reached the check | show why — test data changed by hand, environment not serving the fix, login failed |

Include the time from the status move to the verdict.

## Rules

- Report only what the run says. Never mark a ticket Done unless asked.
- One status change per request. Do not walk a ticket through statuses to
  force a re-check.
- Never change the test data the regression test depends on while verifying.
