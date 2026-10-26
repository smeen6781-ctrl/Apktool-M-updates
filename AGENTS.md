# Repository notes

## What this repository is

It has no application server. It is the update feed consumed by the **Apktool M**
Android client plus the CI that maintains it:

- `updates/update_<lang>.json` — one update manifest per language (`build`, `name`,
  sizes, `loadurl`, `changes`). The client fetches the file for the device locale.
- `files/*.apk` — the release APKs referenced by `name` in those manifests
  (universal build plus `_arm64-v8a`, `_armeabi-v7a`, `_x86`, `_x86_64`).
- `translate.py` — run only by `.github/workflows/translate.yml`: translates
  `updates/update_ru.json` into the other languages via the public Google Translate
  endpoint, commits the results and creates a GitHub release. It needs network access
  and the workflow-provided `GITHUB_TOKEN`; nothing here runs it locally.

Paths matter: the manifests are served from the repository root on
`raw.githubusercontent.com`, so a feed host must expose the same layout
(`/updates/update_ru.json`, `/files/<apk>`).

## Running it here

`docker-compose.base44.yml` serves the feed with nginx on host port 3000, straight
from the bind mount, so edits to a manifest or a new APK appear immediately with no
rebuild or restart.

- `./files` and `./updates` are mounted separately at `/srv/feed/...`, which is the
  document root. Do **not** mount the checkout as the document root: the sandbox
  checkout's root directory is mode `700`, so nginx's worker user cannot traverse it
  (every request 403s). A separate document root also avoids the base image's
  welcome `index.html` shadowing the directory listing.
- `docker/base44-nginx.conf` only adds directory listings; there are no application
  environment variables, no database and no external credentials needed to serve
  the feed.

## Verifying

```sh
docker compose -f docker-compose.base44.yml ps          # web must be healthy
curl -s http://localhost:3000/updates/update_ru.json    # valid JSON manifest
curl -sI "http://localhost:3000/files/Apktool_M_v2.4.0-261014(2026101401).apk"  # 200
```

The APK names contain parentheses — quote them in shell commands and URL-encode
them if a client rejects raw parentheses.

## Releasing (unchanged, CI-only)

Editing `updates/update_ru.json` and pushing triggers the translation workflow, which
regenerates all other languages and publishes a release with the universal and
per-ABI APKs. Keep `updates/update_ru.json` as the single source of truth; the other
`update_<lang>.json` files are generated output.
