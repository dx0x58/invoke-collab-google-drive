# InvokeAI on Google Colab (persistent)

A single Google Colab notebook ([`invokeai_colab.ipynb`](invokeai_colab.ipynb)) that runs
the latest [InvokeAI](https://github.com/invoke-ai/InvokeAI) web UI on a Colab GPU with
**persistent state on Google Drive** — models, generated images, the database and config
survive runtime restarts.

One cell to run: **Start InvokeAI** (a Colab form — only the description and input fields
are shown, the code is collapsed). Stop by killing the Colab runtime.

## Design

| Layer | Location | Persists? | Why |
|-------|----------|-----------|-----|
| InvokeAI package | Colab local disk | No | Fast install (~minutes) each session; reinstalled on Start |
| `INVOKEAI_ROOT` (models, `outputs`, `databases/invokeai.db`, `invokeai.yaml`) | Google Drive (`/content/drive/MyDrive/invokeai`) | Yes | The expensive/irreplaceable data; lives on Drive |

The package is installed locally (not on Drive) so heavy Python I/O stays fast, while only
the data that must survive is kept on the Drive FUSE mount. The server runs in the
background, so Start finishes and Stop can be run as a separate cell whenever you are done.

## Usage

1. Open `invokeai_colab.ipynb` in Google Colab.
2. `Runtime -> Change runtime type -> A100 GPU` (Colab Pro/Pro+ for A100/L4; T4 on free tier).
3. Fill the **Start InvokeAI** form (Drive location, login) and run it: authorizes Drive,
   installs `invokeai[cuda]` (cu128 torch wheels), launches InvokeAI in the background and
   prints a public `https://*.trycloudflare.com` URL plus the login. Open it once up.
4. To stop, kill the runtime: `Runtime -> Disconnect and delete runtime`. Data stays on
   Drive; run the cell again next time to resume.

## Drive location

By default data goes to `My Drive/invokeai`. Change it with the Start form fields
`DRIVE_NAME` (`My Drive` for your personal drive, or the exact name of a Shared Drive) and
`INVOKEAI_FOLDER`.

## Downloading models

No dedicated step — InvokeAI downloads models itself. In the running app open
**Model Manager** and paste a HuggingFace / CivitAI / direct URL; the file lands in the
Drive root and is reused on every later session. For CivitAI, set your API key in the
Model Manager settings first.

## Password-protecting the link

Set `AUTH_USER` / `AUTH_PASSWORD` in the Start form. A [Caddy](https://caddyserver.com)
reverse proxy then enforces HTTP Basic Auth on port `9091` and cloudflared tunnels that
port instead of InvokeAI directly, so the public `trycloudflare.com` link prompts for
credentials (websockets included). Leave `AUTH_PASSWORD` empty for an unprotected link.

## GPU choice

Best throughput and the only Colab GPU that fits large models (Flux.1/Flux.2, SD 3.5 Large,
Qwen Image) without heavy offloading: **A100 (40 GB)**. Fallback: L4 (24 GB) > T4 (16 GB).

## Notes

- **protobuf**: Start upgrades protobuf to `>=5.26` — Colab ships an older build that
  crashes diffusers (`cannot import name 'runtime_version'`), which otherwise leaves the
  server down and the tunnel returning 502 Bad Gateway.
- **Tunnel URL** changes every launch (cloudflare quick tunnels are ephemeral). For a stable
  URL, switch to an authenticated ngrok / cloudflare named tunnel.
- **SQLite on Drive**: fine for a single user. On a rare "database is locked", run Stop, wait
  for Drive to sync, then Start again. Do not open the same root from two runtimes.
- **Drive space**: large models consume tens of GB of Drive quota.

Reference: InvokeAI install docs — https://invoke.ai/start-here/installation/
