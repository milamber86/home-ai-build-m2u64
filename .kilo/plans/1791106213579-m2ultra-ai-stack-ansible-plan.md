# M2 Ultra 64 GB — Local AI Stack (2GuysTek 3×3090 analog), Ansible-orchestrated

## Goal

Stand up a local, self-hosted AI stack equivalent to the referenced
[2GuysTek build](https://2guystek.tv/homelab/ai/infrastructure/2026/07/31/building-my-ai-homelab-server-three-rtx-3090s)
on a Mac Studio **M2 Ultra / 64 GB unified memory**, managed entirely by an
**Ansible** playbook run locally. The goal is 60–80% of frontier capability with
local ownership of the floor — not parity with 72 GB of NVIDIA VRAM.

## Verdict

Feasible. The article's stack is CUDA-based; Apple Silicon has no CUDA and
Docker Desktop does not pass a GPU into containers. Therefore the GPU-accelerated
pieces run **natively** (Ollama on Metal, reranker on MPS) and every CPU service
runs in an **arm64** container. Everything else maps 1:1.

---

## Locked decisions

| Decision | Choice |
|---|---|
| GPU workloads | **Native** Ollama (Metal) + **native** Infinity reranker (MPS); Docker for everything else |
| Container runtime | **Docker Desktop** (already installed at `/Applications/Docker.app`) |
| Ansible target | **Local** (`ansible_connection=local`) against this Mac |
| Edge/TLS | **Extend the existing native Caddy** (`/usr/local/bin/caddy`, `~/Caddyfile`) — no Caddy container |
| Public hostname | **A second free DuckDNS name** (e.g. `beranci-ai.duckdns.org`) → real Let's Encrypt cert |
| Models | 70B flagship + 32B + 8B + embedding (details below) |
| Persistent data | **Internal SSD**: `~/ai-box` for config/DB; Ollama keeps `~/.ollama/models` |
| Optional services | Dozzle, Caddy `basic_auth`, Open Terminal. **No Whisper in v1.** |

## Verified environment facts (from planning scan)

- Mac Studio M2 Ultra, **arm64**, macOS 26 (Darwin 25.6), 64 GB RAM.
- Homebrew is **split**: `/usr/local` is **x86_64 (Rosetta)**; `/opt/homebrew` is **native arm64**.
  → All native installs (Ollama, Python) **must use `/opt/homebrew/bin/brew` explicitly.**
- **Ollama 0.34.4 is already installed natively** (`/opt/homebrew/bin/ollama`, arm64) with
  `~/.ollama/models` holding `qwen2.5:14b` + `translategemma:12b` (~16 GB). Reuse them.
- Native Python **3.11.16** and 3.14 at `/opt/homebrew/bin/`.
- Ansible 11.6.0 installed (x86_64 brew); Docker Desktop + Caddy present.
- Internal free space **~199 GB** (`/System/Volumes/Data`). External SMB share
  `//otto@192.168.1.21/data` (3.5 TB) exists but is **not** used for models.
- Existing native Caddy already reverse-proxies `beranci.duckdns.org` → Jellyfin `127.0.0.1:8096`
  and occupies **:80/:443** (why we extend it instead of adding a container).

> Apple Silicon caveat: a containerized Ollama/reranker would get **no GPU**
> (Docker Desktop can't reach Metal) — hence the native/MPS split.

---

## Target architecture

```
                 Internet / LAN
                       │  :443  (Let's Encrypt TLS, basic_auth)
              ┌────────▼─────────┐   NATIVE (x86_64 today, fine for proxying)
              │   Caddy (host)    │  existing config + new AI site block
              └────────┬─────────┘
                       │ 127.0.0.1:3000
   DOCKER (arm64, CPU-only VM)                    NATIVE (arm64, GPU)
   ┌───────────────────────────┐                ┌────────────────────────────┐
   │ open-webui  :3000→8080     │──HTTP────────▶ │ ollama serve :11434 (Metal)│
   │ qdrant      :6333          │  host.docker   │  ~/.ollama/models          │
   │ tika        :9998          │  .internal     ├────────────────────────────┤
   │ searxng     :8080          │───────────────▶│ infinity reranker :7997    │
   │ open-terminal :8000        │                │  (MPS / torch)             │
   │ dozzle      :8080          │                └────────────────────────────┘
   └───────────────────────────┘
```

| Component | Where | Image / form | Notes |
|---|---|---|---|
| Ollama | native arm64 | `/opt/homebrew/bin/ollama` (v0.34.4), launchd | `OLLAMA_HOST=0.0.0.0:11434`, Metal |
| Infinity reranker | native arm64 | venv Python 3.11, `infinity-emb[all]` | `--device mps`, port 7997, `BAAI/bge-reranker-v2-m3` |
| Open WebUI | container | `ghcr.io/open-webui/open-webui:<pinned>` | **arm64 `:main`/pinned version — NOT `:cuda`** |
| Qdrant | container | `qdrant/qdrant:<pinned>` | arm64 |
| Tika | container | `apache/tika:<pinned>-full` | arm64 multi-platform confirmed |
| SearXNG | container | `searxng/searxng:<pinned>` | arm64; `settings.yml` must enable `json` |
| Open Terminal | container | `ghcr.io/open-webui/open-terminal:<pinned>` | verify arm64 at implementation |
| Dozzle | container | `amir20/dozzle:<pinned>` | mounts `/var/run/docker.sock` |
| Caddy | native | existing `/usr/local/bin/caddy` | new site block + `caddy reload` |

---

## Capacity, memory and risk notes

- **70B needs a raised Metal cap.** Default `recommendedMaxWorkingSetSize` ≈ 75% of
  64 GB ≈ 48 GB; a 70B Q4 (~43 GB) + KV cache can exceed it. Set
  `sysctl -w iogpu.wired_limit_mb=57344` (56 GB) and persist via a LaunchDaemon.
- **Memory budget is tight.** Ollama 70B ≈ 43–47 GB + OS/apps ≈ 8–10 GB + Docker
  Desktop VM. Keep the **Docker Desktop VM at 6–8 GB** (containers are CPU-light because
  embeddings run in Ollama). Avoid running 70B while doing heavy container work.
- **Speed expectation:** ~10–15 tok/s for 70B Q4; ~20–30 tok/s for 32B Q4 on M2 Ultra.
- **Context:** the article used `OLLAMA_CONTEXT_LENGTH=131071`. That KV cache is
  infeasible for 70B on 64 GB — default to **32768** (tunable variable).
- **`OLLAMA_KEEP_ALIVE`**: article used `-1` (never unload). On 64 GB recommend
  **`30m`** so a resident 43 GB model frees memory; make it a variable.
- **Infinity install is heavy** (~2–3 GB of torch). If MPS fails, fallback is
  Open WebUI's built-in reranker (`RAG_RERANKING_ENGINE=local`) — keep as a variable.
- **Disk:** chosen models ≈ 60–90 GB, Docker images ≈ 10–15 GB; fits in ~199 GB.
- **Ollama binding:** to let the container reach it, bind `0.0.0.0:11434`. Do **not**
  expose `:11434` publicly; add a `pf`/firewall rule or rely on the LAN-only +
  Docker-gateway bind. Flagged as a security task.

---

## Repository layout to create

```
home-ai-build-m2u64/
  ansible.cfg
  requirements.yml
  inventory/hosts.yml
  playbooks/site.yml
  group_vars/
    all.yml                 # non-secret vars (paths, ports, hostname, image tags, model list)
    vault.yml               # ansible-vault encrypted secrets
  roles/
    host/                   # dirs, wired-limit sysctl+LaunchDaemon, Docker Desktop running, firewall
    ollama/                 # ensure formula, LaunchAgent, pull models
    reranker/               # venv + infinity-emb, LaunchAgent
    containers/             # compose template + searxng settings + compose_v2 up
    caddy/                  # template AI site into existing Caddyfile + validate + reload
    validate/               # post-deploy assertions
  templates/
    docker-compose.yml.j2
    searxng-settings.yml.j2
    com.otto.ollama.plist.j2
    com.otto.infinity-reranker.plist.j2
    com.otto.iogpu-wired-limit.plist.j2
    Caddyfile.ai.j2
  README.md
```

`ansible.cfg`:
```ini
[defaults]
inventory = inventory/hosts.yml
vault_password_file = ~/.vault_password.txt   # already exists
host_key_checking = False
interpreter_python = /usr/bin/python3          # arm64 system python, avoids Rosetta
stdout_callback = yaml
```

`inventory/hosts.yml`:
```yaml
all:
  hosts:
    m2ultra:
      ansible_connection: local
      ansible_python_interpreter: /usr/bin/python3
```

`requirements.yml` (the `ansible` 11.x bundle already includes these; keep for reproducibility):
```yaml
collections:
  - community.general   # homebrew, launchd
  - community.docker     # docker_compose_v2
  - ansible.posix
```

---

## Secrets (ansible-vault)

Create `group_vars/vault.yml` encrypted with the existing `~/.vault_password.txt`:

| Key | Source |
|---|---|
| `webui_secret_key` | `openssl rand -hex 32` |
| `searxng_secret` | `openssl rand -hex 32` |
| `open_terminal_api_key` | `openssl rand -hex 32` |
| `caddy_basic_auth_user` | chosen username |
| `caddy_basic_auth_hash` | `caddy hash-password --plaintext '<pw>'` (bcrypt) |

Never place real secrets in `all.yml` or the compose template plaintext.

---

## Key variables (`group_vars/all.yml`)

```yaml
brew_native: /opt/homebrew/bin/brew
ai_base: "{{ ansible_env.HOME }}/ai-box"
ai_hostname: beranci-ai.duckdns.org        # new DuckDNS name
webui_port: 3000
ollama_host_port: 11434
reranker_port: 7997
iogpu_wired_limit_mb: 57344                # 56 GB
ollama_context_length: 32768
ollama_keep_alive: 30m
reranking_engine: external                 # fallback: local
open_terminal_enabled: true
dozzle_enabled: true
docker_desktop_memory_gb: 8                # set manually in Docker Desktop UI
models:
  - llama3.3:70b-instruct-q4_K_M
  - qwen2.5:32b
  - llama3.2:8b
  - mxbai-embed-large
  # - qwen2.5-coder:32b                     # optional
```

Image tags: pin explicit versions captured at implementation time (query
`docker manifest inspect`/Docker Hub), never floating `:latest` in production.

---

## Implementation task list (agent-executable)

### 1. Scaffold
- Create the repo tree above; write `ansible.cfg`, `inventory/hosts.yml`, `requirements.yml`.
- Write `all.yml`; run `ansible-vault create/edit` for `vault.yml`.
- `ansible-galaxy collection install -r requirements.yml` (may be a no-op with the bundle).

### 2. `host` role
- Assert `ansible_architecture == arm64` and macOS.
- `file:` create `~/ai-box/{openwebui/core,openwebui/rag/docs,openwebui/rag/uploads,qdrant/storage,caddy}`.
- **Metal cap:** template `com.otto.iogpu-wired-limit.plist` LaunchDaemon running
  `sysctl -w iogpu.wired_limit_mb={{ iogpu_wired_limit_mb }}`; install to
  `/Library/LaunchDaemons` (become), load; also apply now via `command: sysctl -w ...` (become).
- **Docker Desktop running:** check `docker info` (retries); if down,
  `command: open -a Docker`; wait for daemon. Document that the VM memory limit is
  set once in Docker Desktop → Settings → Resources (not scriptable reliably).
- Optionally add a `pf` anchor to block inbound `:11434` from non-bridge interfaces
  (security task; document if skipped).

### 3. `ollama` role
- Ensure formula via native brew: `community.general.homebrew` with `path: /opt/homebrew/bin`,
  `name: ollama`, `state: present`.
- Template `com.otto.ollama.plist` (**LaunchAgent**) at `~/Library/LaunchAgents/` with
  `ProgramArguments: [/opt/homebrew/bin/ollama, serve]` and env:
  `OLLAMA_HOST=0.0.0.0:11434`, `OLLAMA_KEEP_ALIVE={{ ollama_keep_alive }}`,
  `OLLAMA_CONTEXT_LENGTH={{ ollama_context_length }}`, `OLLAMA_ORIGINS=*`.
  Load with `community.general.launchd`.
- Verify `GET http://127.0.0.1:11434/api/tags` (retries).
- Pull each model idempotently: `command: /opt/homebrew/bin/ollama pull {{ item }}`,
  `changed_when` based on presence in `ollama list`.

### 4. `reranker` role
- `command:` create venv: `/opt/homebrew/bin/python3.11 -m venv ~/ai-box/infinity-venv`
  (idempotent via `creates`).
- `pip install "infinity-emb[all]"` into the venv (heavy; allow long timeout).
- Template `com.otto.infinity-reranker.plist` LaunchAgent:
  `~/ai-box/infinity-venv/bin/infinity_emb v2 --model-id BAAI/bge-reranker-v2-m3 --engine torch --device mps --port {{ reranker_port }}`.
  Load via launchd; healthcheck `GET http://127.0.0.1:7997/health` (start_period ~120 s).
- If health fails after retries: set `reranking_engine: local` and skip (documented fallback).

### 5. `containers` role
- Template `searxng-settings.yml.j2`:
  `use_default_settings: true`, `server.limiter: false`, `server.image_proxy: true`,
  `search.formats: [html, json]` (json is mandatory or Open WebUI web search 403s).
- Template `docker-compose.yml.j2` (project name `ai-box-stack`) with services:
  - **open-webui**: `ghcr.io/open-webui/open-webui:<pinned>`; `ports: ["127.0.0.1:3000:8080"]`;
    `extra_hosts: ["host.docker.internal:host-gateway"]`; env:
    `WEBUI_SECRET_KEY={{ webui_secret_key }}`, `WEBUI_URL=https://{{ ai_hostname }}`,
    `OLLAMA_BASE_URL=http://host.docker.internal:11434`,
    `VECTOR_DB=qdrant`, `QDRANT_URI=http://qdrant:6333`, `ENABLE_QDRANT_MULTITENANCY_MODE=true`,
    `RAG_EMBEDDING_ENGINE=ollama`, `RAG_EMBEDDING_MODEL=mxbai-embed-large`,
    `RAG_OLLAMA_BASE_URL=http://host.docker.internal:11434`,
    `CONTENT_EXTRACTION_ENGINE=tika`, `TIKA_SERVER_URL=http://tika:9998`, `TIKA_SERVER_VERSION=4`,
    `ENABLE_RAG_HYBRID_SEARCH=true`,
    `RAG_RERANKING_ENGINE=external`, `RAG_EXTERNAL_RERANKER_URL=http://host.docker.internal:7997/rerank`,
    `RAG_RERANKING_MODEL=BAAI/bge-reranker-v2-m3`, `RAG_TOP_K=20`, `RAG_TOP_K_RERANKER=5`,
    `ENABLE_WEB_SEARCH=true`, `WEB_SEARCH_ENGINE=searxng`, `SEARXNG_QUERY_URL=http://searxng:8080/search`,
    `ENABLE_API_KEYS=true`,
    `TERMINAL_SERVER_CONNECTIONS=[{... "url":"http://open-terminal:8000","key":"{{ open_terminal_api_key }}"}]`.
    Volumes: `~/ai-box/openwebui/core:/app/backend/data`,
    `~/ai-box/openwebui/rag/docs:/app/backend/data/docs`,
    `~/ai-box/openwebui/rag/uploads:/app/backend/data/uploads`.
    **No `deploy.resources.devices` / no `--gpus`.**
  - **qdrant**: `qdrant/qdrant:<pinned>`, volume `~/ai-box/qdrant/storage:/qdrant/storage`.
  - **tika**: `apache/tika:<pinned>-full`.
  - **searxng**: `searxng/searxng:<pinned>`, config bind → `/etc/searxng/settings.yml`,
    env `SEARXNG_SECRET={{ searxng_secret }}`, `SEARXNG_BASE_URL=http://{{ ai_hostname }}/`.
  - **dozzle**: `amir20/dozzle:<pinned>`, mount `/var/run/docker.sock:/var/run/docker.sock:ro`.
  - **open-terminal**: `ghcr.io/open-webui/open-terminal:<pinned>`, env
    `OPEN_TERMINAL_API_KEY={{ open_terminal_api_key }}`, `OPEN_TERMINAL_MULTI_USER=true`.
  - Bring up with `community.docker.docker_compose_v2` (`project_src=~/ai-box`, `state=present`).
    If the module reports a missing dependency, `pip install docker` into the interpreter.
  - **Do not publish** qdrant/tika/searxng/open-terminal/dozzle ports to the LAN;
    only Open WebUI binds `127.0.0.1:3000`. Dozzle reached via Caddy if wanted.

### 6. `caddy` role
- **Discover how Caddy runs** (`brew services list`/launchctl/`ps`) and where it reads its
  config; do not blindly overwrite `~/Caddyfile`.
- Template `Caddyfile.ai.j2` as the new site:
  ```caddy
  {{ ai_hostname }} {
      basic_auth {
          {{ caddy_basic_auth_user }} {{ caddy_basic_auth_hash }}
      }
      reverse_proxy 127.0.0.1:{{ webui_port }}
  }
  ```
- Insert/refresh this block in the active Caddyfile idempotently (managed marker block),
  preserving the existing Jellyfin site.
- `caddy validate --config <active>` then reload: `caddy reload --config <active>`
  (admin API :2019). If reload unavailable, restart the Caddy service via its launchd label.
- User prerequisite: register `{{ ai_hostname }}` in DuckDNS → same home IP (reuse `duckdns.sh`).

### 7. `validate` role
- `docker compose ps` all `running`/`healthy`.
- From a container: `docker run --rm curlimages/curl -s http://host.docker.internal:11434/api/tags`.
- `curl -fsS http://127.0.0.1:7997/health`.
- `curl -fsS http://127.0.0.1:3000/` returns HTML.
- `curl -fsS https://{{ ai_hostname }}/` (from LAN) returns Open WebUI and prompts basic_auth.
- `ollama ps` after loading 70B shows **100% GPU** (confirms Metal cap step worked).
- Manual: upload a doc in Open WebUI → hybrid search + rerank returns cited context;
  web search via SearXNG works; Dozzle shows logs.

---

## Rollback

```sh
docker compose -f ~/ai-box/docker-compose.yml down
launchctl bootout gui/$(id -u)/com.otto.ollama            # stop native ollama agent
launchctl bootout gui/$(id -u)/com.otto.infinity-reranker
sudo launchctl bootout system/com.otto.iogpu-wired-limit  # and rm the plist
# remove the managed AI block from the Caddyfile, then: caddy reload --config <active>
```

---

## Open items / user prerequisites

1. Register the second DuckDNS name and point it at the home IP; confirm HTTPS cert issuance.
2. Confirm the Caddy supervision mode (discovery task) so reload works.
3. Verify `ghcr.io/open-webui/open-terminal` publishes **linux/arm64** (fallback: skip Open
   Terminal for v1).
4. Set Docker Desktop VM memory to ~8 GB once (not fully scriptable).
5. Decide `open-terminal` exposure: keep it internal (recommended) or add a Caddy path.
6. Disk headroom: monitor free space as 70B + 32B + coder are pulled.
