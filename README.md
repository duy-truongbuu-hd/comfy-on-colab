# ComfyUI on Google Colab

Run ComfyUI in Google Colab with optional Google Drive persistence, one-click model downloads, and public access via a **named Cloudflare Tunnel** (your own hostname — no `trycloudflare.com` quick tunnel, no Localtunnel).

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/fluxvane/comfy-on-colab/blob/main/ComfyUIonColab.ipynb)

---

## Features
- Mount or unmount Google Drive to persist your ComfyUI workspace
- Clone and optionally update the upstream ComfyUI repository
- Download models from Civitai or Hugging Face with fast aria2
- Optional custom nodes installation (examples included, commented)
- Start ComfyUI and expose it publicly via a **named Cloudflare Tunnel** (own hostname)

## Quick start
1) Click the Colab badge above to open `ComfyUIonColab.ipynb`.
2) In the first cell, set `MODE` to `MOUNT` and run to mount Google Drive (recommended for persistence).
3) (Optional) Set `DRIVE_PATH` in the Setup cell, e.g. `/content/drive/MyDrive`. If set, the workspace will be created at `<DRIVE_PATH>/ComfyUI` and will persist across sessions. If left empty, the workspace is `/content/ComfyUI` (ephemeral).
4) Run the Setup cell to clone/update ComfyUI and install dependencies.
5) Use the Models section to download checkpoints and VAEs into `./models/checkpoints` and `./models/vae`.
6) Start ComfyUI:
   - The **Cloudflare Tunnel** cell starts `cloudflared` with your named tunnel token and prints your public URL once port 8188 is ready (see the tunnel setup below)
   - The **manual** section installs `cloudflared` and starts ComfyUI without a tunnel, if you want to attach your own connector

## Cloudflare Tunnel setup (one-time, in your Cloudflare account)
1) Cloudflare Zero Trust → **Networks → Tunnels** → create a tunnel (Cloudflared) named e.g. `comfy-colab`.
2) In the tunnel, add a **Public hostname**:
   - Subdomain/hostname: your choice, e.g. `comfy.example.com`
   - Service: `http://localhost:8188`
3) The tunnel page shows an **Install and run a connector** command containing a token
   (`eyJhIjoi...`). Copy only the token value.
4) In Colab: **Tools → Settings → Secrets**, add a secret named `CLOUDFLARE_TUNNEL_TOKEN` with that token value (grant notebook access when prompted).
5) Run the notebook — the tunnel cell starts the connector with the token and your hostname stays stable across sessions.

The tunnel token is a credential: keep it in Colab secrets (or a `.env` you never commit), never inline it in the notebook.

## Notebook structure (what each section does)
- Mount/Unmount Google Drive
  - `MODE = "MOUNT" | "UNMOUNT"` mounts to `/content/drive` and cleans up when unmounting
- Setup and Update ComfyUI
  - `DRIVE_PATH` (string, optional). When provided, workspace is set to `<DRIVE_PATH>/ComfyUI`
  - Clones `https://github.com/comfyanonymous/ComfyUI` if not present
  - If `UPDATE_COMFY_UI = True`, runs `git pull`
  - Installs dependencies with `pip install xformers!=0.0.18 -r requirements.txt` and CUDA wheels extra-indexes
- Models download helpers
  - Installs `aria2` for fast, segmented downloads
  - `downloadModel(url, filename=None)` supports Civitai and Hugging Face
  - Civitai API token is read from Colab user secrets via:
    ```python
    from google.colab import userdata
    CIVITAI_API_TOKEN = userdata.get('CIVITAI_API_TOKEN')
    ```
    Add a secret named `CIVITAI_API_TOKEN` in Colab (Manage user secrets) if you download private/protected links.
  - Example targets used in the notebook:
    - Checkpoints: `./models/checkpoints`
    - VAEs: `./models/vae`
- Custom nodes (optional)
  - Helper: `install_custom_node(url)` clones into `/content/ComfyUI/custom_nodes`
  - Example lines for ComfyUI Manager/Impact Pack are included but commented out—uncomment to enable
- Start and expose ComfyUI
  - **Install cloudflared**: adds Cloudflare's official APT repo (`/usr/share/keyrings/cloudflare-main.gpg` + `pkg.cloudflare.com/cloudflared`) and installs the `cloudflared` package; `sudo` is used only when the notebook does not run as root
  - **CF Tunnel**: reads the `CLOUDFLARE_TUNNEL_TOKEN` Colab secret, waits for port 8188, then runs `cloudflared tunnel --no-autoupdate run --token <TOKEN>` and prints the public hostname
  - **Manual**: installs `cloudflared` and starts `python main.py --dont-print-server` without a tunnel

## Persistence tips
- Recommended `DRIVE_PATH`: `/content/drive/MyDrive`
- With Drive mounted and `DRIVE_PATH` set, your ComfyUI install, models, and custom nodes live in Drive and survive runtime restarts
- Without Drive, everything under `/content` is temporary and will be lost when the Colab session ends

## Troubleshooting
- Cloudflared hangs or prints no URL:
  - The tunnel cell waits for port 8188 before starting the connector; if ComfyUI is not up yet, wait and re-run
  - `missing secret CLOUDFLARE_TUNNEL_TOKEN` → add the tunnel token to Colab secrets and grant the notebook access
  - `Registered tunnel connection` never appears → check the tunnel status in Cloudflare Zero Trust; the token must belong to the tunnel that owns the public hostname
  - `cloudflared` not found → re-run the **Install cloudflared** cell
  - 502/1033 at your hostname → the public hostname's service must point to `http://localhost:8188`
- Civitai 403 or auth errors:
  - Ensure `CIVITAI_API_TOKEN` is set in Colab user secrets and the link is valid
- Hugging Face gated models:
  - Accept the model license on the model page and ensure the direct URL is accessible
- Out-of-memory or slow starts:
  - Use a smaller checkpoint, avoid heavy custom nodes, or try a Colab runtime with more VRAM

## Folder layout (within the workspace)
- `./models/checkpoints` – base models (e.g., SD1.5, XL, etc.)
- `./models/vae` – VAE files
- `./custom_nodes` – optional custom node repos

## License
This repository is licensed under the AGPL-3.0. See [`LICENSE`](./LICENSE).

The underlying ComfyUI project is developed by the ComfyUI authors; please refer to their repository and license for details.

## Acknowledgements
- ComfyUI by comfyanonymous
- Community model authors on Civitai and Hugging Face

