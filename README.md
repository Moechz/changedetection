# changedetection-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [ChangeDetection.io](https://github.com/dgtlmoon/changedetection.io).

## What this is

A four-file TOS Docker application archive (`changedetectiondocker.tar.gz`) that installs
ChangeDetection.io on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `changedetectiondocker`)
- `changedetectiondocker.lang` — 23-language superset (TOS requires 14)
- `changedetectiondocker.svg` — application icon (2531 bytes, 3 SVG elements)
- `docker-compose.yml` — one service, image `dgtlmoon/changedetection.io:0.70.2`

## How it works

| | |
|---|---|
| Container image | `dgtlmoon/changedetection.io:0.70.2` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8037/` (container port 5000) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/datastore` → `/Volume1/DockerAppData/changedetectiondocker/datastore` |
| Logs | `docker logs changedetectiondocker` (stdout/stderr) |

`/datastore` (mounted from `DockerAppData/changedetectiondocker/datastore`) holds `changedetection.json`, the per-watch folders with the stored snapshots and history, and the app's own settings.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 5000.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **Interpreter healthcheck.** The image ships neither curl nor wget, so the healthcheck uses the bundled Python to fetch the start page.
- **HOME moved to /tmp.** The app writes caches under `$HOME`; pointing `HOME` at the container's `/tmp` keeps the read-only application code untouched.
- **One volume, one port.** Only `/datastore` is persistent and only container port 5000 is published.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `/Volume1/DockerAppData/changedetectiondocker/datastore` | ChangeDetection.io application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/changedetectiondocker.tar.gz + .sha256
scripts/build.sh 0.70.2-2   # packaging iteration bump
scripts/build.sh 0.70.2-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/changedetectiondocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v0.70.2-1`**, assets `changedetectiondocker.tar.gz` +
   `changedetectiondocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `changedetectiondocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `0.70.2-1` → automated validation → review.

## License / branding

ChangeDetection.io is distributed by its upstream authors under **Apache-2.0**. The container image is the official image from Docker Hub (`dgtlmoon/changedetection.io:0.70.2`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

- Browser-rendered watches need noticeably more RAM than plain HTML watches.
- The web UI is plain HTTP; use a TLS reverse proxy before exposing it to the internet.
- Notification channels (SMTP, webhook, ntfy) are configured per watch and are not pre-set by this package.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
