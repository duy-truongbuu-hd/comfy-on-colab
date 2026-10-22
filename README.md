# ComfyUI on Google Colab

Run ComfyUI in Google Colab with optional Google Drive persistence, one-click model downloads, and access through Colab's built-in port proxy. All configuration comes from environment variables (`.env` file or Colab secrets).

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/fluxvane/comfy-on-colab/blob/main/ComfyUIonColab.ipynb)

---

## Features
- Mount or unmount Google Drive to persist your ComfyUI workspace
- Persistent home: `$HOME` (configs, `~/.local/bin` tools, SSH keys, shell history) is restored from encrypted restic snapshots on Drive at every fresh VM and backed up automatically
- Env-driven configuration: `.env` file and/or Colab secrets, no values hard-coded in the notebook
- Clone and optionally update the upstream ComfyUI repository
- Parallel, resumable model downloads with aria2 (Hugging Face and Civitai)
- Start ComfyUI and open it through Colab's port proxy
- Optional developer tools: GitHub CLI and omp

## Quick start
1) Click the Colab badge above to open `ComfyUIonColab.ipynb`.
2) Run the Drive cell with `MODE = "MOUNT"` (recommended for persistence).
3) Provide configuration (see [Configuration](#configuration)): copy `.env.example` to `/content/drive/MyDrive/.env` (or `/content/.env`) and fill it in, and/or add the keys as Colab secrets.
4) Run **Load configuration**, then **Persistent home** (restores your home, starts auto-backup), then **Setup / update ComfyUI**.
5) Run **Download models** to fetch the groups listed in `MODEL_GROUPS`.
6) Run **Start ComfyUI** and click the **Open ComfyUI** link it prints once the server is up.

## Configuration
The **Load configuration** cell defines `env(key, default, required)` used by every other cell.

Lookup order: process env → `.env` file → Colab secrets → default. The `.env` file is the first that exists of `$ENV_FILE`, `/content/.env`, `/content/drive/MyDrive/.env`. Re-running the cell picks up edits to the file. Secrets are read on demand and never copied into `os.environ`, so ComfyUI and custom nodes do not inherit them.

| Key | Default | Used by |
|---|---|---|
| `DRIVE_PATH` | empty → `/content/ComfyUI` | Workspace becomes `<DRIVE_PATH>/ComfyUI` |
| `COMFY_UPDATE` | `true` | `git pull --ff-only` on setup |
| `COMFY_PORT` | `8188` | ComfyUI listen port |
| `MODEL_GROUPS` | all groups except `qwen3.5` | Model download |
| `ARIA2_JOBS` | `4` | Concurrent model downloads |
| `CIVITAI_API_TOKEN` | — | Appended as `?token=` to Civitai URLs |
| `RESTIC_PASSWORD` | required for persistent home | Encrypts the snapshot repository |
| `PERSIST_DIR` | `/content/drive/MyDrive/colab-persist` | Repository location (`<PERSIST_DIR>/restic`) |
| `PERSIST_PATHS` | `~` (`/root`) | Paths backed up and restored |
| `PERSIST_EXCLUDES` | caches, DriveFS, Jupyter runtime | `restic --exclude` patterns |
| `PERSIST_INTERVAL_MIN`, `PERSIST_KEEP` | `10`, `10` | Backup interval, snapshots kept |
| `PERSIST_APT_PACKAGES` | — | System packages reinstalled on each fresh VM |
| `RESTIC_VERSION` | `0.19.1` | restic binary downloaded when missing |

Tokens and passwords are credentials: keep them in Colab secrets or a `.env` you never commit (`.gitignore` excludes `.env`).

## Notebook structure
- **Mount/Unmount Google Drive** — `MODE = "MOUNT" | "UNMOUNT"` mounts to `/content/drive` and cleans up when unmounting
- **Configuration (ENV)** — loads `.env`, defines `env()`/`env_bool()`, resolves `WORKSPACE` and `COMFY_PORT`
- **Persistent home (Drive)**
  - On a fresh VM (no `/tmp/.persist-restored` marker) restores the newest snapshot to `/`, falling back to older ones if it fails (e.g. a snapshot whose Drive upload never finished); re-running in the same VM skips the restore
  - Installs `PERSIST_APT_PACKAGES`, then a background thread runs `restic backup --skip-if-unchanged` every `PERSIST_INTERVAL_MIN` minutes, then `forget --keep-last PERSIST_KEEP` (with `--prune` once per session). Output goes to `/content/persist.log`
  - Snapshots use the fixed host `colab` so each VM finds the previous parent snapshot and only reads changed files; permissions, executable bits, and symlinks are restored (a plain copy onto the Drive mount does not reliably keep them)
  - **Save home to Drive now** takes a snapshot immediately: Colab has no shutdown hook, so changes made after the last automatic backup are lost if the runtime dies
- **Setup / update ComfyUI**
  - Clones `https://github.com/comfyanonymous/ComfyUI` into the workspace if missing, otherwise pulls when `COMFY_UPDATE=true`
  - `pip install -r requirements.txt` on top of Colab's preinstalled CUDA torch
  - Commented `git clone` lines for ComfyUI Manager / Impact Pack — uncomment to enable
- **Download models**
  - One manifest lists every model this setup uses (mirrors `ComfyUI-Shared/models`) as verified direct Hugging Face links from `Comfy-Org/*` repacks + `lightx2v` — no auth needed
  - Groups: `qwen-image`, `trellis`, `pixal3d`, `3d-aux`, `acestep`, `minimax-music`, `stable-audio`, `qwen3.5` (~76 GB if all enabled, so prefer `DRIVE_PATH` persistence)
  - Pending files are fetched in one aria2 run (`-j ARIA2_JOBS`, 16 connections per file, `--file-allocation=none` to avoid double writes on Drive); complete files are skipped, unfinished ones (with a `.aria2` control file) resume
  - `downloadModel(url, destdir, filename=None)` works for any ad-hoc Hugging Face or Civitai link; Civitai file names come from the server when `filename` is omitted
  - Files land in `models/<subdir>` (`diffusion_models`, `text_encoders`, `vae`, `loras`, `clip_vision`, `checkpoints`, `background_removal`, `geometry_estimation`)
- **Start ComfyUI** — `python main.py --dont-print-server --port $COMFY_PORT`; a background thread waits for the port, then prints a link via `google.colab.output.serve_kernel_port_as_window`
- **Developer tools (optional)**
  - Installs GitHub CLI (`gh`, latest release) and omp into `~/.local/bin`, skipped if present, so the persistent home keeps them

## Persistence tips
- Recommended `DRIVE_PATH`: `/content/drive/MyDrive`
- With Drive mounted and `DRIVE_PATH` set, your ComfyUI install, models, and custom nodes live in Drive and survive runtime restarts
- Without Drive, everything under `/content` is temporary and will be lost when the Colab session ends
- Anything under `PERSIST_PATHS` persists via the **Persistent home** cell: tool configs (`~/.config/gh`, omp, `~/.config/blender`), binaries installed to `~/.local/bin`, `~/.ssh`, dotfiles. Software installed system-wide (`apt`, `/usr`, `/opt`) does not: install it under `~` (e.g. a Blender tarball in `~/.local/opt`), add its directory to `PERSIST_PATHS`, or list its packages in `PERSIST_APT_PACKAGES`
- Colab cannot run cells on startup: run the notebook top to bottom (Runtime → Run all) on each new runtime

## Troubleshooting
- `Missing config '<KEY>'` → set the key in `.env` or as a Colab secret, then re-run **Load configuration**
- `NotebookAccessError` → the Colab secret exists but this notebook was not granted access
- No **Open ComfyUI** link → the start cell waits up to 10 minutes for `COMFY_PORT`; check the cell output for ComfyUI startup errors
- Civitai 403 or auth errors → ensure `CIVITAI_API_TOKEN` is set and the link is valid
- Hugging Face gated models → accept the model license on the model page and ensure the direct URL is accessible
- Out-of-memory or slow starts → use a smaller checkpoint, avoid heavy custom nodes, or try a Colab runtime with more VRAM

## Folder layout (within the workspace)
- `./models/<subdir>` – model files (see the download section)
- `./custom_nodes` – optional custom node repos

## License
This repository is licensed under the AGPL-3.0. See [`LICENSE`](./LICENSE).

The underlying ComfyUI project is developed by the ComfyUI authors; please refer to their repository and license for details.

## Acknowledgements
- ComfyUI by comfyanonymous
- Community model authors on Civitai and Hugging Face
