---
name: verify
description: Drive index.html in headless Chrome to verify editor changes end-to-end (load real proto, scroll, search, export)
---

# Verifying M2-ItemProtoEditor changes

No build step. Surface = `index.html` opened in a browser via `file://`.

## Handle

Playwright is not a repo dep. Install `playwright-core` in a scratch dir
(`npm i playwright-core`, ~1s) and launch system Chrome:

```js
const { chromium } = require('playwright-core');
const path = require('path');
const browser = await chromium.launch({ channel: 'chrome', headless: true });
const page = await browser.newPage({ viewport:{width:1280,height:800}, acceptDownloads:true });
// run with cwd = repo root; pathToFileURL handles Windows backslashes
await page.goto(require('url').pathToFileURL(path.resolve('index.html')).href);
await page.waitForSelector('input[type=file]');
```

A real 11,005-row test file lives at `item_proto.txt` in the repo root
(gitignored, machine-local — not in clones).

## Flows worth driving

- **Load proto**: `page.setInputFiles('input[type=file] >> nth=0', PROTO)` (nth=1 is
  Load names), then `waitForFunction` on footer text `'11005 items'`. Time it.
- **List scroll**: find scroll container by `scrollHeight >> clientHeight`, set
  `scrollTop`, check `innerText` shows correct absolute gutter indices / vnums.
  List is windowed (virtualized) — total DOM should stay ~650 nodes; full-render
  scrollHeight reference is 517,235px for the repo-root test file.
- **Search**: fill `input[placeholder*="Search"]`, check `N / 11005 items` footer.
- **Export**: click `text=↓ Export both`, collect 2 downloads via `page.on('download')`,
  `download.saveAs()`. Byte-preservation check: unedited rows must be byte-identical
  to input (compare `latin1` lines split on `/\r?\n/` — export writes LF, no
  trailing newline; input may be CRLF).

## Gotchas

- Startup logs a CORS `fetch` error for the page itself under `file://`
  (dc-runtime self-fetch). Harmless, pre-existing — app still boots.
- Panel/group titles are CSS-uppercased; `innerText` returns `IDENTITY`, not
  `Identity` — match case-insensitively.
- Rows are the `cursor:pointer` children of the scroll container; header row and
  virtualization spacer divs are siblings without pointer cursor.
