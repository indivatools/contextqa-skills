---
name: cqa-locators
description: Use when authoring or repairing ContextQA test steps against a UI whose locators are known or discoverable — an existing case, a saved element, a page you can open in a browser, or an AI bootstrap you convert. Produces typed NLP steps (navigate / enter / click / verify) with real `pwLocator` sets and named elements, instead of opaque AI-agent steps. Triggers on "the locators are known", "write the steps by hand", "use the element I saved", "this step can't find the element", "convert the AI steps to real steps", "/cqa-locators".
---

# ContextQA — Locator-First Step Authoring

**The goal is a case made of typed steps you can edit.** A typed step is
deterministic, diffable, and repairable one field at a time. An AI-agent step is
a black box that re-derives its own actions on every run — N AI steps means N
independent agent invocations, N times the cost and N times the nondeterminism.

AI has exactly two legitimate uses here: **bootstrap** a page you have never
seen, and **fallback** for an action typed steps genuinely cannot express.

> **Never "fix" a case by deleting it and dropping in one AI step.** That throws
> away every typed step and all the work in them. Edit with
> `manage_test_step(action="update")`.

## Step 0 — Refresh the action catalogue

`naturalTextActionId` values are **tenant- and version-specific**. Hard-coding
one silently produces the wrong step.

```
list_natural_text_actions(workspace_type="WebApplication")   # or MobileApplication
```

Look up by `display_name` (`navigateToUrl`, `enter`, `click`, `clickIfPresent`,
`verifyTextContains`, `ai_verify`, …) and use the id it returns, this session.

## Step 1 — Get real locators. Five sources, in cost order

You cannot invent an id like `#v-0-0-0`. Take it from one of:

1. **An existing case on the same page.** `manage_test_step(action="list",
   test_case_id=N)` returns raw wire dicts including `event.pwLocator` and
   `event.selector`. Cheapest and most common.
2. **A saved element.** `manage_element(action="get_by_name", name="Login
   button")` — bind the step by `element_id` and the locator is maintained in
   one place for every case that uses it.
3. **Search across the workspace** when you know the label but not the case:
   `search_test_steps("element:*Sign in*,")` returns the matching steps **and**
   the deduplicated `test_case_ids`. Never repeat a key in that grammar — it
   ANDs and silently matches nothing; use `@` for in-list.
4. **A browser.** If your agent has a browser tool (a Playwright MCP, an
   in-client browser, Claude in Chrome), open the page and read the DOM. The
   most self-sufficient route: no run, works on authenticated pages.
5. **An AI bootstrap, executed.** `create_test_case(task_description=...,
   environment_id=N)` makes one AI step; **executing it converts the case in
   place into typed steps with captured locators.** ~100–200s per run, and it
   handles interstitials you would never have anticipated. Then read the
   converted steps with `manage_test_step(action="list")` and hand-author from
   their locators.

### Pick locators that survive a re-render

The `element` field is a **meaning** ("E-mail address field"), and the platform
re-resolves it at run time when the raw selector drifts. Give it good raw
candidates to start from:

- **Prefer** stable ids, `data-testid`, BEM classes, attribute selectors
  (`input[type='email'][placeholder='…']`), and text-anchored selectors.
- **Avoid** framework-generated ids (`#v-1-1` shifts across renders), absolute
  XPaths through a modal portal, and CSS that matches twice in a grid that
  renders a frozen pane and a scroll pane.
- Supply **several** `pwLocator` candidates, not one. That list is the fallback
  chain.

## Step 2 — Write the step

A typed step that targets an element **must be able to find it at run time**,
via either `element_id` or `event: {pwLocator: [...], selector: "..."}`. With
neither, the step saves cleanly and then fails after a 30s timeout — so the MCP
rejects it at author time. (`allow_missing_locator=True` defers this only if you
intend to fill them in later.)

```
manage_test_step(action="create", test_case_id=N, step={
  "kind": "nlp",
  "action": "<the action HTML>",
  "display_name": "enter",
  "element_id": null,
  "test_data": "*|username|",
  "environment_id": 497
}, dry_run=True)
```

`dry_run=True` returns the exact payload that would be sent — review it, then
replay with `dry_run=False`. Every write action supports it.

### The wire shape that actually runs

Believe this, captured from an executed, passing `enter` step — not intuition:

```json
{
  "action": "Enter <span data-key=\"test-data\" data-event-key=\"value\" class=\"test_data\">*|username|</span> in the <span data-key=\"label\" data-event-key=\"label\" class=\"element\">E-mail address</span>",
  "element": "E-mail address",
  "testDataType": "raw",
  "testData": "*|username|",
  "environmentId": 497,
  "requestList": {},
  "event": {
    "customEvent": "enter", "label": "E-mail address", "value": "*|username|",
    "isIFrame": false, "iFrameLocator": null,
    "pwLocator": ["#email", "xpath=//input[@type='email']", "input[type='email'][placeholder='you@acme.com']"],
    "selector": "#email"
  }
}
```

Five rules that trip everyone:

- **`testDataType` is `"raw"` even for an environment variable.** `*|url|`
  resolves against the bound `environmentId` regardless. `"environment"`
  produces a dead step.
- **`navigateToUrl` needs `event.href` AND `requestList.url`.** Without a URL it
  fails in ~188ms reporting "network idle" — a misleading message for a step
  that simply has no address.
- **`enter` / `click` / `verify` need the full `event`** — `label`, `value` for
  `enter`, and `pwLocator[]` + `selector`. Missing locators give a 30s timeout
  reading `" is not clickable on the page…"`, where the leading space is the
  empty element name.
- **Never include `contenteditable` in the action HTML.** The platform emits it
  sometimes; the authoring API's XSS validator rejects it (`Disallowed attribute
  'contenteditable'`). Allowed attributes are `class` and `data-*`.
- **`<>` is stripped from step text** and **`position` is ignored on create** —
  steps append. Reorder afterwards.

## Step 3 — Promote repeated locators to elements and screens

The second time a locator appears in two cases, it belongs in one place.

```
manage_screen(action="create", name="Login", url="/login")        # idempotent by name
manage_element(action="create", name="Login button",
               locator_type="csspath", locator_value="button[type='submit']",
               screen_name_id=<screen id>)
```

`locator_type` is one of `xpath`, `csspath`, `id_value`, `name`, `link_text`,
`partial_link_text`, `class_name`, `tag_name`, `accessibility_id`.
`manage_element(action="list")` is not supported — look up by name with
`get_by_name`, which is also the load-bearing call when recreating a case in
another workspace.

Repeated *sequences* — login, seed a cart, reach a settings page — belong in a
**step group**: `manage_test_case(action="create", is_step_group=True)`, author
its steps normally, then embed it with a `step_group` step. A group that would
contain its own test case is rejected at save.

## Step 4 — Bind the data, not the value

`*|name|` for an environment variable, `${name}` for a case variable, `{{name}}`
for a global, and a **bare** column name for a test-data profile. See
`cqa-environments` — this is where most "it saved and then failed" comes from.

## Step 5 — Execute and fix, one step at a time

```
execute_test_case(test_case_id=N)            # share live_url with the user FIRST
get_execution_status(session_id=..., wait=True, timeout=60)
get_execution_step_details(result_id=R)      # failure_reason names what broke
```

Read `result` — the executor's own verdict. It is `null` while `is_completed`
is false, and a new result row appearing is **not** a pass.

Then: fix the offending step's fields, re-run. A well-formed step turns red →
green and the **next** step becomes the new failure; that forward march is how
you know the fix was real. **Cap at ~3 attempts per case.** If it is still red,
report the actual reason rather than thrashing — and never delete a failing case
to tidy the numbers.

## Repairing an AI-heavy case you inherited

1. `manage_test_step(action="list")` — see what is actually there.
2. `manage_test_step(action="coalesce_ai_steps", test_case_id=N)` — merges each
   run of consecutive AI Agent steps into one step carrying the whole numbered
   sequence. Defaults to `dry_run=True`; review the plan, then apply.
   `ai_verify` steps are boundaries, not members — folding a verification into
   an action step would erase its pass/fail meaning.
3. Execute once. The conversion captures locators.
4. Replace the AI steps with typed ones using those locators, highest-value
   flow first.

## Known sharp edges

- **`verifyAttribute` can fail on an element `enter` succeeded on one step
  earlier** (post-input re-render). Prefer `verifyTextContains` for post-input
  assertions.
- **Step lists come back unsorted.** Sort by `position` yourself.
- **`bulk_operation` can silently no-op.** Verify with a fresh `list` after a
  bulk write.
- **Vue/SPA inputs:** setting `.value` directly often does not update the
  framework's model and the field resets on re-render. Real keyboard events
  (`page.fill()` / `.type()`) are what work — relevant when you do recon
  yourself.

Ground truth for the wire format lives on the server:
`get_contextqa_skill(name="cqa-authoring-tests")`.
