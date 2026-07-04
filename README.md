# InvokeAI on Google Colab — persistent, on Google Drive

A single Google Colab notebook ([`invokeai_colab.ipynb`](invokeai_colab.ipynb)) that runs
the latest [InvokeAI](https://github.com/invoke-ai/InvokeAI) web UI on a Colab GPU and keeps
**all state on Google Drive**, so your models, generated images, gallery and settings
survive when the runtime stops. Open the notebook, fill a short form, run one cell, and open
the printed link.

> Open in Colab:
> **https://colab.research.google.com/github/dx0x58/invokeai-colab-google-drive/blob/main/invokeai_colab.ipynb**

## Features

- **Persistent on Drive** — models, `outputs`, the database (boards/gallery/workflows) and
  `invokeai.yaml` live under `INVOKEAI_ROOT` on Google Drive and are reused every session.
- **One collapsed form** — code is hidden; you only set a few fields and press Run.
- **Password-protected link** — an HTTP Basic Auth proxy sits in front of the public URL.
- **Latest InvokeAI** — installs the current release; auto-fixes the Colab `protobuf` issue.
- **Waits for readiness** — the link is only useful once the server is actually up, and the
  cell reports it (no more 502 Bad Gateway guesswork).
- **Logs cell** — the server runs in the background; a dedicated cell tails its log.

## How it works

| Layer | Location | Persists? | Why |
|-------|----------|-----------|-----|
| InvokeAI Python package | Colab local disk | No | Fast install each session; reinstalled on Start |
| `INVOKEAI_ROOT` = `/content/drive/MyDrive/invokeai` (models, `outputs`, `databases/invokeai.db`, `invokeai.yaml`) | Google Drive | **Yes** | The expensive, irreplaceable data |

The package is installed to Colab's fast local disk (not Drive) so imports stay fast; only
the data that must survive is kept on the Drive mount. The server and tunnel run in the
background, so the Start cell finishes and prints the URL instead of blocking.

## Quick start

1. Open the notebook in Colab (link above).
2. `Runtime → Change runtime type →` pick a GPU (see **GPU** below).
3. In the **Start InvokeAI** cell, set the fields (defaults are fine) and run it:
   - `DRIVE_NAME` / `INVOKEAI_FOLDER` — where on Drive to store data
     (default `My Drive` / `invokeai` → `/content/drive/MyDrive/invokeai`).
   - `AUTH_USER` / `AUTH_PASSWORD` — login for the public link (default `invoke` / `invoke`;
     empty password = no auth).
   - `USE_XFORMERS` — off by default for a faster launch (the default installs plain
     `invokeai` and reuses Colab's preinstalled torch). Enable it for memory-efficient
     attention (saves VRAM) at the cost of a slower start (it reinstalls torch).
4. Authorize Google Drive when prompted. Wait until the cell prints
   `Open InvokeAI at: https://…trycloudflare.com` and the login, then open it.
5. **To stop:** `Runtime → Disconnect and delete runtime`. Data stays on Drive; run the
   cell again next time to resume.

## GPU

Recommended default: **L4 (24 GB)** — the best balance of availability and capability on
Colab Pro; enough for SDXL and, with offloading, Flux-class models. Step up to
**A100 (40 GB)** only for the largest models (Flux.1/Flux.2, SD 3.5 Large, Qwen Image) at
full size. **T4 (16 GB, free tier)** handles SD1.5/SDXL. The Start cell warns if the runtime
has neither L4 nor A100.

## Downloading models

No extra step — InvokeAI downloads models itself. In the running app open **Model Manager**
and paste a HuggingFace / CivitAI / direct URL; the file lands under the Drive root and is
reused on every later session. For CivitAI, set your API key in the Model Manager settings
first.

## Password-protecting the link

Set `AUTH_USER` / `AUTH_PASSWORD` in the form. A [Caddy](https://caddyserver.com) reverse
proxy then enforces HTTP Basic Auth on port `9091` and cloudflared tunnels that port (so the
public link prompts for credentials, websockets included). Leave the password empty for an
unprotected link. Note: cloudflare quick-tunnel URLs are ephemeral and change every launch.

## Logs

The server runs in the background, so its output goes to `/content/invokeai.log`, not the
Start cell. Run the **View server logs & paths** cell to tail the log and confirm where data
is stored (it prints `INVOKEAI_ROOT`, the newest `outputs/images`, and the `databases` dir).

## Troubleshooting

- **502 Bad Gateway** on the link → the server hasn't finished starting yet, or it crashed.
  Wait, then check the **View server logs & paths** cell.
- **`cannot import name 'runtime_version'`** → Colab's old `protobuf`. The Start cell already
  upgrades it to `>=5.26`; just rerun Start.
- **`invokeai-web: command not found`** (in other notebooks) → the install step didn't finish
  before launch. Let the install cell complete first.
- **Missing model / 404 thumbnails** → the database has records but the files aren't at the
  current path. This happens when you mix roots/notebooks. Always use the **same** root
  (`/content/drive/MyDrive/invokeai`); don't mix with other notebooks that use a different
  path. Delete the dead entry in Model Manager / gallery and re-install/re-generate.

## Notes

- **Keep-alive**: `KEEP_ALIVE` (on by default) sends a light request to the server every
  60 s so the runtime keeps seeing activity. Best-effort — Colab's idle disconnect is mainly
  triggered by the browser tab being inactive, which a notebook can't fully prevent; keep the
  tab open for long unattended runs.
- **Always use one root**: keep `DRIVE_NAME = My Drive` and `INVOKEAI_FOLDER = invokeai`.
  Mixing with other paths (e.g. `MyDrive/InvokeAI`) splits the database from the files.
- **Drive space**: large models (Flux, SD 3.5 Large, Qwen ~40 GB) use a lot of Drive quota.
- **SQLite on Drive**: fine for a single user. On a rare "database is locked", stop, wait a
  few seconds for Drive to sync, and start again. Don't open the same root from two runtimes.
- **Before killing the runtime**: stop generating a few seconds early so the database
  finishes writing to Drive.

Reference: InvokeAI install docs — https://invoke.ai/start-here/installation/
