# SteamLab NVIDIA

NVIDIA-only Umbrel package for [SteamLab](https://github.com/9vibes/SteamLab):
single-stream OBS monitoring, authenticated live playback, manual MP4 recording,
and opt-in face grouping. App ID: `kunas-steamlab`. Version: `1.0.0`.

## Requirements

- Linux x86-64 (`linux/amd64`) with an NVIDIA GPU.
- An NVIDIA driver compatible with CUDA 12.6 and NVIDIA Container Toolkit configured for Docker's `nvidia` runtime.
- One GPU is reserved for the worker. CUDA embedding inference must initialize successfully; there is no CPU fallback. Face detection itself runs on CPU.
- ARM devices, including Umbrel Home ARM and Raspberry Pi, are not supported by this package.
- Sufficient storage for recordings and sensitive face data. The default 2 GiB free-space reserve is a safety guard, not a storage quota.

## First Launch

1. Add `https://github.com/9vibes/KNS-Umbrel` as a community app store in Umbrel and install **SteamLab NVIDIA**.
2. Open the app from Umbrel on web port **28081**. Log in with the generated application password shown by Umbrel; no username is required.
3. In Settings, copy the server URL and **complete stream key** into OBS. The server URL is `rtmp://<device-hostname>:21935/live`. Preserve the entire `stream?user=publisher&pass=...` key, not just `stream`.
4. Configure OBS for **H.264 video, AAC audio, and a 1-second keyframe interval**. Video is not transcoded.
5. Start publishing. If face analysis is wanted and consent has been obtained, enable it in the Faces tab and confirm the reported provider is `CUDAExecutionProvider`.
6. Start recording manually when needed. Recording never automatically resumes after a restart or interruption.

The advertised host uses Umbrel's `DEVICE_DOMAIN_NAME` (the device's `.local`
hostname), falling back to `umbrel.local`. If OBS cannot resolve it, replace the
hostname in OBS with the server's LAN/VPN IPv4 address, preserving port `21935`
and path `/live`. Supported umbrelOS versions also expose `PUBLIC_HOST` in the
app's environment settings; enter a hostname or IPv4 address without a scheme,
port, path, or credentials.

## Network And Login

**RTMP on TCP 21935 is plaintext**, including video and publishing credentials.
It listens on host interfaces for LAN/VPN encoders. Do not router-forward this
port or expose it to the public Internet. Use a VPN for remote publishing and
HTTPS or a trusted private network for dashboard access. Tor browser access
does not make RTMP available through Tor.

Umbrel's app proxy sends web traffic to `kunas-steamlab_web_1:80`; no web port is
published directly by this Compose package. SteamLab handles login itself, so
the proxy's additional authentication is disabled. Set `COOKIE_SECURE=true` only
when the browser uses HTTPS; leave it `false` for local HTTP.

Only `web` joins both Umbrel's default network and the package-private bridge.
`backend`, `mediamtx`, and `worker` join only the private bridge. Backend HTTP,
RTSP, HLS, and the MediaMTX control API are not host-published. The private bridge
is deliberately not `internal: true`, so published RTMP remains reachable.
Do not attach untrusted containers to it: MediaMTX's API trusts network membership.

Umbrel supplies `APP_PASSWORD` as the administrator password and the separately
derived, app-specific `APP_SEED` as the backend/worker `INTERNAL_TOKEN`. Both are
64-character values. No exports script, hardcoded password, or secret files are
needed. SteamLab independently generates its publishing key and persists it in
SQLite. Do not share expanded Compose output, container environments, stream
keys, or diagnostic logs that might contain credentials.

## Data And Settings

Persistent data lives in `${APP_DATA_DIR}/data`, including SQLite, face thumbnails
and embeddings, and recordings under `data/recordings`. A one-shot root `init`
service runs `python -m backend.init_data` without networking, creates `/data`
and `/data/recordings`, and sets only those directories to UID/GID `65532:65532`
and mode `0700`. It does not recursively change existing files. The backend
starts only after initialization succeeds; backend and worker run as UID/GID
65532. Keep exactly one backend process and replica because it owns recording
and database lifecycle.

Face thumbnails and embeddings are **sensitive biometric data**. Obtain consent,
restrict access, and choose the shortest appropriate retention. Face grouping
does not verify identity. Analysis is off initially and after every backend
restart; enabling it is an explicit opt-in. Deleting face data does not redact
existing videos. Recordings are never automatically deleted.

On supported umbrelOS versions, the app's environment settings expose these
backend options. Applying settings may restart the app, disabling analysis and
requiring any recording to be started again manually.

| Setting | Default | Purpose |
| --- | --- | --- |
| `PUBLIC_HOST` | Device `.local` hostname, otherwise `umbrel.local` | Advertised OBS hostname or IPv4; not a bind address |
| `COOKIE_SECURE` | `false` | Use `true` only with HTTPS |
| `MIN_FREE_GB` | `2` | Free-space reserve in GiB; 0.05 to 1000000 |
| `FACE_RETENTION_DAYS` | `7` | Face-data retention; 1 to 365 days, not video retention |
| `MAX_FACES` | `2000` | Stored face-group limit; 1 to 10000 |
| `MATCH_THRESHOLD` | `0.5` | Cosine similarity threshold; greater than 0 and at most 1 |
| `DETECTION_THRESHOLD` | `0.85` | Detection confidence; greater than 0 and at most 1 |
| `ANALYSIS_FPS` | `2` | Analysis sampling rate; 0.2 to 10 frames per second |

Low disk space stops recording and pauses analysis. Analysis can resume when
space recovers; recording must be started manually. Delete or archive recordings
deliberately. Back up the entire app-data directory while the app is stopped to
keep SQLite and files consistent, and protect backups as sensitive data.

## Packaging

- Custom amd64 images: `ghcr.io/9vibes/steamlab-web:1.0.0`, `ghcr.io/9vibes/steamlab-backend:1.0.0` (also used for initialization), and `ghcr.io/9vibes/steamlab-worker:1.0.0-cuda`.
- Media server: `bluenviron/mediamtx:1.12.3`.
- nginx configuration is included in the web image; no host nginx configuration is required.
- Umbrel generates `${APP_DATA_DIR}/mediamtx.yml` from `mediamtx.yml.template`, which is retained by Umbrel's app-update whitelist. The generated configuration is mounted read-only.
- Images are pinned to immutable release digests. All layers of the three custom images were downloaded anonymously and checksum-verified before publication. A public source repository alone does not make GHCR packages public.
- The icon and gallery screenshots are publicly available in `9vibes/SteamLab`. Screenshots use synthetic API fixtures and contain no personal footage or real credentials.
- The release workflow passed 97 backend/worker tests (including native CPU model parity), 10 browser tests, and all image builds. Docker Compose configuration validation passed. Installation on an actual Umbrel/NVIDIA host and actual CUDA inference still require a host smoke test.

See the upstream [Umbrel guide](https://github.com/9vibes/SteamLab/blob/main/docs/UMBREL.md)
for deployment details. CPU Compose support is available upstream for non-Umbrel
installations, not as a fallback in this package.
