# M2 Ultra 64 GB — Local AI Stack (Ansible)

Ansible-orchestrated reproduction of the
[2GuysTek 3×RTX-3090 AI homelab stack](https://2guystek.tv/homelab/ai/infrastructure/2026/07/31/building-my-ai-homelab-server-three-rtx-3090s)
on a **Mac Studio M2 Ultra / 64 GB**, run locally against this Mac.

## How it differs from the original (and why)

Apple Silicon has no CUDA, and Docker Desktop cannot pass a GPU into containers.
The stack is therefore split:

- **Native, arm64, GPU-accelerated (Metal/MPS):** Ollama (LLM + embeddings) and the
  Infinity reranker.
- **Containers (arm64, CPU-only):** Open WebUI, Qdrant, Tika, SearXNG, Open Terminal, Dozzle.
- **Caddy stays native** (your existing install), extended with one new site block.
  No NVIDIA drivers, no NVIDIA Container Toolkit, no `:cuda` images, no Whisper (v1).

## Layout

```
ansible.cfg                     # inventory, vault password file, interpreter
inventory/hosts.yml             # local connection to this Mac
inventory/group_vars/all/main.yml     # non-secret config
inventory/group_vars/all/vault.yml    # secrets (gitignored; encrypt before sharing)
playbooks/site.yml              # the deployment play
playbooks/preflight.yml         # read-only environment checks
playbooks/verify.yml            # post-deploy functional checks
playbooks/verify-70b.yml        # loads the 70B and confirms it is 100% GPU
roles/{host,ollama,reranker,containers,caddy,validate}/
```

## Prerequisites

1. **Native arm64 Homebrew** at `/opt/homebrew` (present). `/usr/local` is x86_64 here
   and is deliberately never used for installs.
2. **Docker Desktop** running. Optional but recommended: set its VM memory to ~8 GB
   (Settings → Resources), leaving room for the 70B model in unified memory.
3. **A second DuckDNS hostname** (e.g. `beranci-ai.duckdns.org`) pointing at the same
   home IP as your existing `beranci.duckdns.org` record. Update `ai_hostname` in
   `inventory/group_vars/all/main.yml` and re-run if you use a different name. Your `duckdns.sh`
   can be reused with the new token/subdomain.
4. **sudo/become** available for the Metal wired-limit step (`sysctl`) and the
   LaunchDaemon. Run the playbook with `-K` if sudo prompts.

Ollama 0.34.4 (arm64) is already installed and `~/.ollama/models` already holds
`qwen2.5:14b` and `translategemma:12b`; those are reused.

## Usage

```sh
# from the repo root (ansible.cfg is picked up automatically)
ansible-playbook playbooks/site.yml --ask-become-pass

# or, non-interactively:
ansible-playbook playbooks/site.yml --become-password-file ~/.ansible_become_pass

# functional checks after deploy:
ansible-playbook playbooks/verify.yml
ansible-playbook playbooks/verify-70b.yml      # loads ~47 GB; be patient
```

First run pulls tens of GB of models (~43 GB for the 70B alone) and installs
torch for the reranker, so it can take a long time.

### Models

Base models are pulled for use and reference (`llama3.3:70b-instruct-q4_K_M`,
`qwen2.5:32b`, `llama3.1:8b`, `mxbai-embed-large`). In addition the play creates
derived models with a **32k context baked in**:

- `llama3.3-70b-32k`
- `qwen2.5-32b-32k`
- `llama3.1-8b-32k`

**Use these `-32k` variants in Open WebUI.** Ollama 0.34 chooses a context from
available VRAM and otherwise loads the 70B at its 131072 maximum, which overflows
the 56 GB wired limit and spills ~11% of layers to CPU (66 GB, `11%/89% CPU/GPU`).
At 32k the same model is fully resident: **47 GB, 100% GPU**. The derived models
share blobs with the base models, so they add no disk usage.

## Secrets

Everything sensitive lives in `inventory/group_vars/all/vault.yml`, encrypted
(AES256) with `~/.vault_password.txt`. `ansible.cfg` sets `vault_password_file`,
so playbooks decrypt transparently. View or edit it with:

```sh
ansible-vault view inventory/group_vars/all/vault.yml --vault-password-file ~/.vault_password.txt
```

Keys: `webui_secret_key`, `searxng_secret`, `open_terminal_api_key`, and the
Open WebUI admin bootstrap (`webui_admin_name`, `webui_admin_email`,
`webui_admin_password`). The headless admin is only created when the database has
**no users** (fresh install); on an existing instance the env vars are ignored and
you log in with the account you already created. `ENABLE_SIGNUP=false` keeps the
public site locked down.

Rendered `~/ai-box/docker-compose.yml` and `~/ai-box/searxng/settings.yml` contain
secrets and are written mode `0600`.

Caddy basic auth is **disabled** by default (`caddy_basic_auth_enabled: false`) —
see the caveats. The `caddy_basic_auth_*` vault keys are kept so you can re-enable it.

## What gets deployed

| Component | Form | Endpoint |
|---|---|---|
| Ollama | native LaunchAgent `com.otto.ollama` | `0.0.0.0:11434` (Metal) |
| Infinity reranker | native LaunchAgent `com.otto.infinity-reranker` | `127.0.0.1:7997` (MPS) |
| Open WebUI | container | `127.0.0.1:3000` |
| Qdrant | container | internal `:6333` |
| Tika | container | internal `:9998` |
| SearXNG | container | internal `:8080` |
| Open Terminal | container | internal `:8000` |
| Dozzle | container | `127.0.0.1:8088` |
| Caddy | native LaunchDaemon `com.otto.caddy` | `https://<ai_hostname>` → `:3000` (TLS; auth via Open WebUI) |
| Metal wired limit | LaunchDaemon `com.otto.iogpu-wired-limit` | `iogpu.wired_limit_mb=57344` |

Only Open WebUI (via Caddy) and Dozzle-on-localhost are exposed; everything else
stays on the internal Docker network or loopback. Note that `OLLAMA_HOST=0.0.0.0`
makes `:11434` reachable on the LAN — add a `pf` rule or keep the Mac on a trusted
network if that matters.

Caddy needs root to bind `:443`, so it runs as a LaunchDaemon; its `HOME` is set to
your user home so the existing ACME data/certificates in
`~/Library/Application Support/Caddy` are reused. **Docker Desktop owns host port
80** on this machine, so the managed global option `http_port 8080` moves Caddy's
plain-HTTP listener to `:8080`; HTTPS/cert issuance (TLS-ALPN on `:443`) is
unaffected.

## Known caveats / TODO

- Caddy HTTP basic auth is intentionally **off** (`caddy_basic_auth_enabled: false`).
  With Open WebUI's SPA, a cached basic-auth credential does not cover the app's
  XHR/WebSocket/service-worker requests, so Chrome re-prompts on essentially every
  request. Access is protected by Open WebUI's own login with `ENABLE_SIGNUP=false`.
  Set `caddy_basic_auth_enabled: true` to re-enable it, but expect the re-prompting.
- Container image tags are floating (`:main`, `:latest`). Pin explicit versions for
  reproducibility.
- The reranker installs `infinity-emb[torch,server]` with `click<8.2`. The `[all]`
  extra is avoided (its `ctranslate2`/`onnx` deps have no arm64 macOS wheels), and
  `click<8.2` avoids a typer incompatibility. It runs with `--no-bettertransformer`
  because the BetterTransformer path is CUDA-oriented and crashes without `optimum`.
  If the install fails, the play falls back to Open WebUI's built-in reranker.
- `OLLAMA_CONTEXT_LENGTH` is not honored by Ollama 0.34, which picks a VRAM-based
  default instead — hence the derived `-32k` models above.
- `open-terminal` is an admin-level shell. It is not published to the host and sits
  behind Open WebUI auth + Caddy basic auth. Keep it that way.
- On 64 GB, keep the Docker Desktop VM modest (~8 GB) so the 70B has room. Expect
  roughly 10–15 tok/s for 70B and ~20–30 for 32B.

## Rollback

```sh
docker compose -f ~/ai-box/docker-compose.yml down
launchctl bootout gui/$(id -u)/com.otto.ollama
launchctl bootout gui/$(id -u)/com.otto.infinity-reranker
sudo launchctl bootout system/com.otto.caddy
sudo rm /Library/LaunchDaemons/com.otto.caddy.plist
sudo launchctl bootout system/com.otto.iogpu-wired-limit
sudo rm /Library/LaunchDaemons/com.otto.iogpu-wired-limit.plist
# remove the "ANSIBLE MANAGED AI STACK" and "ANSIBLE MANAGED AI STACK GLOBAL"
# blocks from ~/Caddyfile
```
