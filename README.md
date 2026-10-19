# code-server — VS Code in the browser (v4.141.0, Code 1.141.0, official image, persistent home, generated password). Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/TEMPLATE_CODE)

[code-server](https://github.com/coder/code-server) by [Coder](https://coder.com) (MIT) runs VS Code on a server and serves it to any browser: the full editor, integrated terminal, git, debugging and extensions from [Open VSX](https://open-vsx.org). This template runs the **official** `codercom/code-server` image at **4.141.0** (VS Code 1.141.0, released 2026-10-08), pinned by tag and digest, as a single Railway service with one volume. It needs **no deploy-time input**: open the URL, paste the generated password, and start coding.

## Why this template
- **Current, pinned release.** `4.141.0` is the latest stable code-server release, pinned by tag **and** digest (`sha256:72ab24b5…dbb8`, multi-arch index). A redeploy never jumps to a different editor version behind your back.
- **Official image, no custom build.** Plain `codercom/code-server` from Docker Hub, so it stays easy to audit and to upgrade (change one tag and digest).
- **Everything persists.** The volume is the whole home directory (`/home/coder`): your projects, `settings.json`, keybindings, installed extensions, `config.yaml`, shell history, `~/.ssh`, `~/.gitconfig` and language toolchains you install under your home. Redeploys and restarts keep all of it.
- **Volume permissions handled properly.** Railway volumes mount root-owned while the image runs as `coder` (uid 1000). The start command runs as root only long enough to hand the empty volume to `coder` on first boot (and seed `.bashrc`/`.profile` from `/etc/skel`), then drops to `coder` with `runuser`. The editor, terminals and extensions never run as root.
- **Secure generated password.** `PASSWORD` is a 32-character random secret generated per deploy. code-server drops it from its environment at start, so it is not passed on to terminal or extension processes. Login is rate-limited by code-server.
- **Healthcheck.** Railway waits for the unauthenticated `/healthz` endpoint before routing traffic.
- **Opens straight into a workspace.** The editor opens `/home/coder/workspace` (created on first boot) instead of an empty window.
- **Workbench state on the server.** Uses 4.141.0's new `--enable-remote-storage`, so open editors, layout and UI state are stored on the volume and follow you across browsers and devices.
- **Persistent startup hooks.** `ENTRYPOINTD=/home/coder/.entrypoint.d`: drop executable scripts there (for example `sudo apt-get install -y python3-venv`) and they run on every boot. System packages live outside the volume, so this is how to keep them across redeploys.
- **Telemetry off** (`--disable-telemetry`). The built-in port proxy stays on: an app you run on port 3000 in the terminal is reachable at `https://<your-domain>/proxy/3000/` behind the same login.

## Service
| Service | Source | Port | Volume | Notes |
|---|---|---|---|---|
| code-server | `codercom/code-server:4.141.0@sha256:72ab24b5680ee597dd647b30d82625752f40109b5f61bef90ffbf1ea1f02dbb8` | `8080` (public) | `/home/coder` | Health `/healthz` (timeout 120 s), restart `ALWAYS`, start command below |

### Start command
Railway replaces the image ENTRYPOINT when a start command is set, so the command calls upstream's `/usr/bin/entrypoint.sh` itself (which still runs fixuid, the `ENTRYPOINTD` hooks and `dumb-init`). It is also in `railway/start-command.txt`.
```
/bin/sh -c 'set -e; W="${DEFAULT_WORKSPACE:-/home/coder/workspace}"; if [ "$(stat -c %u /home/coder)" != 1000 ]; then cp -an /etc/skel/. /home/coder/; chown -R 1000:1000 /home/coder; echo "volume: /home/coder handed to coder (uid 1000)"; fi; runuser -u coder -- mkdir -p "$W" "${ENTRYPOINTD:-/home/coder/.entrypoint.d}"; exec runuser -u coder -- /usr/bin/entrypoint.sh --bind-addr 0.0.0.0:8080 --disable-telemetry --enable-remote-storage "$W"'
```
The `chown -R` runs only when the volume root is not yet owned by uid 1000 (first boot), so large homes don't slow down later restarts.

## Variables
Every value is also in `env.example` (and `railway/template-variables.env`).

| Variable | Template value | Notes |
|---|---|---|
| PASSWORD | `${{secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | login password, generated per deploy. Read it in the service's Variables tab |
| PORT | `8080` | code-server listens here (`$PORT` overrides `--bind-addr`); the public domain routes to it |
| DEFAULT_WORKSPACE | `/home/coder/workspace` | folder the editor opens by default. Keep it under `/home/coder` so it is on the volume |
| ENTRYPOINTD | `/home/coder/.entrypoint.d` | executable scripts here run (as `coder`) on every boot |
| RAILWAY_RUN_UID | `0` | start as root so the start command can fix the root-owned volume, then drop to `coder` |

### Optional (add after deploy)
| Variable | Example | What it does |
|---|---|---|
| HASHED_PASSWORD | `$argon2i$v=19$m=4096,t=3,p=1$…` | argon2 hash (`echo -n "pass" \| npx argon2-cli -e`); wins over `PASSWORD`, then delete `PASSWORD` |
| GITHUB_TOKEN | `ghp_…` | pre-authenticates GitHub for the editor |
| CODE_SERVER_IDLE_TIMEOUT_SECONDS | `3600` | exits after N idle seconds |
| CS_DISABLE_FILE_DOWNLOADS | `true` | blocks downloading files from the editor |
| CS_DISABLE_PROXY | `true` | turns off `/proxy/<port>/` |
| TZ | `Asia/Kolkata` | terminal timezone |

## How to use
1. Click **Deploy on Railway** and wait for the service to turn healthy (about 30 seconds).
2. Open the service's **Variables** tab and copy `PASSWORD`.
3. Open the public URL, paste the password, and you land in `/home/coder/workspace`.
4. Clone a repo from the terminal (`git clone …`) or the Source Control view, install extensions from the Extensions view, and work as usual.

To change the password, edit `PASSWORD` and redeploy. To upgrade, change the image tag and digest in the service settings to a newer release; your home directory stays.

## Notes
- **Who can get in.** Anyone with the password gets a full terminal, and `coder` has passwordless `sudo` (upstream default), so treat the password like an SSH key. This is a single-user dev box, not a multi-tenant one.
- **What persists.** Only `/home/coder`. Packages installed with `apt` go to the container filesystem and disappear on redeploy; put the install commands in a script in `~/.entrypoint.d/` (and `chmod +x` it), or install toolchains into your home (nvm, pyenv, rustup, `pip --user`, Go in `~/go`).
- **Extensions** come from Open VSX (Microsoft's marketplace is licensed for Microsoft products only). Most popular extensions are there; others can be installed from a `.vsix` file.
- **Memory.** VS Code plus language servers is comfortable at 1–2 GB; large TypeScript or Java projects want more. Builds and dev servers you run share the same container.
- **Custom domain.** Add it in the service's Settings → Networking. No variable changes needed. code-server checks WebSocket `Origin` against `Host`, which Railway preserves.
- **Upstream docs:** [FAQ](https://github.com/coder/code-server/blob/v4.141.0/docs/FAQ.md), [guide](https://github.com/coder/code-server/blob/v4.141.0/docs/guide.md), [release notes](https://github.com/coder/code-server/releases/tag/v4.141.0).

## License
code-server is MIT-licensed by Coder Technologies. This repository only holds Railway configuration.
