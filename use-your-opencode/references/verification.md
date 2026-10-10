# Verification

A report is a claim. Evidence is a number produced by a command you can paste
into the next session.

## The ladder

Go as deep as the change reaches, then stop:

1. **Files** — `git status --short`, `git diff --stat`: only whitelisted files.
2. **Static** — `cmd; echo $?` for typecheck and lint. An unrun check is not a
   passing check.
3. **Data** — query the store, compare with the claimed change.
4. **Runtime** — drive the real surface: click, submit, reload, `curl`.
5. **Visible** — attributes, geometry, and computed style in a real browser.

1–2 catch an executor that lies. 3–5 catch one that is confidently wrong.

## Recipes

**Database — read-only, always.**

```bash
psql "$DATABASE_URL" -At -c 'SELECT slug, "fotoJson" FROM "Model" ORDER BY id LIMIT 8'
```

Check the row the page should render, not just the row count. Counts stay equal
while content rots.

**HTTP surface.**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/koleksi
curl -s -o /dev/null -w '%{http_code}\n' 'http://localhost:3000/_next/image?url=%2Fimages%2Fmodel%2Fx.jpg&w=640&q=75'
```

Hit the image-optimizer URL explicitly: that is where `next/image` fails
silently (SVG, relative paths) while the raw file answers 200.

**DOM — counts and computed values, not screenshots.**

```js
document.querySelectorAll('.reveal-item').length
document.getAnimations().length
[...document.querySelectorAll('.reveal-item')].filter((el) => {
  for (let n = el; n; n = n.parentElement) if (getComputedStyle(n).opacity === '0') return true;
  return false;
}).length                                   // must be 0
[...document.querySelectorAll('button[aria-pressed]')].map((b) => b.getAttribute('aria-pressed'))
document.elementFromPoint(x, y) === el      // proves the target is hittable
```

`el.click()` hides overlay bugs; click real coordinates after `scrollIntoView()`
and assert the state change afterwards.

**Hydration.** A `"use client"` component that reaches database-only code still
renders, still lets server actions save, and simply has no event handlers. Only a
real click that flips state proves React is alive; a screenshot never does.

**The executor's actual work.** `tee` stdout to `/tmp/<name>.log` and grep the
tool calls it made. A claim with no tool call behind it is not work.

## Traps (each one cost a debugging session)

- `getBoundingClientRect()` returns a size for `opacity: 0` elements. Visible
  means `getComputedStyle(el).opacity` non-zero **up the whole ancestor chain**.
- Never inject CSS or JS that changes what you are measuring. If the page looks
  empty, that is the finding. An override added "just to see it" hides the bug
  for months.
- Lazy images below the fold have `naturalWidth === 0` until fetched. Scroll,
  wait, `await img.decode()` in `try/catch` — then judge. Setting `loading` after
  load does not restart the fetch.
- Headless/hidden tabs freeze animations and painting; prefer finishing
  animations (`document.getAnimations().forEach((a) => a.finish())`) over
  overriding styles, and never treat a frozen frame as a finding.
- A vision model judging a screenshot reports missing images that are
  `complete: true` with a real `naturalWidth`. Numbers decide; screenshots
  illustrate.
- CDP file inputs set `input.files` without firing React's `onChange`. Build a
  `DataTransfer`, dispatch `new Event('change', { bubbles: true })`, then confirm
  the submit button is actually enabled before clicking it.
- Buttons inside scrolled rows sit off-screen. `scrollIntoView()` first.
- HTTP 200 pages are routinely broken: assert the state change, not the status.
- Anything that writes data automatically (importer, fixer, seeder script) needs
  the page that renders it requested afterwards. A stored path that produces
  HTTP 500 is invisible to a storage-only check.
- `pnpm dev` can wedge after many optimizer requests (curl hangs, code 000):
  kill the dev process, free the port, restart, then re-measure.

## What counts as proof

A pair: command → output, before and after the change. Not "looks right", not
"should work", not the executor's summary. Report the ones you ran; name the ones
you did not and why. A check you cannot run is a risk to state, not a detail to
skip.
