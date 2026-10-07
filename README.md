# Deployment notes

Telegram only opens Mini Apps over **HTTPS**, so both `index.html` and your tour JSON/images must be served from `https://` URLs.

## 1. Host the frontend on GitHub Pages

1. Create a public repo (e.g. `tour`) and add `index.html` and `sample_tour.json` to the root (put your panorama images in an `images/` folder).
2. Repo **Settings → Pages → Build and deployment**: Source = *Deploy from a branch*, Branch = `main` / `(root)`.
3. After a minute the site is live at `https://<username>.github.io/tour/`.
4. Check it in a browser: `https://<username>.github.io/tour/` should show the demo tour, and
   `https://<username>.github.io/tour/?config=sample_tour.json` should show the sample tour (relative paths work).

Use that Pages URL as `WEB_APP_BASE_URL`. Edit `sample_tour.json` so `panorama` points at your own images, e.g. `https://<username>.github.io/tour/images/living_room.jpg`.

## 2. CORS: what needs it and why

Two kinds of requests are cross-origin whenever they come from a different host than `index.html`:

| Request | Why CORS is required |
|---|---|
| The tour JSON (`fetch(configUrl)`) | Browsers block reading cross-origin responses without `Access-Control-Allow-Origin`. |
| Each panorama image | WebGL cannot use cross-origin images as textures unless they are served with CORS headers. |

**GitHub Pages** already sends `Access-Control-Allow-Origin: *` for every file, so JSON and images hosted there need no extra setup.

If you host files elsewhere, add the header:

- **nginx**
  ```nginx
  location ~* \.(json|jpg|jpeg|png|webp)$ {
      add_header Access-Control-Allow-Origin "*" always;
      add_header Access-Control-Allow-Methods "GET, OPTIONS" always;
  }
  ```
- **AWS S3 / Cloudflare R2**: bucket CORS configuration
  ```json
  [{ "AllowedOrigins": ["*"], "AllowedMethods": ["GET", "HEAD"], "AllowedHeaders": ["*"], "MaxAgeSeconds": 3600 }]
  ```
  (S3 uses this JSON array in the console; R2 accepts the same rule shape.) To lock it down, replace `*` with your Pages origin, e.g. `https://<username>.github.io`.
- **Hosts that cannot set headers** (some file-sharing links, Google Drive, Dropbox share pages): these will not work. Use GitHub Pages, S3/R2, or a CDN instead.

## 3. Run the bot

```bash
pip install -r requirements.txt
export BOT_TOKEN="123456:ABC..."                       # from @BotFather
export WEB_APP_BASE_URL="https://<username>.github.io/tour/"
python bot.py
```

No BotFather Mini App setup is needed: the button is an inline `web_app` button, which works as soon as the URL is HTTPS.

Test in a **private chat** with your bot:

- `/demo` opens the embedded demo tour.
- `/tour https://<username>.github.io/tour/sample_tour.json` opens the sample tour.

For a long-running bot, use systemd, Docker, or any process manager on a VPS (the bot uses long polling, so it needs no public port).

## 4. Tour JSON format

The frontend reads Pannellum's native multi-scene format:

- `default.firstScene`: the scene shown first (falls back to the first scene if missing).
- `scenes.<id>.title`: the room name shown in the floating label.
- `scenes.<id>.panorama`: URL of an equirectangular image.
- `scenes.<id>.hotSpots[]`: `{ "type": "scene", "pitch", "yaw", "text", "sceneId", "targetYaw", "targetPitch" }`.
  `pitch`/`yaw` place the hotspot in the current room; `targetYaw`/`targetPitch` set where the viewer looks after arriving.

Hotspots that point to a `sceneId` that does not exist are dropped (a warning is logged in the console) so one typo does not break the tour.

## 5. Troubleshooting

- **"Could not load the tour file"**: wrong URL or missing CORS header on the JSON (see section 2).
- **Panorama stays black or shows a WebGL error**: the image has no CORS header, or it is larger than the device's maximum texture size. Keep images at **4096×2048 or smaller** for broad mobile support; use 2:1 equirectangular JPEGs under ~3 MB.
- **Old tour still shows**: GitHub Pages and CDNs cache files. Add a version to the URL (`villa_tour.json?v=2`) or purge the CDN. If you do this in `/tour`, the whole URL is encoded automatically.
- **Button does nothing in a group**: Telegram only allows `web_app` inline buttons in private chats; the bot replies with a notice instead.
- **Very long config URLs**: the final Mini App URL must stay under about 2,000 characters; the bot rejects longer ones.
- **Local testing**: expose a local server through an HTTPS tunnel (ngrok or cloudflared) and use that as `WEB_APP_BASE_URL`.
