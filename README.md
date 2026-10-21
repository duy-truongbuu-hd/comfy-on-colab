# ComfyUI on Google Colab

Run ComfyUI in Google Colab with optional Google Drive persistence, one-click model downloads, and public access via Cloudflare Tunnel or Localtunnel.

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/nazdridoy/ComfyUI-On-Colab/blob/main/ComfyUIonColab.ipynb)

---

## Features
- Mount or unmount Google Drive to persist your ComfyUI workspace
- Clone and optionally update the upstream ComfyUI repository
- Download models from Civitai or Hugging Face with fast aria2
- Optional custom nodes installation (examples included, commented)
- Start ComfyUI and expose it publicly via:
  - Cloudflare Tunnel (trycloudflare)
  - Localtunnel (fallback)

## Quick start
1) Click the Colab badge above to open `ComfyUIonColab.ipynb`.
2) In the first cell, set `MODE` to `MOUNT` and run to mount Google Drive (recommended for persistence).
3) (Optional) Set `DRIVE_PATH` in the Setup cell, e.g. `/content/drive/MyDrive`. If set, the workspace will be created at `<DRIVE_PATH>/ComfyUI` and will persist across sessions. If left empty, the workspace is `/content/ComfyUI` (ephemeral).
4) Run the Setup cell to clone/update ComfyUI and install dependencies.
5) Use the Models section to download checkpoints and VAEs into `./models/checkpoints` and `./models/vae`.
6) Start ComfyUI:
   - Cloudflare Tunnel cell prints a public URL ending with `trycloudflare.com` once port 8188 is ready
   - Or use the Localtunnel cell as an alternative

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
  - Cloudflare: downloads `cloudflared` `.deb`, starts tunnel, and prints a `trycloudflare.com` URL when port 8188 is ready
  - Localtunnel: global `lt` install and public URL; also prints your endpoint IP for password prompt if required

## Persistence tips
- Recommended `DRIVE_PATH`: `/content/drive/MyDrive`
- With Drive mounted and `DRIVE_PATH` set, your ComfyUI install, models, and custom nodes live in Drive and survive runtime restarts
- Without Drive, everything under `/content` is temporary and will be lost when the Colab session ends

## Troubleshooting
- Cloudflared hangs or prints no URL:
  - Re-run the cell after ComfyUI fully starts, or use the Localtunnel cell as fallback
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

