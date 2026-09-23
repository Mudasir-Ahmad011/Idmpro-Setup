# IDM Pro

A download manager built the way IDM is built: a desktop app that owns the
downloading, plus a browser extension that hands it every download you start in
Chrome or Edge.

- **Multi-connection downloads** — a file is split across up to 32 parallel HTTP
  range requests written into one sparse file.
- **Real pause and resume** — per-segment offsets are persisted, so a pause, a
  crash, or a reboot resumes from the exact byte it stopped at.
- **Browser capture** — the extension cancels the browser's own download and
  forwards the URL together with the cookies, referer and user agent that made
  it work, so session-protected links keep working.
- **Video downloads** — YouTube and ~1800 other sites, via yt-dlp for stream
  extraction and ffmpeg for muxing. The streams themselves are fetched by the
  normal engine, so video downloads get the same parallel connections and the
  same resume behaviour as any other file.
- **In-page download panel** — a button floats over any video on any page;
  clicking it lists the real available qualities and sends the chosen one
  straight to the queue.
- **Windows installer** — `Setup.exe` installs the app and registers the native
  messaging host for every Chromium browser on the machine.

---

## The three pieces

Downloads reach the engine through this chain:

```
Chrome / Edge
     |  chrome.downloads.onCreated -> cancel -> sendNativeMessage
     v
extension/background.js
     |  native messaging (4-byte length prefix + JSON over stdio)
     v
host/idmpro-host.js                   <- launched by the browser, not by us
     |  HTTP POST to 127.0.0.1:<random port>, bearer token from bridge.json
     v
src/main/bridge.js
     |
     v
src/engine/  ->  N parallel range requests  ->  one file on disk
```

| Folder | What lives there |
|---|---|
| `src/engine/` | The download engine. No Electron, no UI — plain Node, separately testable. |
| `src/main/` | Electron main process: windows, tray, IPC, the loopback bridge, registry registration. |
| `src/renderer/` | The UI (queue list, settings, the "Download File" prompt). |
| `host/` | The native messaging host the browser launches. Dependency-free by necessity. |
| `extension/` | The MV3 extension for Chrome and Edge, including the in-page panel. |
| `build/` | `installer.nsh` — the custom NSIS steps that wire up the browsers. |
| `tools/` | `yt-dlp.exe` and `ffmpeg.exe`. Fetched by a script, never committed. |
| `scripts/` | Icon generation, key generation, host registration, tests. |

### Why a native messaging host at all

A browser extension cannot talk to a desktop program directly. Chrome will,
however, launch a local executable listed in a manifest whose path sits in the
registry under `HKCU\Software\Google\Chrome\NativeMessagingHosts\`, and pipe
JSON to it over stdin/stdout. That is the only sanctioned route, and it is why
IDM needs both a setup *and* an extension you enable once.

The host is deliberately tiny: it forwards the job to the running app and, if
the app is not running, starts it and waits. On an end user's machine there is
no Node installed, so the host runs under the app's own Electron binary in Node
mode (`ELECTRON_RUN_AS_NODE=1`, set by `host/idmpro-host.bat`).

### Why the extension ID is pinned

The native messaging manifest lists exactly which extension IDs may call the
host. An unpacked extension normally gets a random ID, which would break that
list on every machine. `scripts/gen-extension-key.js` generates an RSA key once
and pins it into `manifest.json` as the `key` field, so the ID is stable
everywhere: `dhefpheeoofkhdlojhbomngnppokacoe`.

Keep `extension/key.pem` private. It is the identity of the extension.

---

## Running from source

```bash
npm install
npm run fetch-tools    # download yt-dlp + ffmpeg into tools/ (~175 MB, once)
npm start              # launch the app
npm run register-host  # let Chrome/Edge find the host in this source tree
```

Everything except video downloading works without `fetch-tools`; the app simply
reports that a link is a web page rather than trying to extract it.

Then load the extension:

1. open `chrome://extensions` (or `edge://extensions`)
2. turn on **Developer mode**
3. **Load unpacked** → select the `extension/` folder

The app also does the registration itself on every launch, so `register-host`
is only needed if you want the browser wired up before first run. Undo it with
`npm run unregister-host`.

### Downloading without the UI

```bash
node scripts/cli.js https://example.com/big.iso -n 16 -o D:\Downloads
```

Same engine the app uses, with a progress bar in the terminal.

---

## Building the installer

```bash
npm run dist
```

Produces `dist/IDMPro-Setup-1.0.0.exe`. The installer:

1. copies the app, the host and the extension into the install folder
2. writes the `NativeMessagingHosts` registry value for Chrome, Edge, Chromium,
   Brave, Vivaldi and Opera
3. runs the app once with `--install-host` so the manifest's `path` points at
   the directory the user actually chose
4. drops a `LOAD-ME.txt` next to the bundled extension

Uninstalling removes the registry values but **keeps** `%APPDATA%\IDMPro`, so
unfinished downloads survive a reinstall.

### If the build fails on `winCodeSign`

On a Windows machine without Developer Mode, extracting electron-builder's
code-signing toolchain fails:

```
ERROR: Cannot create symbolic link : A required privilege is not held by the client.
        ...\winCodeSign\<id>\darwin\10.12\lib\libcrypto.dylib
```

The archive carries macOS symlinks that Windows refuses to create without the
`SeCreateSymbolicLinkPrivilege`. Because the extraction fails, the cache never
populates and every build re-downloads it. Two fixes:

- **Turn on Developer Mode** (Settings → System → For developers). Permanent fix.
- **Or populate the cache by hand**, skipping the two macOS symlinks:

  ```bash
  CACHE="$LOCALAPPDATA/electron-builder/Cache/winCodeSign"
  # exits with code 2 on the two symlinks -- that is expected
  node_modules/7zip-bin/win/x64/7za.exe x -snld -y \
      "$CACHE/<downloaded>.7z" "-o$CACHE/winCodeSign-2.6.0"
  rm -rf "$CACHE/winCodeSign-2.6.0/darwin"
  ```

  `winCodeSign-2.6.0` is the exact directory name electron-builder looks for.
  Only `rcedit` and `signtool` from that archive are used on Windows, so
  dropping the `darwin` folder costs nothing.

> The installer is unsigned, so Windows SmartScreen will warn about an unknown
> publisher. Fixing that needs a code-signing certificate, which is a purchase,
> not a code change. Point `CSC_LINK`/`CSC_KEY_PASSWORD` at the `.pfx` once you
> have one and electron-builder signs automatically.

---

## Tests

```bash
npm test                            # all three suites

node scripts/test-engine.js         # engine correctness
node scripts/test-integration.js    # the whole browser handoff chain
node scripts/test-video.js          # format selection and tool presence
node scripts/test-video.js --live   # downloads a real video end to end
```

`test-engine.js` runs a 24 MB download against a local range-capable server,
pauses it mid-flight, tears the manager down, rebuilds it from the persisted
JSON, resumes, and checks the SHA-256 of the result against the source. It also
checks the fallback path for servers that ignore `Range`.

`test-integration.js` replaces Chrome with a script that speaks the same native
messaging protocol, then verifies the handoff end to end: stdio framing, the
loopback token (including that unauthenticated callers get a 401), cookie and
referer pass-through, and the bytes on disk.

`test-video.js` checks format selection against a recorded YouTube format list
— quality caps, storyboard and manifest entries rejected, container fallbacks —
without touching the network. With `--live` it downloads a real video and asks
ffmpeg whether the result actually contains both a video and an audio stream,
then repeats the exact round trip the in-page panel performs: list the
qualities, pick the lowest one, and confirm the file that comes back really is
that resolution rather than the best available.

---

## How video downloading works

A page like `youtube.com/watch?v=...` is HTML. There is no file at that address
— the picture and the sound are separate DASH streams on `googlevideo.com`,
behind URLs that are signed and expire within hours, and the signature is
produced by running the site's own player JavaScript.

So a video download is three jobs in one row of the queue:

```
  extracting   yt-dlp reads the page and reports every available stream
               -> pick the best mp4 video + m4a audio under the quality cap
  downloading  each stream is an ordinary DownloadTask: 8 connections,
               retries, resume, all of it
  merging      ffmpeg copies both into one container (no re-encoding, so it
               takes seconds and the streams stay bit-identical)
```

`VideoTask` composes two `DownloadTask`s rather than reimplementing them, and
exposes the same interface, so the queue, persistence and UI never need to know
which kind they are holding.

**Resume across expiry.** Signed stream URLs go stale, so every start
re-extracts. The format ids are matched against what was downloaded before; if
they still exist, the segment offsets are kept and only the URL is swapped. The
bytes for a given format do not change, just the address that serves them.

**Any HTML link is retried as a video.** When a plain download finds `text/html`
that the server never marked as an attachment, it fails with a clear message
rather than saving the page — and if the extractor is available, the manager
swaps that row for a video task and tries again. That covers video sites which
are not in the hardcoded list.

**Format choice.** An mp4/AVC video plus an m4a/AAC audio track is preferred,
because those remux into a `.mp4` that plays everywhere. Failing that, a webm
pair; failing that, any pair into `.mkv`; failing that, a single progressive
file with no muxing needed at all.

Two things worth knowing:

- Downloading from YouTube is against its Terms of Service, whatever tool does
  it. yt-dlp is legal, mainstream open-source software; how you use it is your
  call.
- **DRM-protected streams stay impossible.** Netflix, Spotify and similar
  encrypt their content by design. No download manager gets around that.

---

## The progress window

Each download can open its own detail window (`src/renderer/progress.*`),
showing the overall figures and, underneath, what every connection is doing:
a map of where each one sits in the file and a table of bytes, speed and state
per connection.

That detail is real rather than decorative. `download-task.js` now tracks a
`state` and a `speed` per segment -- connecting, receiving, retrying, done,
failed -- sampled with the same moving average as the whole-task figure. A
connection that stalls or is retrying is visible instead of being hidden
behind an averaged percentage.

Two things worth knowing:

- The **Speed limit** tab adjusts the *global* limit, because the engine has
  one shared rate limiter. The tab says so; per-download limits would need a
  limiter per task.
- **On completion** choices (open the file, open the folder, close the window)
  belong to that window only and are not persisted.

Turn the window off under Settings with "Show the progress window while a file
downloads". The details button on any row in the main list reopens it.

---

## Working out what a file actually is

Plenty of servers -- Google Drive among them -- send a name with no extension
and a content type of `application/octet-stream`, which says nothing. The
engine tries three things in order, stopping at the first that answers:

1. **The MIME type**, when the server states a real one.
2. **The server's own name**, preferred over a name handed over by the browser
   when that one carries no extension. Chrome frequently reports a Drive
   download as a bare title.
3. **The opening bytes of the file.** One small range request, made only when
   the first two failed, is enough to recognise PDFs, images, archives,
   executables and media by their magic numbers.

OOXML files need one extra step, because `.docx`, `.xlsx` and `.pptx` are all
zips: the entry names inside decide which it is (`word/`, `xl/`, `ppt/`),
falling back to `.zip` when it is a plain archive.

Without this, every such download landed in the Other folder with no type at
all. `scripts/test-filetype.js` covers all three layers, and
`npm run fix-extensions` repairs files saved before the fix.

---

## Two browsers, one source

`extension/` is the Chrome/Edge build; `extension-firefox/` is generated from
it by `npm run build-firefox` (and automatically before `npm run dist`, so it
cannot go stale). The JavaScript is shared byte for byte -- every file resolves
`globalThis.browser || globalThis.chrome` and uses the promise form of each
API, which both engines provide under MV3.

Only three things genuinely differ, and all three live in the generated
manifest or the host registration:

| | Chrome / Edge | Firefox |
|---|---|---|
| background | `service_worker` | `scripts` (event page) |
| identity | packed `key` | `browser_specific_settings.gecko.id` |
| native host | `allowed_origins` | `allowed_extensions`, under `Software\Mozilla\NativeMessagingHosts` |

Chrome rejects `background.scripts` under MV3 and Firefox has no MV3 service
worker, which is why two manifests exist rather than one shared file. The app
writes both native messaging manifests on every launch.

`scripts/test-intercept.js` loads the real background worker twice: once with
only `chrome` defined, once with only `browser`. Both must cancel the browser
transfer in the same tick and hand the job over.

Two Firefox constraints worth knowing, neither of which IDM Pro can remove:

- An unsigned add-on can only be loaded **temporarily**, so it disappears when
  Firefox restarts. Developer Edition or ESR with
  `xpinstall.signatures.required=false` installs it permanently.
- Firefox treats host permissions as **opt-in** under MV3, so site access has
  to be granted in about:addons before downloads are captured.

---

## Catching an ordinary file download

Two paths lead into the queue, because one alone is not enough.

**The download hook.** `chrome.downloads.onCreated` fires, the extension
cancels Chrome's transfer and hands the URL over with its cookies and referer.
The listener is deliberately *not* async and awaits nothing before the cancel:
a small file finishes downloading during a single `chrome.storage` round trip,
and a download that has already finished can no longer be cancelled, so Chrome
would keep it. Settings are hydrated when the worker starts and refreshed from
`storage.onChanged`, keeping the hot path synchronous.

**The link hook.** Some files never reach that event at all. Chrome renders a
PDF in its built-in viewer instead of downloading it, so no download is ever
created. `content.js` therefore watches clicks in the capture phase and claims
a link when the page marks it `download` or the URL ends in a file type people
save rather than read. Plain navigation, modified clicks (ctrl, shift, middle)
and non-file links are left alone, and if the desktop app cannot be reached the
click is handed straight back to the browser so nothing is ever stranded.

Both are covered by `scripts/test-intercept.js`, which evaluates the service
worker against a stub of the chrome APIs whose storage read is deliberately
slow — the delay the cancel used to be hiding behind.

---

## The in-page video panel

`extension/content.js` runs on every page, watches for `<video>` elements big
enough to be a real player, and floats a button over each one. The quality list
is not guessed in the browser — the extension cannot run yt-dlp — so clicking it
makes a round trip:

```
content script  ──get-formats──▶  background worker
                                       │ sendNativeMessage
                                       ▼
                                  native host
                                       │ POST /api/formats
                                       ▼
                                  extractor  ──▶  one entry per resolution,
                                                  each already paired with the
                                                  audio track it needs
```

Picking an entry sends its format ids back down the same path as a normal
download, and the queue honours that exact choice instead of picking "best".
Because choosing a quality *is* a confirmation, the desktop app skips its usual
"Download File" prompt for these.

Implementation notes worth keeping:

- The panel lives in a **closed shadow root** with a constructed stylesheet, so
  page CSS cannot reach it, its CSS cannot leak, and a strict page CSP cannot
  block it the way an inline `<style>` would be blocked.
- Fullscreen puts the video in the browser's top layer, where anything attached
  to `<body>` is hidden behind it. On `fullscreenchange` the panel re-parents
  itself into the fullscreen element.
- Position is re-read in a `requestAnimationFrame` loop, but only while the
  panel is visible, so idle pages pay nothing.
- Single-page sites swap their player without a navigation event, so panels are
  pruned whenever their `<video>` leaves the DOM. Without that, browsing
  YouTube would pile up dead panels and live listeners.
- If the extractor has nothing to offer, the panel falls back to the player's
  own `currentSrc` when that is a plain HTTP(S) file.

---

## How resume actually works

Each download owns a list of segments:

```json
{ "start": 3145728, "end": 6291455, "downloaded": 1048576 }
```

Every segment has its own file descriptor and writes at an absolute offset, so
the connections never interfere with each other. Progress is flushed to
`%APPDATA%\IDMPro\downloads.json` about once a second.

That flush can lag behind what is on disk, never lead it — so the worst case
after a hard crash is re-fetching a second of data at offsets that get
overwritten with identical bytes. Resuming asks each segment for
`bytes=(start + downloaded)-(end)` and carries on.

The partial file lives beside the destination as `<name>.idmdownload` and is
renamed into place only when every segment reports complete.

---

## Not implemented

- **Raw HLS/`.m3u8` links pasted directly.** Video *pages* work through the
  extractor, but pasting a bare manifest URL is not handled: the engine would
  need to fetch every segment and concatenate them itself.
- **Firefox.** It uses a different extension API surface and a different
  registry path for native hosts.
- **DRM-protected streams.** Not a gap to be filled — Netflix/Spotify-style
  content is encrypted by design and IDM cannot download it either.
- **FTP, torrents, proxy support.** HTTP and HTTPS only.
