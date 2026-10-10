# Verification

A report is a claim. Evidence is a number produced by a command you can paste
into the next session.

## First find the project's own tools

Never hardcode a client, a port, or a host: they differ per project. Discover
them from the repo before measuring.

| you need | where it is written down |
|---|---|
| install / run / test / lint commands | manifest scripts (`package.json`, `pyproject.toml`, `Makefile`, `justfile`, `compose.yml`), README, CI config |
| the port and base URL | the dev-server script, `.env`/`.env.example`, `compose.yml`, framework config; or the output the server prints when it starts |
| the datastore and its client | migration files, `compose.yml`, ORM config, docs — then use that client (SQL shell, `curl` against the API, file reader, key-value CLI) |
| the auth/session mechanism | middleware, cookie or header names in the login flow |

Two consequences: a measurement that needs a tool the machine does not have is
not a measurement, and a command copied from another project is a guess.

## The ladder

Go as deep as the change reaches, then stop:

1. **Files** — `git status --short`, `git diff --stat`: only whitelisted files.
2. **Static** — run the project's typecheck/lint/test and capture exit codes
   (`cmd; echo $?`). An unrun check is not a passing check.
3. **Data** — read the store with the project's own client; compare with the
   claimed change.
4. **Runtime** — drive the real surface: click, submit, reload, call, restart.
5. **Visible** — for a UI, the rendered state of the elements, not their presence
   in the DOM.

1–2 catch an executor that lies. 3–5 catch one that is confidently wrong.

## Generic recipes

**Datastore.** One read-only query before, one after, against the row the surface
renders — not just a row count. Counts stay equal while content rots.

**HTTP surface.** Request the exact URL the app requests at runtime, and check the
status the app sees, not the status of the file on disk:

```bash
curl -s -o /dev/null -w '%{http_code}\n' "<url the app/frontend actually requests>"
```

Assets are the classic case: the raw file answers 200 while the URL the frontend
builds (image optimizer, CDN path, signed URL, relative path) answers 4xx — and
the page shows an empty box.

**Browser UI.** Counts and computed values, not screenshots:

```js
document.querySelectorAll('<selector>').length
[...document.querySelectorAll('<selector>')].filter((el) => {
  for (let n = el; n; n = n.parentElement)
    if (getComputedStyle(n).visibility === 'hidden' || getComputedStyle(n).opacity === '0') return true;
  return false;
}).length                                   // must be 0 to count as "visible"
document.getAnimations().length             // 0 when the animation that reveals content never exists
document.elementFromPoint(x, y) === el      // proves the target is actually hittable
```

Then click real coordinates after `scrollIntoView()`, and assert the state change
afterwards. `el.click()` hides overlay bugs.

**Event handlers alive.** A component that renders fine can have no handlers at
all — the page looks correct, the form even saves through a server path, and
nothing responds in the browser. Only a real interaction that changes state
proves the runtime is alive. A screenshot never does.

**The executor's actual work.** `tee` stdout to a log and grep the tool calls it
made. A claim with no tool call behind it is not work.

## Traps (each one cost a debugging session)

- Box measurements lie about visibility: a zero-opacity or clipped element still
  reports a position and a size. Check computed style **up the whole ancestor
  chain**, plus paint state for animations.
- Never inject CSS or JS that changes what you are measuring. If the output looks
  empty, that is the finding. An override added "just to see it" can hide the bug
  for months.
- Lazy/deferred content is not missing content. Scroll, wait, or await the
  platform's decode/ready signal before declaring something broken; flipping a
  loading flag after load does not restart the fetch.
- Background or headless contexts freeze animations and painting. Finish
  animations (`document.getAnimations().forEach((a) => a.finish())`) instead of
  overriding styles, and never treat a frozen frame as a finding.
- Vision models judging screenshots report confident false negatives. Numbers
  decide; screenshots illustrate.
- Automated file pickers set the input's files without firing the framework's
  change event; dispatch a real `change` event (with `bubbles: true`) and confirm
  the submit control is enabled before clicking.
- Targets inside scrolled containers sit off-screen. Scroll first.
- HTTP 200 pages are routinely broken: assert the state change, not the status.
- Anything that writes data automatically (importer, migration, fixer) must be
  followed by requesting the surface that renders that data.
- An agent that answers a question from "context" may simply have searched the
  repo and opened a file. To know what it actually loaded, ask the tool to list
  its instruction sources or read its debug output — a correct answer alone
  proves nothing (`bootstrap.md`).
- Long-running dev servers wedge after many requests (hanging curls, empty
  responses). Restart the process, free the port, re-measure.

## What counts as proof

A pair: command → output, before and after the change. Not "looks right", not
"should work", not the executor's summary. Report the ones you ran; name the ones
you did not and why. A check you cannot run is a risk to state, not a detail to
skip.
