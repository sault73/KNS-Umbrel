# KUNAS/Labs

NVIDIA-only Umbrel package for [KUNAS/Labs](https://github.com/9vibes/SteamLab):
OBS monitoring, authenticated live playback, MP4 recording,
and opt-in face grouping. App ID: `kunas-steamlab`. Version: `1.2.2`.

Version 1.2.2 places **Signal Directory** immediately after **Faces** in each
stream's tool panel. The tab order is Faces, Signal Directory, Recordings, Settings.
The sidebar and drawer are removed. Panels keep independent management targets
while sharing one directory and management forms. Archived-history selection opens
Single view. Tab changes preserve players, recording, and analysis.

Previously named SteamLab NVIDIA. Update the existing app; do not uninstall it.
Version 1.2.0 adds Multi-view and default-on automatic recording, retaining 1.1.0's
four independent streams and the tested CUDA and RTSP fixes from 1.0.3. Installation
identifiers, data, credentials, proxy boundaries, and ports are unchanged.

The authoritative implementation contract is
[MULTIVIEW.md](https://github.com/9vibes/SteamLab/blob/main/docs/MULTIVIEW.md).

**1.2.0 upgrade warning: `AUTO_RECORD` defaults to `true`.** Already-live feeds
automatically record after an update or backend restart, **even if you previously
stopped recording manually**. For manual-only operation, stop encoders before
updating, configure the operator setting `AUTO_RECORD=false` in deployment
configuration (Umbrel environment settings where supported), and apply it before
reconnecting encoders. Settings displays the policy but cannot change it; Stop is
not a persistent opt-out. Back up the stopped app and re-enable analysis afterward
only if wanted.

## Multistream

Four-stream support was introduced in 1.1.0. Deploy the **complete 1.2.2 update** with matching
backend, frontend, worker, and MediaMTX configuration
atomically, including this package's MediaMTX template. Do not combine the new
configuration with older images. No extra ingest port, container, or app ID change:
all feeds use the existing `kunas-steamlab` installation and RTMP port 21935.

- A maximum of four active feeds share one administrator. The original feed becomes
  **Stream 1** (`stream`, path `live/stream`) with its existing publishing key intact.
  Additional feeds use unique `stream-<32 lowercase UUID hex digits>` IDs, individual
  keys, and `live/{id}` paths on the same server URL. Select the feed before copying
  its complete `<id>?user=publisher&pass=...` OBS key.
- Add and rename feeds; names are trimmed, 1-64 characters, with no control
  characters. Browser selection does not change server activity. Analysis toggles,
  recordings, bitrate monitoring, sessions, and face groups are independent per feed.
- Archive only additional feeds that are offline and not recording, while MediaMTX
  is reachable and confirms no pending publisher. Stream 1 cannot be archived.
  Archive retains face/session/recording history and frees an active slot; archived
  feeds cannot ingest/analyze, IDs are never reused, and no restore endpoint exists.
  Archived history can still be read/deleted and the feed renamed.
- `MAX_FACES` is a shared total across all feeds, including archived face groups;
  `MIN_FREE_GB` is one shared reserve for recording and analysis. One initialized
  NVIDIA engine fairly round-robins bounded latest-frame slots from up to four
  independent FFmpeg decoders. Achieved per-feed FPS depends on hardware and load;
  the configured capture rate is a target, not guaranteed analysis throughput.
- The 1.1.0 migration adds stream ownership to sessions, faces, and recordings and backfills
  legacy rows to Stream 1. Existing row IDs, files, and the publishing key remain
  intact; files are not moved or overwritten. Default settings keep their original
  keys; additional settings use `stream:{id}:{key}`. Back up the stopped app first.

Original scoped REST calls default to Stream 1. The player uses directory-scoped
`/api/streams/{id}/live/index.m3u8` for relative playlists/segments, while
`/api/live/{file}` remains a default-stream alias. The worker retains tested
CUDA 12.4.1/cuDNN 9.1, strict CUDA warmup/provider checks, and FFmpeg 4.4 RTSP option
detection. See the upstream [API contract](https://github.com/9vibes/SteamLab/blob/main/docs/API.md)
and [verification results](https://github.com/9vibes/SteamLab/blob/main/docs/VERIFICATION.md).

Back up the complete stopped app before updating. Upgrades from before 1.1.0 apply
its schema migration; rolling back across that migration requires a matching
pre-upgrade data backup rather than older container images alone. Version 1.2.0
retains the 1.1.0 schema and introduces no new database format or API migration.
This package updates all required components together.

## Multi-view (1.2.0)

Choose Single view or Multi-view in the workspace. Multi-view includes only
connected (`online`), non-archived streams, up to four in two desktop columns
(2x2 with four feeds) or one mobile column. Each named tile has recording controls
and telemetry plus its own independent Faces, Recordings, and Settings tabs below
the video, never beside it. Players initially stay muted.

The directory and add/rename/archive management remain outside the grid. Directory
selection targets management without filtering live tiles. Offline-feed settings
and history remain accessible in Single view; selecting archived history switches
to Single view. Empty Multi-view offers connection guidance and a Single view
switch. View/tab changes never start/stop recording or analysis or affect server
feeds; each tile's controls and requests remain stream-scoped.

## Automatic Recording (1.2.0)

`AUTO_RECORD=true` is the default in backend `Config`, source/Umbrel Compose,
`.env.example`, and setup-generated configuration. Set `AUTO_RECORD=false` in
deployment configuration (Umbrel environment settings where supported) for
manual-only recording. Settings shows the read-only policy, not a writable toggle.

Each backend monitor starts one recording when its configured feed is confirmed
ready, including existing live feeds after backend startup/restart. No browser
needs to be open. Automatic and explicit Start use the same H.264, disk, and reader
authentication guards. FFmpeg copies the stream into fragmented MP4 without
transcoding; automatic recording on reconnect creates a new file rather than
appending to the old one.

- Explicit Stop suppresses automatic recording for the current publisher
  fingerprint, even while disk-paused without an active recorder. The latch
  survives temporary MediaMTX API outages and clears for a genuinely different
  publisher connection. It is in-memory, not a persistent opt-out across backend
  restarts; use `AUTO_RECORD=false` for that policy.
- Disconnect finalizes the old recording; a genuine publisher reconnect automatically
  starts a new file unless `AUTO_RECORD=false` requires explicit Start.
  Explicit Start overrides Stop/failure suppression and retries/resumes
  subject to the guards. Start returns an already active recording with HTTP 200
  and does not create duplicates; Stop remains idempotent (204).
- Low disk pauses recording. Automatic recovery requires free space above the
  shared reserve plus headroom for five continuous seconds, even across publisher
  reconnects, unless manually stopped. Headroom is 10% of the reserve, bounded to
  16-256 MiB.
- Spawn failures or unexpected recorder exits are latched for the same connection,
  surfaced as `recording_state=error` with a sanitized `recording_error`. There is
  no repeated automatic attempt/file churn; use Start or reconnect to retry.

Status adds `auto_record: boolean`, a sanitized `recording_error: string | null`,
`can_stop_recording: boolean`, and
`recording_state`: `recording`, `waiting`, `stopped`, `disk_paused`, `error`, `manual`,
or `archived`. Settings adds read-only `auto_record: boolean`. Routes, scoping,
and existing recording metadata remain unchanged; Start is now idempotent.
`can_stop_recording` keeps Stop available in Single view during a media API outage
when the backend remembers a publisher that can be suppressed.
Video auto-recording is intentional and never enables face analysis or identity
recognition. Face analysis remains a separate, explicit opt-in after restart.
Local 1.2.0 verification recorded passing results for 291 backend/worker/registry-verifier
tests, 33 Chromium browser tests, and 21 real-media checks. See the upstream
[verification document](https://github.com/9vibes/SteamLab/blob/main/docs/VERIFICATION.md),
separate from historical 1.0.0/1.1.0 results. Release CI will run; these local checks
do not establish 1.2.0 image publication, anonymous image verification, or installed
Umbrel/NVIDIA behavior.

## Requirements

- Linux x86-64 (`linux/amd64`) with an NVIDIA GPU.
- An NVIDIA driver compatible with CUDA 12.4 and NVIDIA Container Toolkit configured for Docker's `nvidia` runtime.
- One GPU is reserved for the worker. CUDA embedding inference must initialize successfully; there is no CPU fallback. Face detection itself runs on CPU.
- ARM devices, including Umbrel Home ARM and Raspberry Pi, are not supported by this package.
- Sufficient storage for recordings and sensitive face data. The default 2 GiB free-space reserve is a safety guard, not a storage quota.

## First Launch

1. Add `https://github.com/9vibes/KNS-Umbrel` as a community app store in Umbrel and install **KUNAS/Labs**.
2. Open the app from Umbrel on web port **28081**. Log in with the generated application password shown by Umbrel; no username is required.
3. In Settings, copy the server URL and **complete stream key** into OBS. The server URL is `rtmp://<device-hostname>:21935/live`. Preserve the entire `stream?user=publisher&pass=...` key, not just `stream`.
4. Configure OBS for **H.264 video, AAC audio, and a 1-second keyframe interval**. Video is not transcoded.
5. Start publishing. If face analysis is wanted and consent has been obtained, enable it in the Faces tab and confirm the reported provider is `CUDAExecutionProvider`.
6. Publishing automatically records by default in 1.2.0; use Stop/Start for the current connection or configure `AUTO_RECORD=false` for manual-only operation as described above.

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
published directly by this Compose package. KUNAS/Labs handles login itself, so
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
needed. KUNAS/Labs independently generates its publishing key and persists it in
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
backend options. Applying settings may restart the app, disabling analysis.
With the 1.2.0 default `AUTO_RECORD=true`, this also automatically records existing
live feeds, even if previously stopped manually. Use `AUTO_RECORD=false` for a
persistent manual-only policy.

| Setting | Default | Purpose |
| --- | --- | --- |
| `AUTO_RECORD` | `true` | Backend policy: automatically record ready feeds, including already-live feeds after update/restart; `false` for manual-only |
| `PUBLIC_HOST` | Device `.local` hostname, otherwise `umbrel.local` | Advertised OBS hostname or IPv4; not a bind address |
| `COOKIE_SECURE` | `false` | Use `true` only with HTTPS |
| `MIN_FREE_GB` | `2` | Free-space reserve in GiB; 0.05 to 1000000 |
| `FACE_RETENTION_DAYS` | `7` | Face-data retention; 1 to 365 days, not video retention |
| `MAX_FACES` | `2000` | Application-wide stored face-group limit, including archived feeds; 1 to 10000 |
| `MATCH_THRESHOLD` | `0.5` | Cosine similarity threshold; greater than 0 and at most 1 |
| `DETECTION_THRESHOLD` | `0.85` | Detection confidence; greater than 0 and at most 1 |
| `ANALYSIS_FPS` | `2` | Per-feed capture target; 0.2 to 10 frames per second, achieved analysis FPS hardware-dependent |

Low disk space stops recording and pauses analysis. Analysis can resume when
space recovers if already opted in. Automatic recording resumes only
after the stable recovery threshold above and never overrides manual Stop;
manual-only mode requires Start. Four simultaneous recordings grow
storage at the combined rate of all feeds, even without a viewer. The shared
reserve is a safety guard, not a quota or a separate allowance for each feed.
Monitor actual capacity and plan separate storage/quota management. Recordings
are never automatically deleted, even on low disk; export completed files to
separate archival storage or delete them deliberately. Face retention does not
apply to video. Back up the entire app-data directory while the app is stopped to
keep SQLite and files consistent, and protect backups as sensitive data.

## Packaging

The [1.2.2 release workflow](https://github.com/9vibes/SteamLab/actions/runs/34806760792)
passed on Python 3.10 and 3.12, including native model checks and 45 browser tests,
and built all image variants. Every layer of the web, backend, and CUDA worker
images was downloaded anonymously and SHA256-verified before publication.

- Custom amd64 images: `ghcr.io/9vibes/steamlab-web:1.2.2`, `ghcr.io/9vibes/steamlab-backend:1.2.2` (also used for initialization), and `ghcr.io/9vibes/steamlab-worker:1.2.2-cuda`.
- Media server: `bluenviron/mediamtx:1.12.3`.
- nginx configuration is included in the web image; no host nginx configuration is required.
- Umbrel generates `${APP_DATA_DIR}/mediamtx.yml` from `mediamtx.yml.template`, which is retained by Umbrel's app-update whitelist. The generated configuration is mounted read-only.
- Images are pinned to immutable release digests. Anonymous pulls and every layer checksum were verified for all three custom images before publication. A public source repository alone does not make GHCR packages public.
- The icon and gallery screenshots are publicly available in `9vibes/SteamLab`. Screenshots use synthetic API fixtures and contain no personal footage or real credentials.
- Release CI covers backend/worker tests on Python 3.10 and 3.12 (including native CPU model parity), browser tests, and all image builds. Compose configuration and anonymous registry access are verified before publication. Installation on an actual Umbrel/NVIDIA host and actual CUDA inference still require a host smoke test.

See the upstream [Umbrel guide](https://github.com/9vibes/SteamLab/blob/main/docs/UMBREL.md)
for deployment details. CPU Compose support is available upstream for non-Umbrel
installations, not as a fallback in this package.
