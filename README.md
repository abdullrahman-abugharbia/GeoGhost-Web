# GeoGhost landing page

The page you give a customer. One static file plus the APK — no build step, no server code.

```
site/
  index.html             the page (English + Arabic, RTL)
  logo.svg               favicon, same mark as admin/logo.svg
  GeoGhost.apk           the Android download
  GeoGhost-Windows.zip   the desktop download, app + bundled engine
  GeoGhost-macOS.zip     the desktop download — not built yet, see below
```

Upload the whole folder to any static host (Netlify, Cloudflare Pages, GitHub Pages, an
S3 bucket, or a plain `public_html`). The download link is relative, so the APK has to sit
next to `index.html`.

## Shipping a new build

`flutter build apk --release` writes to
`mobile/build/app/outputs/flutter-apk/app-release.apk`. Copy that over
`site/GeoGhost.apk`, then update `version` and `size` in `CONFIG` at the top of the
`<script>` block in `index.html`:

```js
const CONFIG = {
  channel: 'apk',
  url: 'GeoGhost.apk',
  version: '1.0.0',
  size: '58.6 MB',
  minAndroid: '8.0',
  desktop: { version: '1.0.0', maxIos: '16',
             windows: { url: 'GeoGhost-Windows.zip', size: '47.9 MB' },
             mac:     { url: '', size: '' } },
  contact: { email: '', whatsapp: '', website: '' }
};
```

## The desktop downloads

The **Get GeoGhost** band carries two cards: the Android app, which installs on the phone, and the
desktop app (`desktop/`), which moves a **USB-connected iPhone** from a Windows PC or a Mac. The
FAQ's iPhone answer describes the same thing, so the two stay consistent.

### Windows — shipping

`GeoGhost-Windows.zip` holds the Flutter release build and the Python engine side by side, so the
customer unpacks one folder and runs it. Nothing else has to be installed except Apple's USB driver,
which `README.txt` inside the zip explains.

```
GeoGhost/
  GeoGhost.exe        renamed from geoghost_desktop.exe
  flutter_windows.dll
  data/
  agent/              PyInstaller output: agent.exe + _internal/
  README.txt          what the customer reads first (English + Arabic)
```

The app finds the engine at `<folder next to the exe>/agent/agent.exe`, which is one of the paths
`IosAgent._locate()` already looks in, so the layout above is the whole wiring. Do **not** ship
`desktop/agent/.venv` instead: a Windows venv points at the Python install that created it and will
not run anywhere else.

Rebuilding the zip after a change:

```bash
cd desktop/app && flutter build windows --release
cd ../agent && .venv/Scripts/python -m PyInstaller --noconfirm --onedir --name agent --console \
  --collect-all pymobiledevice3 --copy-metadata pymobiledevice3 \
  --exclude-module IPython --exclude-module jedi --exclude-module parso \
  --exclude-module PIL --exclude-module av --exclude-module matplotlib \
  --exclude-module pytest --exclude-module setuptools agent.py
```

The exclusions matter: without them the bundle is 179 MB instead of 72 MB, almost all of it PyAV's
bundled FFmpeg and IPython's autocompletion, neither of which the three calls the agent makes ever
load. Then copy `app/build/windows/x64/runner/Release/*` and `agent/dist/agent` into one folder,
rename the exe, add `README.txt`, zip it, and update `size` in `CONFIG`.

### Still missing

- **macOS** — never built. `flutter build macos --release` only runs on a Mac, and the engine needs
  the same PyInstaller step there (no `.exe` suffix).
- **iOS 17+** — the engine drives **iOS 16 and below**. Newer phones are recognised and reported as
  `ios17_needs_tunnel`; the tunnel is the next step in `desktop/README.md`. `maxIos` prints that limit
  under the buttons — set it to `''` once the tunnel lands.
- **Code signing** — the exe is unsigned, so Windows SmartScreen warns on first run. `README.txt`
  tells the customer to choose *More info → Run anyway*.
- **Window title** — the taskbar still reads `geoghost_desktop`; it is the `window.Create(...)` call
  in `desktop/app/windows/runner/main.cpp`.

To turn the remaining button on, drop the archive next to `index.html` and fill in its two fields:

```js
mac: { url: 'GeoGhost-macOS.zip', size: '64 MB' },
```

An empty `url` keeps that platform showing *Coming soon*, which is where macOS sits. Setting
`desktop` to `null` removes the whole card and leaves the Android one alone, centred.

## Moving to Google Play

Set `channel: 'play'` and put the listing URL in `url`. The button label, the file size
next to it, the download metadata and step 01 of "Three steps, once" all change with it,
in both languages. Nothing else has to be touched.

## Contact links

The three `contact` fields are empty, so the footer shows no links. Fill in any of them
and that link appears, labelled in whichever language the visitor is reading.

## Chat

The round button in the corner opens a chat. Messages land in the dashboard under
**Chats** (`admin/dashboard.html`), where the admin replies; the reply shows up in the
visitor's chat window, and a green dot on the button tells them when it is closed.

- One-time setup: run `supabase/setup.sql` again in the Supabase SQL Editor. It adds the
  two chat tables and the `chat_send` / `chat_fetch` functions, and leaves the codes alone.
- `CONFIG.chat` holds the same project URL and publishable key as the dashboard. Empty
  values hide the chat button.
- The visitor's browser keeps a random token that is the only key to their conversation.
  Opening the page in another browser starts a new chat, so they can leave a WhatsApp
  number or email with their first message.
- No live connection: the page checks for replies every 4 seconds while the chat is open
  and every 30 seconds while it is closed. The dashboard checks every 5 seconds.
- A visitor can send at most 8 messages a minute, up to 2000 characters each.

## Editing the text

Every string lives in the `T` object (`T.en` and `T.ar`, keyed to the `data-i18n`
attributes in the markup); the questions live in `FAQ`. Change both languages together.

Brand colours at the top of the `<style>` block are the same values as
`admin/dashboard.html` and `mobile/lib/theme.dart` — keep the three in sync.

## Previewing locally

```bash
python -m http.server 8766 --directory site
```

Or run the `site` configuration in `.claude/launch.json`.
