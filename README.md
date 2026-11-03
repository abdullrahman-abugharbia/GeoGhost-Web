# GeoGhost landing page

The page you give a customer. One static file plus the APK — no build step, no server code.

```
site/
  index.html             the page (English + Arabic, RTL)
  logo.svg               favicon, same mark as admin/logo.svg
  GeoGhost.apk           the Android download
  GeoGhost-Windows.zip   the desktop download — not built yet, see below
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
             windows: { url: '', size: '' },
             mac:     { url: '', size: '' } },
  contact: { email: '', whatsapp: '', website: '' }
};
```

## The desktop downloads

The **Get GeoGhost** band carries two cards: the Android app, which installs on the phone, and the
desktop app (`desktop/`), which moves a **USB-connected iPhone** from a Windows PC or a Mac. The
FAQ's iPhone answer describes the same thing, so the two stay consistent.

Both desktop buttons currently read **Coming soon**, because no customer-ready build exists yet:

- **Windows** — `flutter build windows --release` produces `geoghost_desktop.exe`, but that build does
  **not** bundle the Python engine. On a machine without it the app only reports that it cannot find
  the agent. A shippable zip needs `agent.exe` next to it (PyInstaller over `desktop/agent/agent.py`),
  which is still on `desktop/README.md`'s to-do list. Copying `desktop/agent/.venv` instead does not
  work: a Windows venv points at the Python install that created it.
- **macOS** — never built. `flutter build macos --release` only runs on a Mac.
- Either way the engine drives **iOS 16 and below** today; iOS 17+ needs the elevated tunnel that
  `desktop/README.md` describes as the next step. `maxIos` puts that limit under the buttons — set it
  to `''` to drop the line once the tunnel lands.

To turn a button on, drop the archive next to `index.html` and fill in its two fields:

```js
windows: { url: 'GeoGhost-Windows.zip', size: '124 MB' },
```

An empty `url` keeps that platform showing *Coming soon*. Setting `desktop` to `null` removes the
whole card and leaves the Android one alone, centred.

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
