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
inventory/group_vars/all.yml    # non-secret config
inventory/group_vars/vault.yml  # secrets (gitignored; encrypt before sharing)
playbooks/site.yml              # the deployment play
playbooks/preflight.yml         # read-only environment checks
roles/{host,ollama,reranker,containers,caddy,validate}/
```

## Prerequisites

1. **Native arm64 Homebrew** at `/opt/homebrew` (present). `/usr/local` is x86_64 here
   and is deliberately never used for installs.
2. **Docker Desktop** running. Optional but recommended: set its VM memory to ~8 GB
   (Settings → Resources), leaving room for the 70B model in unified memory.
3. **A second DuckDNS hostname** (e.g. `beranci-ai.duckdns.org`) pointing at the same
   home IP as your existing `beranci.duckdns.org` record. Update `ai_hostname` in
   `inventory/group_vars/all.yml` and re-run if you use a different name. Your `duckdns.sh`
   can be reused with the new token/subdomain.
4. **sudo/become** available for the Metal wired-limit step (`sysctl`) and the
   LaunchDaemon. Run the playbook with `-K` if sudo prompts.

Ollama 0.34.4 (arm64) is already installed and `~/.ollama/models` already holds
`qwen2.5:14b` and `translategemma:12b`; those are reused.

## Usage

```sh
# from the repo root (ansible.cfg is picked up automatically)
ansible-playbook playbooks/site.yml --ask-become-pass

# later, to also test the public HTTPS endpoint (after DNS resolves):
ansible-playbook playbooks/site.yml -e check_public_endpoint=true
```

First run pulls tens of GB of models (~43 GB for the 70B alone) and installs
torch for the reranker, so it can take a long time.

## Secrets

`inventory/group_vars/vault.yml` currently contains generated secrets in plaintext and is
gitignored. Encrypt it before it leaves this machine:

```sh
ansible-vault encrypt inventory/group_vars/vault.yml --vault-password-file ~/.vault_password.txt
```

The Caddy basic-auth credentials are in the vault
(`caddy_basic_auth_user`, `caddy_basic_auth_password`, `caddy_basic_auth_hash`).
Regenerate the hash after changing the password:

```sh
caddy hash-password --plaintext '<new-password>'
```

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
| Caddy site | native config block | `https://<ai_hostname>` → `:3000`, basic auth |
| Metal wired limit | LaunchDaemon `com.otto.iogpu-wired-limit` | `iogpu.wired_limit_mb=57344` |

Only Open WebUI (via Caddy) and Dozzle-on-localhost are exposed; everything else
stays on the internal Docker network or loopback. Note that `OLLAMA_HOST=0.0.0.0`
makes `:11434` reachable on the LAN — add a `pf` rule or keep the Mac on a trusted
network if that matters.

## Known caveats / TODO

- Container image tags are floating (`:main`, `:latest`). Pin explicit versions for
  reproducibility.
- If `infinity-emb[all]` fails to install (torch/MPS), the play automatically falls
  back to Open WebUI's built-in reranker (`RAG_RERANKING_ENGINE=local`).
- `open-terminal` is an admin-level shell. It is not published to the host and sits
  behind Open WebUI auth + Caddy basic auth. Keep it that way.
- On 64 GB, running the 70B model while the Docker Desktop VM is large can cause
  memory pressure. Keep the VM modest and expect ~10–15 tok/s for 70B, ~20–30 for 32B.

## Rollback

```sh
docker compose -f ~/ai-box/docker-compose.yml down
launchctl bootout gui/$(id -u)/com.otto.ollama
launchctl bootout gui/$(id -u)/com.otto.infinity-reranker
sudo launchctl bootout system/com.otto.iogpu-wired-limit
sudo rm /Library/LaunchDaemons/com.otto.iogpu-wired-limit.plist
# remove the "# BEGIN/END ANSIBLE MANAGED AI STACK" block from ~/Caddyfile, then:
caddy reload --config ~/Caddyfile
```
