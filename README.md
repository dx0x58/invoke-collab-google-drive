# InvokeAI on Google Colab (persistent)

A single Google Colab notebook ([`invokeai_colab.ipynb`](invokeai_colab.ipynb)) that runs
the latest [InvokeAI](https://github.com/invoke-ai/InvokeAI) web UI on a Colab GPU with
**persistent state on Google Drive** — models, generated images, the database and config
survive runtime restarts.

## Design

| Layer | Location | Persists? | Why |
|-------|----------|-----------|-----|
| InvokeAI package | Colab local disk | No | Fast install (~minutes) each session; reinstalled on launch |
| `INVOKEAI_ROOT` (models, `outputs`, `databases/invokeai.db`, `invokeai.yaml`) | Google Drive (`/content/drive/MyDrive/invokeai`) | Yes | The expensive/irreplaceable data; lives on Drive |

The package is installed locally (not on Drive) so that heavy Python I/O stays fast,
while only the data that must survive is kept on the Drive FUSE mount.

## Usage

1. Open `invokeai_colab.ipynb` in Google Colab.
2. `Runtime -> Change runtime type -> A100 GPU` (Colab Pro/Pro+ for A100/L4; T4 on free tier).
3. Run cells top to bottom:
   - **Step 1** — verify GPU and Python (InvokeAI needs Python 3.11–3.12).
   - **Step 2** — mount Google Drive (authorize when prompted) and set `INVOKEAI_ROOT`.
   - **Step 3** — install latest `invokeai[cuda]` (cu128 torch wheels).
   - **Step 4** — start a cloudflared tunnel and launch `invokeai-web`; open the printed
     `https://*.trycloudflare.com` URL once the server is up.
4. Install models from the UI **Model Manager**. They download into the Drive root once.
5. **Shutdown** — when finished, press Stop on the Step 4 cell, then run the Shutdown
   cell (kills the tunnel, flushes + unmounts Drive). To free the GPU:
   `Runtime -> Disconnect and delete runtime`.

## GPU choice

Best throughput and the only Colab GPU that fits large models (Flux.1/Flux.2, SD 3.5 Large,
Qwen Image) without heavy offloading: **A100 (40 GB)**. Fallback: L4 (24 GB) > T4 (16 GB).

## Notes

- **Tunnel URL** changes every launch (cloudflare quick tunnels are ephemeral). For a stable
  URL, switch to an authenticated ngrok / cloudflare named tunnel.
- **SQLite on Drive**: fine for a single user. On a rare "database is locked" error, stop the
  server, wait for Drive to sync, and restart. Do not open the same root from two runtimes.
- **Drive space**: large models consume tens of GB of Drive quota.

Reference: InvokeAI install docs — https://invoke.ai/start-here/installation/
