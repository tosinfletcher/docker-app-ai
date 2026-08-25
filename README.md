# docker-app-ai

A self-contained, self-hosted **local AI stack** delivered as a single Docker Compose project. It pairs a local large-language-model inference engine with a full-featured web interface and API, so all model serving happens on your own hardware — no cloud LLM provider required.

| Service   | Image                          | Role                                                                  |
| --------- | ------------------------------ | --------------------------------------------------------------------- |
| Ollama    | `ollama/ollama`                | Runs and manages local LLMs; exposes the inference API (port 11434)  |
| Open WebUI | `ghcr.io/open-webui/open-webui` | Chat UI + OpenAI-compatible API backed by Ollama (port 8080)         |

> **Status:** All processing is local. Models are downloaded once and then served from disk. Outbound traffic is limited to image pulls, model downloads, and (in Traefik mode) certificate issuance.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Repository Layout](#repository-layout)
3. [Prerequisites](#prerequisites)
4. [Getting Started (Option A — Local)](#getting-started-option-a--local)
5. [Traefik Deployment (Option B)](#traefik-deployment-option-b)
6. [Configuration Reference](#configuration-reference)
7. [Working with Models](#working-with-models)
8. [API & Endpoints](#api--endpoints)
9. [CI/CD Pipeline (Jenkins)](#cicd-pipeline-jenkins)
10. [Data Persistence & Backup](#data-persistence--backup)
11. [Security Considerations](#security-considerations)
12. [Operations & Troubleshooting](#operations--troubleshooting)
13. [References](#references)

---

## Architecture

```
                            Option B (recommended)              Option A (local dev)
                        ┌───────────────────────┐             ┌───────────────────────┐
                        │   Domain: <DOMAIN>    │             │  Host: <WEBUI_PORT>   │
                        │   TLS via Traefik     │             │  Direct port publish  │
                        └───────────┬───────────┘             └───────────┬───────────┘
                                    │ HTTP/HTTPS                          │
                          ┌─────────▼─────────┐                           │
                          │  Traefik (ext.)   │             ┌─────────────┼────────────┐
                          │  network: EXTERNAL│◄────────────┤ ai_network  │            │
                          └─────────┬─────────┘   (Option B)└──┬─────────┴────┬───────┘
                                    │                           │              │ :8080
                            ┌───────▼───────┐            ┌──────▼───┐   ┌──────▼─────────┐
                            │ open-webui    │            │          │   │  open-webui    │
                            │ :8080 (chat,  │            │ shared   │   │  (same image)  │
                            │  API, admin)  │            │ compose  │   └──────┬─────────┘
                            │               │            │ network  │          │
                            │ OLLAMA_BASE_  │            └──────────┘          │
                            │ URL=ollama    │                                   │
                            └───────┬───────┘                                   │
                                    │  HTTP://OLLAMA:11434 (internal only)       │
                            ┌───────▼───────────┐                               │
                            │   ollama          │                              │
                            │   :11434          │◄──── internal call ───────────┘
                            │   models + config │
                            │   (ollama-data)   │
                            └───────────────────┘
```

**Data flow**

1. A user (or client tool) talks to **Open WebUI** over HTTP(S) — via the published host port (Option A) or through Traefik routing on `DOMAIN_NAME` with TLS (Option B).
2. Open WebUI forwards chat completions to **Ollama** using the internal service name `ollama` on port `11434`. Ollama is **never exposed directly to the host network**, so the inference API is reachable only from inside the compose network.
3. Both services persist state in named Docker volumes (`ollama-data`, `open-webui-data`), so containers can be recreated on every deploy — infrastructure is treated as cattle, state lives in volumes.

**Key design decisions**

- **Isolated network:** Both containers share a single compose network. In Traefik mode this is an *external* network (default: `proxy`) so the proxy can reach the app without the app reaching out.
- **Dual deployment modes:** The same Compose project serves both local testing (published port) and production (reverse proxy) — you flip two commented blocks, nothing else.
- **Hardened containers:** `security_opt: no-new-privileges=true` on every service.
- **Optional CI/CD:** A declarative Jenkins pipeline (see [CI/CD Pipeline](#cicd-pipeline-jenkins)) injects a secret-backed `.env`, pulls fresh images, and force-recreates the services. It is **optional** — the stack deploys identically with plain `docker compose`.

---

## Repository Layout

```
docker-app-ai/
├── README.md              # This document
├── docker-compose.yaml    # Service definitions (ollama + open-webui), network, volumes
├── Jenkinsfile            # CI/CD pipeline: checkout → inject secrets → deploy → verify
├── .env.example           # Committed template — copy to .env and fill in real values
└── .gitignore             # Keeps .env and build artifacts out of VCS
```

`docker-compose.yaml` defines:

| Construct          | Detail                                                                        |
| ------------------ | ------------------------------------------------------------------------------ |
| `services.ollama`  | Image `ollama/ollama`, data volume `ollama-data` → `/root/.ollama`             |
| `services.open-webui` | Image `ghcr.io/open-webui/open-webui`, volume `open-webui-data` → `/app/backend/data`, env `OLLAMA_BASE_URL=http://ollama:11434` + `WEBUI_SECRET_KEY` |
| `networks.ai_network` | **External** network named by `${TRAEFIK_NETWORK}` (Option B) **or** local bridge (Option A — commented) |
| `volumes`          | `ollama-data`, `open-webui-data` (both persistent named volumes)               |

> Note: `.env` itself is **git-ignored by design**. Real secrets never enter the repository; they are either copied from `.env.example` locally or injected by Jenkins from a secret file credential.

---

## Prerequisites

**Always required**

- A host running Docker Engine with the **Docker Compose v2 plugin** (`docker compose` command).
- Sufficient disk space for the models you plan to run (see [Working with Models](#working-with-models)).
- A GPU and compatible driver only if using CUDA-capable models — Ollama and Open WebUI will otherwise fall back to CPU.

**Required for Option B (Traefik)**

- An existing Traefik deployment that already publishes an **external network** whose name you will set in `TRAEFIK_NETWORK` (commonly `proxy`). The Docker host running this compose project (and the Jenkins agent, if you adopt the optional pipeline) must be attached to that network.
- A **DNS A (or AAAA) record** for `DOMAIN_NAME` pointing at the Traefik host.
- A Traefik TLS **entrypoint** (`TRAEFIK_ENTRYPOINT`) and a **certificate resolver** (`TRAEFIK_CERTRESOLVER`) already configured in Traefik — this project consumes them via labels; it does not configure Traefik itself.
- (For Let's Encrypt) public reachability of the port behind that entrypoint, or a Cloudflare-style DNS-01 resolver.

**Optional — required only if you adopt the Jenkins pipeline (preparing a Jenkins agent + two credentials first)** — see [CI/CD Pipeline (Jenkins)](#cicd-pipeline-jenkins).

---

## Getting Started (Option A — Local)

Use this path for first-time setup, local testing, or a single-node deployment without a reverse proxy.

**1. Switch `docker-compose.yaml` to the local mode.** Two edits, both clearly commented in the file:

- In `services.open-webui`: **uncomment the `ports:` block** (`${WEBUI_LOCAL_PORT}:8080`) and **comment out the entire `labels:` (Option B) block**.
- In top-level `networks:ai_network`: **uncomment the local bridge definition** and **comment out the external network block** so the project creates its own private network.

**2. Prepare the environment file:**

```bash
cp .env.example .env
# Then edit .env: set WEBUI_LOCAL_PORT (default 3000) and generate a strong secret key:
openssl rand -hex 32
```

(Option B variables in `.env` are inert in this mode.)

**3. Deploy and verify:**

```bash
docker compose up -d --remove-orphans
docker compose ps                 # both services should be "running"
```

**4. First-run steps:**

1. Open `http://<host-ip>:${WEBUI_LOCAL_PORT}` (e.g. `http://localhost:3000`).
2. Create the **administrator account** — done inside Open WebUI, not via env vars.
3. Pull and run a model (see [Working with Models](#working-with-models)). The first request that touches a model also downloads it on demand if you chose to let Open WebUI fetch it.

---

## Traefik Deployment (Option B)

The compose file **ships in Option B mode by default** — `labels` active, external network referenced. If you are deploying without changes, this is the production path.

**1. Ensure the Traefik prerequisites above exist**, then set the four `TRAEFIK_*` / `DOMAIN_NAME` variables in `.env`:

```ini
DOMAIN_NAME=ai.example.com            # DNS record → Traefik host
TRAEFIK_NETWORK=proxy                 # must match the EXISTING external network name
TRAEFIK_ENTRYPOINT=https              # a TLS-capable Traefik entrypoint
TRAEFIK_CERTRESOLVER=letsencrypt     # e.g. letsencrypt, cloudflare (DNS-01)
```

**2. Keep `docker-compose.yaml` as checked in** (Traefik labels + external network active; the `ports:` and local-network blocks remain commented).

**3. Deploy:**

```bash
docker compose up -d --remove-orphans
```

**4. Verify routing from an outsider's perspective** (not from inside the Docker host):

```bash
curl -vI https://${DOMAIN_NAME}/   # expect TLS handshake + 200/302 from Traefik
```

Inside Traefik you should see a router `open-webui` matching `Host(\`ai.example.com\`)` on your entrypoint, load-balancing to port `8080` of the `open-webui` container.

> **Switching between options later:** the two `ports`/`labels` pairs and the two network definitions are mutually exclusive. Exactly one of each pair must be active per service. Change them, then run `docker compose up -d --remove-orphans` again.

---

## Configuration Reference

All values are read from `.env` in the project root (Docker Compose auto-loads it). The committed template is [`.env.example`](.env.example) — copy it to `.env` before first use; never commit real values.

### Image versions

| Variable              | Default (template) | Consumed by     | Purpose / notes                                                            |
| --------------------- | ------------------ | --------------- | -------------------------------------------------------------------------- |
| `OLLAMA_VERSION`      | `latest`           | `ollama`        | Ollama image tag. **Pin to a release tag in production** for reproducible deploys. |
| `OPEN_WEBUI_VERSION`  | `main`             | `open-webui`    | Open WebUI image tag. Same pinning advice.                                 |

### Security

| Variable           | Default (template) | Consumed by  | Purpose / notes                                                                                   |
| ------------------ | ------------------ | ------------ | -------------------------------------------------------------------------------------------------- |
| `WEBUI_SECRET_KEY` | *generate your own* (e.g. `openssl rand -hex 32`) | `open-webui` | Master secret used by Open WebUI for session/JWT signing. **Never reuse across installations.** Changing it invalidates existing sessions/tokens. |

### Option A — local deployment

| Variable           | Default (template) | Consumed by  | Purpose / notes                                                    |
| ------------------ | ------------------ | ------------ | ------------------------------------------------------------------ |
| `WEBUI_LOCAL_PORT` | `3000`             | compose (`ports`, Option A block only) | Host port Open WebUI's internal `8080` is published on. Change if port is taken. |

### Option B — Traefik deployment

| Variable                 | Default (template) | Consumed by   | Purpose / notes                                                                                             |
| ------------------------ | ------------------ | ------------- | ------------------------------------------------------------------------------------------------------------- |
| `DOMAIN_NAME`            | `ai.example.com`   | compose labels | Hostname used in the `Host()` router rule. Must resolve to the Traefik host.                                 |
| `TRAEFIK_NETWORK`        | `proxy`            | compose `networks` | **Name of an existing external Docker network** owned by Traefik. Compose does not create it — it must already exist. |
| `TRAEFIK_ENTRYPOINT`     | `https`            | compose labels | Traefik entrypoint the router is attached to.                                                                |
| `TRAEFIK_CERTRESOLVER`   | `letsencrypt`      | compose labels | Traefik cert resolver name; TLS termination happens entirely in Traefik.                                    |

> In Option A, the four Traefik variables are not used. In Option B, `WEBUI_LOCAL_PORT` is not used. Both sets may coexist safely in `.env`.

---

## Working with Models

Model binaries live **inside the `ollama-data` volume** (`/root/.ollama`). They survive container recreation and image re-pulls.

**Pull from the host (recommended, gives you progress output):**

```bash
docker compose exec ollama ollama pull llama3.2
docker compose exec ollama ollama list        # what's installed
docker compose exec ollama ollama show llama3.2
docker compose exec ollama ollama rm <model>   # remove
```

**Pull through the UI:** the model "pull" affordance in Open WebUI talks to Ollama for you — useful for first-time setup since the admin flow is one screen.

Practical guidance:

- Start small (`llama3.2` 3B, `qwen2.5` 7B-class) to validate RAM/CPU throughput before chasing 70B-class models.
- Check free disk: each model download plus KV-cache headroom during inference should fit with margin. `du -sh /var/lib/docker/volumes/*` is the quick way to see real footprint.
- Ollama's inference API is reachable **only from within the compose network** (container-to-container via the service name `ollama`); there is no host port to lock down — the true attack surface is Open WebUI, which TLS and (optionally) auth cover.

---

## API & Endpoints

**External surface**

| Target      | Option A                          | Option B                        | Notes |
| ----------- | --------------------------------- | ------------------------------- | ----- |
| Open WebUI  | `http://<host>:${WEBUI_LOCAL_PORT}` | `https://${DOMAIN_NAME}/`       | Chat UI, admin console, and the **OpenAI-compatible** API (`/api/...`) for programmatic chat completions. |

**Internal surface** (not published to the host)

| Target | Address (inside `ai_network`) | Consumed by |
| ------ | ----------------------------- | ----------- |
| Ollama | `http://ollama:11434`         | `open-webui` via `OLLAMA_BASE_URL` |

Because Ollama has **no published host port**, anything needing model access should go through Open WebUI's API (which also carries the auth/JWT layer) rather than reaching for `11434` directly — that address only resolves on the internal compose network.

---

## CI/CD Pipeline (Jenkins)

> **This section is optional.** The service runs fully without Jenkins — every workflow in this document also works with the plain `docker compose` commands shown in the getting-started sections. This pipeline exists purely to automate deployments for teams that want it; you can ignore this whole section (and its two Jenkins credentials) and operate production from this repo alone.

`Jenkinsfile` defines a declarative pipeline that deploys this exact stack to a Docker host **when adopted**. It is intentionally stateless: every run injects secrets, pulls fresh images, and force-recreates the two services.

**Pre-setup — required before the first trigger**

This pipeline is **not self-contained**. You must prepare your Jenkins instance and at least one agent (node) before you queue the job, or it will fail at configuration time before any deployment step runs.

*1. Prepare a Jenkins agent.* The job declares `agent any`, so every `docker compose` command executes **on whichever node picks up the job**. That node must:

- have Docker Engine installed with the **Compose v2 plugin** (the pipeline invokes the `docker compose` CLI — legacy v1 `docker-compose` will not work), plus a `git` client for `checkout scm`;
- be attached to the **same external Docker network** referenced by `TRAEFIK_NETWORK` (e.g., `proxy`) when deploying behind Traefik — that daemon can only route containers onto networks it is a member of, and `up` will fail with *network not found* otherwise;
- grant the build user access to the Docker socket (group membership or a `docker.sock` ACL) if it does not run as root.

*2. Create both Jenkins credentials beforehand.* Both credential IDs are resolved when the pipeline runs: `withCredentials` and the `environment` lookup **abort the build immediately** if either ID is missing — so even a smoke-test build requires both to exist. Create them under *Manage Jenkins → Credentials*:

| Credential ID (type)            | What to store                                                                                                             |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `jenkins-job-webhook-url` (secret text) | The notification endpoint (e.g., pumble) that receives a `{"text": "..."}` JSON POST on build failure with job name, build number, and log URL. If you have no real sink, store a placeholder URL that accepts the POST — the ID itself still must resolve. |
| `ai-env-file` (secret file)     | The **full contents of a real `.env`** (start from [`.env.example`](.env.example) and fill in production values). The pipeline copies it into the workspace before deploy, so repo edits to environment values have no effect on CI deploys — this credential is the source of truth for production. |

**Pipeline stages**

1. **Checkout** — clears the workspace, then checks out the current SCM branch (a `BRANCH_NAME` pipeline parameter, default `main`, is available for triggering specific branch deploys).
2. **Deploy with Docker Compose** —
   - Copies the secret file to `.env` in the workspace.
   - `docker compose pull` (honouring whatever tags are configured in `.env`).
   - `docker compose up -d --remove-orphans --force-recreate` — idempotent, converges the stack to the declared state and recreates both containers so tag changes always take effect.
3. **Post-Deploy Check** — `docker compose ps` as a basic liveness confirmation.

**Environment pins set by the Jenkinsfile**

- `COMPOSE_PROJECT_NAME=docker-app-ai` — deterministic project prefix, prevents collisions with other compose projects on the same agent and keeps volume names stable (`docker-app-ai_ollama-data`, etc.).
- `COMPOSE_FILE=docker-compose.yaml` — explicit path pin.

**Post-status behaviour**

- `always` → logs a summary and clears the workspace (`cleanWs()`), so no stale `.env` survives on the agent.
- `failure` → fires the webhook notification with failure context.

**Rollback notes:** because deployments are image-tag driven from `.env`, rolling back means deploying the previous known-good version of the *values*, i.e. re-injecting the old `.env` content (rollback is a config/credential change, not a code change). The `--force-recreate` flag is what makes this reliable.

---

## Data Persistence & Backup

| Volume               | Mounted at                  | Contents                                                       | Relevance |
| -------------------- | --------------------------- | -------------------------------------------------------------- | --------- |
| `ollama-data`        | `/root/.ollama`             | Downloaded models, blobs, Ollama runtime state                   | Lose it = re-pull all models (network + time cost) |
| `open-webui-data`    | `/app/backend/data`         | Users, API keys, chat history, workspace/prompt data, metadata DB | Lose it = accounts and history are gone |

Actual Docker volume names carry the project prefix: `docker-app-ai_ollama-data`, `docker-app-ai_open-webui-data` (the Jenkinsfile pins `COMPOSE_PROJECT_NAME` to keep these stable).

**Backup (host side, outside compose):**

```bash
# On the Docker host, with $PWD as the backup destination dir:
mkdir -p ./ai-backup/$(date +%F)
for v in docker-app-ai_ollama-data docker-app-ai_open-webui-data; do
  docker run --rm -v "$v":/src:ro -v "$PWD/ai-backup/$(date +%F)":/dst alpine \
    tar czf "/dst/${v}.tgz" -C /src .
done
```

Restore is the inverse: unpack the tarball into the directory a volume provides and recreate the container (Open WebUI re-attaches to its existing DB on start).

---

## Security Considerations

**Implemented by this project**

- `security_opt: no-new-privileges=true` on **both** containers.
- Ollama's model API is **not host-publishable by design** — only reachable from the internal compose network.
- Open WebUI's sensitive operations are tied to `WEBUI_SECRET_KEY`; the template explicitly directs you to generate a strong per-installation value rather than shipping one.
- TLS termination (Option B) in Traefik with a cert resolver — no plaintext exposure in production mode.
- Deployments are fully declarative and reproducible; there is no ad-hoc `docker run` or host-file mutation during release.

**Operational recommendations for this deployment**

- **Pin image tags.** `latest` / `main` are acceptable for a staging/dev setup; production should pin digests or at least release tags (the Jenkins `pull` + `up --force-recreate` pair makes this change atomic).
- **Treat `.env` as code-adjacent secret.** Keep it out of the repo (already enforced), and in CI prefer the Jenkins secret-file credential (already the path taken) over a checked-in values file.
- **Least-privilege for the Ollama user inside its container** — it runs as root by default (`/root/.ollama`). If you need a hardened runtime profile, consider wrapping Ollama in a sidecar that drops privileges to a non-root user, or running it under a seccomp/AppArmor profile. Trade-off vs. the CPU affinity + shared-memory needs of inference — benchmark before adopting.
- **Admin account hardening:** ensure the Open WebUI admin account has a strong password, and (if you use `/api` from other services) prefer scoped API keys over the admin token where possible.
- **Egress posture:** only image registry, model source, and certificate provider (Option B) need internet — any other egress is suspicious and worth a policy check.

---

## Operations & Troubleshooting

**Routine**

```bash
docker compose ps                                # service liveness
docker compose logs -f open-webui                # follow app logs
docker compose logs -f ollama                    # follow inference logs
docker compose exec ollama ollama ps             # loaded models / VRAM
docker compose pull && docker compose up -d      # manual update loop
```

**Common symptoms**

| Symptom                                                   | Likely cause / fix                                                                                              |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `network "proxy" not found` (or compose's own)            | Option B referencing a non-existent external network. Either create the network (Traefik side) or switch to Option A in `docker-compose.yaml`. |
| Open WebUI won't find Ollama after a host migration        | `OLLAMA_BASE_URL` still points at a service name that resolves only inside this network — keep both containers in the same project. |
| UI works locally, not via domain (Option B)               | DNS A record missing or pointing at the wrong IP; Traefik entrypoint/labels mismatch (compare against `traefik_api` router debug output). |
| `WEBUI_SECRET_KEY` change resets sessions                 | Expected and intentional — the key signs JWTs. Change only when rotating, and expect a one-time re-login storm. |
| Ollama pull is slow, then stops                            | Model download interrupted mid-write. Re-run the pull (Ollama resumes), or clear partial state under `/root/.ollama` inside the container. |
| Port conflict (Option A)                                   | `WEBUI_LOCAL_PORT` already bound. Change it in `.env`, recreate.                                                 |
| CPU fan spinning at idle after a GPU model was loaded      | Model still resident in VRAM; `docker compose exec ollama ollama ps` → unload with `ollama stop <model>` or restart ollama.                    |

---

## References

- [Ollama](https://github.com/ollama/ollama) — inference engine, model gallery at `https://ollama.com/library`
- [Open WebUI](https://github.com/open-webui/open-webui) — UI + OpenAI-compatible API
- [Traefik documentation](https://doc.traefik.io/traefik/) — labels, entrypoints, cert resolvers used by this project
- [Docker Compose reference](https://docs.docker.com/compose/reference/)

---

*Repository: `docker-app-ai` · Deployment: Docker Compose · CI/CD: optional Jenkins declarative pipeline*.
