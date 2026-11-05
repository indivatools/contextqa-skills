---
name: cqa-ship-bug
description: Use when one bug ticket should go all the way from report to verified fix — reproduce it with ContextQA, fix the code, verify the fix on the ticket (before the PR when the tested app runs from this working copy, otherwise after the merge is deployed), open a pull request, run the tests the PR affects (fixing failures, at most 3 rounds), and stop for a person to merge. Chains /cqa-bug-repro, /cqa-verify-fix, /cqa-impact and /cqa-debug. Triggers on "ship CON-123", "take CON-123 end to end", "reproduce, fix and verify CON-123", "/cqa-ship-bug".
---

# ContextQA Ship a Bug

One ticket, start to finish, without stopping until a person has to merge. The
agent writes the code; ContextQA tests it on a real browser; a person merges.
After every stage, post a one-line status with how long the stage took (use
ContextQA's timestamps where they exist).

## Step 0 — Ready?

If `/cqa-init` has not passed in this session, run it. Also confirm:

- a tracker is connected (`/cqa-integrations`), and the reproduction's
  environment serves the app this repo builds;
- `gh` (or `glab`) is signed in, so a PR can be opened.

Stop on anything missing and say the one action that fixes it.

Then decide **where the fix becomes live**:

- **Live from this working copy** — the environment URL reaches a dev server
  running from this repo (for example through a tunnel), so a saved file is
  live at once. Take this only when the user or the repo's instructions
  (CLAUDE.md, README) say so; never guess from the URL alone.
- **Live after deploy** — anything else. The fix is live only after the PR is
  merged and deployed.

## Step 1 — Reproduce

`/cqa-bug-repro`, the platform run that reports on the ticket. Go on only on
`REPRODUCED`.

- `NEEDS_INFO` → ask the user for the missing facts, `submit_info`, wait again.
- `NOT_REPRODUCED` → stop. Check the environment URL before concluding the bug
  is gone.
- `INTERNAL_ERROR` → stop and report. Do not retry blindly.

## Step 2 — Fix

The reproduction is the spec: the failed step and expected vs actual say what
"fixed" means.

1. **Find the cause.** Trace from the screen, route or field named in the
   failed step to the code that produces the wrong value. Name the file and
   line before changing anything. Unsure after reading → say what you checked
   and ask; never ship a guess.
2. **Smallest fix** that turns actual into expected, on a branch from the
   up-to-date base, e.g. `fix/<ticket>-<short-name>`. Follow the repo's own
   conventions. Add or update a test when the repo has tests, and run them.
3. **Don't touch the data ContextQA tests against** while checking — no
   requests that create, edit or delete records in that environment.

## Step 3 — Verify (when live from this working copy)

The fix is already live, so prove it before anyone reviews it:
`/cqa-verify-fix`. `FIXED` → Step 4, with the verdict in hand. `NOT_FIXED` →
back to Step 2 with the new evidence, at most **3 attempts** in total.

Live after deploy → skip this step; verification happens in Step 6.

## Step 4 — Open the PR

Commit message and PR title start with the ticket key. PR body: the bug in one
line, the cause (file:line), the fix, how it was checked, the reproduction's
evidence link — and, when Step 3 ran, ContextQA's verdict with its link. Push
and open it; report the URL. Never force-push. Never commit secrets or `.env`
changes.

## Step 5 — PR impact

Only when the repo is enrolled for PR impact for this PR's base branch
(`/cqa-impact`); otherwise say it is not enrolled and go on.

Run the impacted cases against an environment that serves **the PR's build**.

- A failure → read the evidence. Product regression → fix it on the same
  branch and push; broken test → that is `/cqa-debug`, with the user's OK.
- At most **3 rounds**. Still red after 3 → stop and hand over with all three
  attempts.
- Nothing impacted is not the same as safe. Say which one it is.

## Step 6 — A person merges

Stop and tell the user: PR link, the verify verdict (if Step 3 ran), the impact
verdict, and that the PR is ready for them to merge. Never merge yourself, and
never approve your own PR.

Live after deploy → once the user says it is merged and deployed, run
`/cqa-verify-fix`.

## Step 7 — Summary

Ticket; each verdict with its time (reported → reproduced → fixed → verified →
PR → impact); the PR link; the regression `test_case_id`; credits used (one
reproduction, one re-check, the impacted runs); and how many times a person had
to step in.

## Rules

- Each stage starts only when the one before it has a real result.
- Credits: ask before anything beyond one reproduction, one re-check per fix
  attempt and the impacted runs of this PR.
- Quote ContextQA's own text for verdicts and failures; never fill gaps with
  a guess.
