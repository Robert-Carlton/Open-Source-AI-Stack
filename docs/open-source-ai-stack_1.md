# A Fully Open Source AI Stack for SMBs — Reference Architecture

*Runs on 1–2 physical servers. Everything from the OS to the end-user chat/analytics interface is open source. No data or inference calls leave the building unless the org chooses to enable them.*

Last updated: August 2026

---

## 1. Design principles

Before the component list, the choices below all follow from five constraints:

**Everything must run fully on-prem.** No component in the recommended path requires a cloud account, a SaaS API key, or outbound internet access to function. Anything that *can* optionally call out (e.g., pulling model weights, checking for updates) must be able to run entirely offline once provisioned.

**It has to fit on 1–2 servers.** This is not a "how would a hyperscaler do it" design — it deliberately avoids multi-node Kubernetes, distributed message queues, and other infrastructure that only pays for itself at a scale SMBs don't operate at. Every component is chosen to run comfortably as a set of containers on modest hardware.

**One identity system, one door in.** Every application in the stack — the chat UI, the dashboards, the admin consoles — sits behind a single SSO/OIDC provider and a single reverse proxy. Users log in once; there's no sprawl of app-specific logins for IT to manage.

**Boring and maintainable beats cutting-edge.** Where two open source options are roughly comparable, this document favors the one with the larger community, the more boring operational story, and the better track record of not disappearing in 18 months. A two-person IT team needs to be able to run this without a dedicated platform engineer.

**Open source means the license, not just the price — and licenses move, so treat every entry below as perishable.** Most components ship under a genuine OSI-approved license (Apache 2.0, MIT, AGPLv3, MPL, etc.) with no required paid tier to reach the functionality described here. Where a project has an open core plus a separately-licensed paid/enterprise layer (Langfuse, LiteLLM, Flowise, Authentik all have this shape to varying degrees), this document calls that out explicitly rather than blanket-labeling the whole project by its friendliest license. A few components recommended here — Open WebUI and n8n, specifically — are source-available rather than OSI-approved; they're included because they're the most capable option for the job, with a genuinely OSI-licensed alternative named alongside them for anyone who needs to stay strictly OSI-clean. This whole stack should be re-audited for license drift periodically, not just once at build time — see §6's footnote for why that's not a hypothetical concern.

---

## 2. The stack at a glance

The stack is organized into ten layers, running bottom-to-top from bare metal to the applications end users actually touch.

| # | Layer | Job | Primary pick | Alternative |
|---|---|---|---|---|
| 1 | Operating system | Host OS on bare metal | Ubuntu Server LTS or Debian | Rocky Linux |
| 2 | Virtualization (optional) | Carve 1–2 physical boxes into VMs | Proxmox VE | none (bare metal) |
| 3 | Container runtime & orchestration | Run and manage all services | Docker + Docker Compose | k3s (lightweight Kubernetes) |
| 4 | Networking & edge | TLS, routing, reverse proxy | Traefik | Caddy / Nginx |
| 5 | Identity & SSO | Single sign-on, MFA, user/group management | Keycloak | Authentik |
| 6 | Secrets management | Store and inject credentials/API keys | OpenBao (Vault fork) | Docker/K8s secrets + SOPS |
| 7 | Data platform | Relational + vector + object + cache storage | PostgreSQL (+pgvector), SeaweedFS, Redis | Qdrant, Milvus |
| 8 | Model serving / inference | Run LLMs and embedding models locally | Ollama or vLLM, llama.cpp | TGI, LocalAI |
| 9 | AI application & orchestration layer | RAG, agents, chat UI, ML pipelines | Open WebUI, LiteLLM, LangChain/LlamaIndex, MLflow, Airflow | LibreChat, Langflow, Dagster, n8n |
| 10 | Observability & guardrails | Metrics, logs, LLM tracing, PII/safety filtering | Prometheus, Grafana, Loki, Langfuse, Presidio | Wazuh (SIEM) |

Everything below expands on why each pick was made and how the pieces connect.

---

## 3. Hardware & topology

### How much hardware do you actually need?

The single biggest variable in this stack is whether a server has a GPU. It changes which inference engine you use, which models are practical, and how many concurrent users you can support.

**CPU-only path.** Entirely viable for SMB workloads that are moderate in volume: internal Q&A over company documents, ticket classification, churn/lead scoring, light summarization. Use `llama.cpp`/Ollama with quantized 3B–14B models — see §4.8 for current model picks in this size class, since this is the fastest-aging set of specifics in the whole document. Expect single-digit to low-teens tokens/second per request and plan for a handful of concurrent chat users, not dozens. Classical ML (scikit-learn/XGBoost) for classification and prediction is CPU-native and unaffected by this choice — it runs well regardless.

**GPU-accelerated path (recommended if budget allows).** One server with a 24GB VRAM GPU (RTX 4090, or a used A6000/L40S) comfortably handles 13B–34B-class models at usable multi-user latency via vLLM, and makes embedding generation for RAG close to instant. Be precise about the ceiling here: a genuine 70B model at a reasonable quantization (Q4) needs roughly 40GB of VRAM just for weights, plus headroom for KV cache — 24GB is *not* enough for 70B without dropping to aggressive sub-4-bit quantization, which comes with a real quality hit. If 70B-class quality is the target, budget for a 48GB card (A6000 48GB, L40S 48GB) or two 24GB cards, not one. This is the difference between "a chatbot the whole company can use during business hours" and "a proof of concept."

You don't have to choose up front — Ollama and vLLM both speak an OpenAI-compatible API, so the application layer (Open WebUI, LiteLLM, your RAG code) doesn't need to change when you add a GPU later. Start CPU-only, prove out the use cases, add the GPU box when usage justifies it.

### 1-server topology

Everything on one machine, sized around 8+ cores, 64–128GB RAM, NVMe storage, GPU optional. Good for pilots and single-department deployments. Capacity depends heavily on whether a GPU is present: on CPU-only hardware, hold to the "handful of simultaneous chat sessions" ceiling from above; add a GPU and the same box can reasonably support several dozen simultaneous *active* users, since in a typical org most people aren't issuing an LLM request at the exact same instant — total headcount and concurrent in-flight requests are very different numbers for intermittent chat usage. Don't treat either figure as a hard SLA; validate against your own usage pattern before committing to a headcount.

```
Server A (single box)
├── Proxmox (optional) or bare Ubuntu
└── Docker Compose stack:
    Traefik → Keycloak, Open WebUI, LiteLLM, Ollama/vLLM,
              PostgreSQL+pgvector, SeaweedFS, Redis, Airflow,
              MLflow, Grafana/Prometheus/Loki, Langfuse
```

The main risk here is resource contention: the LLM eating all the RAM/VRAM while Postgres and Airflow jobs compete for CPU. Set container resource limits deliberately.

### 2-server topology (recommended once past pilot stage)

Splits general-purpose services from GPU/inference workloads, which is the split that actually matters for performance and blast radius.

```
Server A — "Platform"                Server B — "AI/Compute"
├── Traefik (edge/TLS)                ├── Ollama / vLLM (model serving)
├── Keycloak (SSO)                    ├── Embedding model server
├── OpenBao (secrets)                 ├── GPU driver stack (if present)
├── PostgreSQL + pgvector             └── MLflow model registry / training jobs
├── SeaweedFS (object storage)
├── Redis
├── Airflow / Dagster (pipelines)
├── Open WebUI, LiteLLM (app/gateway)
└── Grafana/Prometheus/Loki/Langfuse
```

Server A can be a modest CPU box (8–16 cores, 32–64GB RAM); Server B is where the GPU budget goes. This split also gives a clean backup/DR story — Server A holds all the state that matters (databases, configs, secrets), Server B is stateless compute that can be rebuilt from scratch.

### When to graduate to Kubernetes (k3s)

Docker Compose is the right default for 1–2 servers — it's dramatically simpler to operate, debug, and back up, and an SMB IT team can hold the whole thing in their head. Move to k3s only when you outgrow 2 servers, need rolling zero-downtime deploys, or need workload auto-placement across more nodes than that. It's a real upgrade path (most of these components ship official Helm charts), just not a starting point.

---

## 4. Layer-by-layer detail

### 4.1 Operating system

**Ubuntu Server LTS or Debian.** Both are free, rock-solid, have the longest security-patch runway of any Linux distro, and have first-class support from every project in this stack (NVIDIA drivers, Docker, ZFS). Debian if you want maximum stability and don't mind older package versions; Ubuntu LTS if you want more current packages and slightly easier NVIDIA/CUDA driver setup. Either is a legitimate choice — pick based on what your team already knows.

Enable unattended security updates, disable password SSH auth in favor of keys, and use ZFS or LVM for the data volumes so you get snapshotting for cheap, fast backups.

### 4.2 Virtualization layer (optional but recommended)

**Proxmox VE** (AGPLv3) turns each physical server into a hypervisor host, letting you carve out VMs — e.g., a VM for the "Platform" role and a separate VM with GPU passthrough for the "AI/Compute" role, even on a single physical box. This buys you: clean snapshots before upgrades, the ability to isolate a compromised or misbehaving service inside its own VM, and an easy path to add a second physical node into a small cluster later. It's optional — you can run Ubuntu bare-metal with Docker directly — but the operational safety net is usually worth the modest overhead for a business-critical system.

### 4.3 Container runtime & orchestration

**Docker + Docker Compose.** One `docker-compose.yml` (or a small set of them per host) becomes the entire system definition — every service, every dependency, every volume, version-controlled in git. Upgrades are `git pull && docker compose up -d`. Backups are "back up the named volumes and the compose files." This is deliberately the simplest thing that works at 1–2-server scale, and it's what the entire industry defaults to for self-hosted stacks at this size.

**k3s** is the noted growth path (see §3) — a genuinely lightweight, single-binary Kubernetes distribution from Rancher/SUSE, not the sprawling full-fat k8s. Worth adopting once you have 3+ nodes or need real HA, but adds real operational complexity (etcd or SQLite state, RBAC, ingress controllers, cert-manager) that isn't justified below that scale.

### 4.4 Networking & edge — reverse proxy, TLS, ingress

**Traefik** (MIT license) sits in front of everything, terminates TLS (via Let's Encrypt if internet-facing, or an internal CA/self-signed cert if fully air-gapped), and routes each hostname/path to the right backend container. It auto-discovers services from Docker labels, so adding a new internal app doesn't mean hand-editing proxy config. **Caddy** is a strong, arguably even simpler alternative if the team prefers its config file style; both are good choices and either is a defensible pick.

Pair this with Traefik's **forward-auth** middleware pointed at Keycloak/Authentik so that *every* service behind the proxy gets SSO enforcement for free, without each individual app needing to implement OIDC itself.

### 4.5 Identity & SSO

**Keycloak** (Apache 2.0, a CNCF/Red Hat-backed project) is the recommended identity provider: full OIDC and SAML support, fine-grained role/group management, MFA (TOTP, WebAuthn/passkeys), and the broadest compatibility list of any open source IdP — every tool in this stack (Grafana, Open WebUI, Airflow, etc.) speaks OIDC to it natively. It's Java-based and a bit heavier to run than the alternative, but it's the most "enterprise-ready" and best-documented choice, which matters when this is the single point every user authenticates through.

**Authentik** is a very credible alternative — lighter weight, a more modern UI, Python-based, and its forward-auth "outpost" pattern is arguably a cleaner fit for a Traefik-fronted homelab-style deployment. Either is a legitimate default; Keycloak is favored here for its maturity and breadth of pre-built integrations, which reduces the amount of custom OIDC wiring an SMB IT team has to do by hand.

Practical setup: one Keycloak realm for the org, groups mapped to roles (e.g., `analytics-users`, `ai-admins`, `finance-viewers`), and every downstream app configured for OIDC/group-claim-based authorization rather than its own local user table. This is what "run all activities locally, one login" actually means in practice.

### 4.6 Secrets management

**OpenBao** (the community/Linux-Foundation fork of HashiCorp Vault, MPL 2.0, created after Vault's 2023 license change) stores database credentials, API keys, and TLS certs, and injects them into containers at runtime instead of sitting in plaintext `.env` files. For a 1–2-server deployment this can feel like overkill early on — a reasonable minimum viable version is git-encrypted secrets via **SOPS + age**, graduating to OpenBao once the number of services and credential-rotation needs grows.

### 4.7 Data platform

- **PostgreSQL** as the relational backbone — application state, Keycloak's user store, Airflow/MLflow metadata, and business data all live here. Rock solid, and every other tool in this list supports it as a backend.
- **pgvector** extension turns that same Postgres instance into a capable vector store for RAG embeddings — for SMB-scale document collections (a reasonable rule of thumb is up to a few million chunks, though the real ceiling is whether your HNSW index fits comfortably in RAM, not a fixed vector count — performance falls off a cliff once it spills to disk) this is genuinely enough, and it means one database to back up instead of two. If retrieval volume/latency outgrows it, add **Qdrant** (Apache 2.0, Rust, excellent open source vector DB) as a dedicated service — it's a drop-in swap since most RAG frameworks abstract the vector store behind an interface.
- **Object storage:** this recommendation changed recently and it's worth tracking the full arc, because it illustrates exactly the kind of risk this whole document tries to flag. **MinIO** used to be the default answer here. Through 2025 it stripped most management features (and LDAP/OIDC login) out of its open source Community Edition, pushing them behind its paid AIStor product; stopped publishing prebuilt Docker images/binaries for the community edition as of October 2025; went into "maintenance mode" that December; and by February 2026 the community edition's GitHub repository was archived outright — meaning upstream MinIO CE is no longer a maintained open source project at all, only a community fork (`pgsty/minio` is the one currently being used as its continuation). **SeaweedFS** (Apache 2.0) is the right default now — it's the closest thing to a genuine drop-in S3-compatible replacement and remains fully open and actively maintained. **Garage** (AGPLv3, purpose-built for small/geo-distributed deployments) is a solid alternative if SeaweedFS doesn't fit. Whichever you pick, treat "is this project's open source edition still fully functional and actively maintained" as a standing question to re-check periodically, not a one-time check at build time — this is now the second component in this document alone (after Redis) to go through real license/availability churn within about a year, and it won't be the last.
- **Redis** (or **Valkey**, see below) for caching, session state, and lightweight job/queue needs (e.g., Celery-backed background tasks).

### 4.8 Model serving / inference layer

This is the layer that runs the actual LLMs and embedding models.

- **Ollama** — the easiest on-ramp. One command pulls and serves a model, exposes an OpenAI-compatible API, and handles both GPU and CPU-only hosts gracefully. Best fit for low-to-moderate concurrency (a handful of simultaneous users) and for teams who want the least operational overhead.
- **vLLM** — the production-grade choice once you have a GPU and real concurrent usage. Its PagedAttention memory management delivers substantially higher throughput under concurrent load than Ollama — third-party benchmarks commonly show something on the order of a 5–20x aggregate-throughput gap at 5+ simultaneous requests, though the exact multiple swings a lot with model, hardware, and batch settings, so treat any specific number (including ones in this document) as directional rather than a guarantee you should quote verbatim. The qualitative point stands regardless: vLLM is built for concurrency, Ollama is built for simplicity, and the gap between them widens as usage grows.
- **llama.cpp** underlies Ollama specifically (Ollama is, at its core, a management/packaging layer on top of it) and is also a fine direct choice for CPU-only or embedded/edge deployments where you want maximum control over quantization and memory footprint. It's worth being precise that llama.cpp is *not* what vLLM runs on — vLLM is an independently engineered inference engine with its own CUDA/HIP kernels and its own memory manager (PagedAttention); the two are competing alternatives, not one built on the other.

Recommended models to start with, and a reminder that this is the fastest-moving part of the whole stack — whatever's named here will be at least one generation stale within a year, so treat these as illustrative of the *size class* to look for, not a fixed shopping list: as of this writing, Qwen3 and Mistral 3/Ministral 3 are the current-generation picks in the 8B–14B range for CPU/small-GPU hosts (both superseding the Qwen2.5/Mistral-Nemo generation), alongside Llama 3.x and Phi-4, which both remain reasonable; a current-generation model in the 32B–70B class for larger GPU hosts. All of the above ship under open-weight, commercially usable licenses, but verify the specific license per model/version before deployment — that varies model to model and sometimes version to version. Use a dedicated embedding model rather than reusing the chat model for embeddings — purpose-built embedding models are smaller, faster, and better at the retrieval task; `bge-m3` or `Qwen3-Embedding` are more current picks than the earlier `bge-large`/`nomic-embed-text-v1` generation.

**LiteLLM** sits in front of Ollama/vLLM as a unified OpenAI-compatible gateway/proxy — it gives you one API endpoint, per-user/per-team API keys, rate limiting, cost/usage tracking, and the ability to add a cloud model as a fallback later without touching application code. Every app above this layer (Open WebUI, custom RAG code, agents) talks to LiteLLM, not directly to the inference engine. One license nuance worth flagging: LiteLLM's core proxy is MIT, but it also ships an `enterprise/` directory under separate, non-MIT terms, and there's open community discussion about enterprise feature-gating logic embedded in files that are nominally MIT — the free proxy functionality described here (routing, keys, rate limits, cost tracking) is confirmed to work without a license key, but don't assume every feature you find in the repo is freely usable without checking which side of that line it's on.

### 4.9 AI application & orchestration layer

This is where the "ChatGPT-like interface" and the classification/prediction tooling actually live.

**Open WebUI** is the recommended chat frontend on functionality — it's the closest equivalent to the ChatGPT UI available today: chat history, multi-model switching, file upload with built-in RAG, per-user OIDC login (via Keycloak), team/workspace ("Channels") support, and a plugin ("tools"/"functions") system for calling out to other services. It talks to models through LiteLLM. A license flag worth taking seriously given this document's own standard: Open WebUI was BSD-3-Clause through v0.6.5, but as of v0.6.5 (April 2025) it moved to a custom "Open WebUI License" — BSD-3 as a base, plus a clause requiring visible Open WebUI branding for any deployment serving more than 50 users over a 30-day period, unless you're a substantive contributor or hold an enterprise license. That change drew real community pushback (a further licensing dispute followed in November 2025) and the license is not OSI-approved. Most SMB pilots described in this document will sit under that 50-user threshold and never trigger the clause, but if branding-neutral deployment or a strictly OSI-approved license is a hard requirement, **LibreChat** (MIT) is the clean alternative — slightly less polished RAG-file-upload UX out of the box, but functionally comparable and unencumbered.

For **RAG** beyond what Open WebUI's built-in retrieval offers — more control over chunking strategy, hybrid search, re-ranking, multi-source ingestion pipelines — build the pipeline with **LangChain** or **LlamaIndex** (both MIT), pulling from PostgreSQL/pgvector or Qdrant, and expose the result either as a LiteLLM-compatible endpoint or as an Open WebUI "tool." **Haystack** (Apache 2.0) is a strong alternative RAG framework if the team prefers its more pipeline-oriented API.

For **agentic workflows** (multi-step reasoning, tool use, task automation) — **LangGraph** (part of the LangChain ecosystem, MIT) for code-first agent graphs, or for low-code workflow building that lets non-engineers wire up AI-assisted business processes (e.g., "summarize this inbound email, classify its intent, draft a reply, route for approval"): **Langflow** (MIT) is the current pick — actively maintained, DataStax-backed, and fully OSI-licensed. **n8n** (fair-code/Sustainable Use License — not OSI-approved, verify against the org's license requirements) remains a strong option if its broader library of pre-built integrations outweighs the license caveat. **Flowise** (Apache 2.0 for the community codebase, with a separately-licensed `enterprise/` module) was the previous default recommendation here but has announced end-of-life — code freeze July 2026, repository archived shortly after — so it's noted for completeness rather than recommended for a new build; existing Flowise deployments remain usable and forkable under Apache 2.0, but plan a migration path rather than building new on it.

For **classification and prediction** (the "other algorithms" beyond LLMs — churn prediction, lead scoring, fraud/anomaly detection, demand forecasting): this is classical ML, not LLM territory, and it's the highest-ROI, lowest-compute part of the stack for most SMBs. **scikit-learn** and **XGBoost/LightGBM** for models, **MLflow** (Apache 2.0) for experiment tracking and a model registry, and **Airflow** or **Dagster** (Apache 2.0) to schedule the retraining/scoring pipelines. Package trained models behind a simple **FastAPI** service (or MLflow's own model-serving) so the rest of the stack can call them over HTTP just like any other internal API. This is a fully supported, self-contained workflow — a developer building something like a churn classifier never has to leave this stack. See §5, Example B for the concrete build/deploy/serve walkthrough, including exposing a scoring endpoint other internal systems can push data to.

### 4.10 Observability, guardrails & model performance monitoring

This layer answers two distinct questions the org will ask constantly: "is the system healthy," and "are the models actually still good." Both are covered:

**LLM usage & performance.** **LiteLLM** (from the model-serving layer) logs every request per user/team/app — token counts, cost estimate, latency, error rate — and can enforce per-user/per-team rate limits or budgets out of the box. **Langfuse** (MIT core) sits alongside it for deeper tracing — every prompt/completion, full retrieval context for RAG calls, latency and token usage broken down per user/app/model, and side-by-side comparison when you swap models (e.g., "did the new model answer these test questions better than the old one"). Together these are the "which model is being used, how much, how fast, how well" logging layer for LLMs specifically.

**Classical ML model performance.** A separate concern, covered by **MLflow** (training-time: every run's hyperparameters and metrics — accuracy/precision/recall/AUC — plus full version history of what was deployed when) and **Evidently AI** (Apache 2.0) for *production* monitoring — it compares live prediction data against training data to catch data drift and performance decay (e.g., "the churn model's precision has quietly dropped over the last month because customer behavior shifted"), with pre-built dashboards for exactly this. Without something like Evidently, a deployed classifier is a black box once it leaves the training notebook — this is the piece that keeps it honest over time.

**Infrastructure health.** **Prometheus + Grafana + Loki** — the standard open source metrics/dashboards/logs trio for the underlying containers/hosts (CPU, memory, GPU utilization, disk, error logs). Every component above exposes Prometheus metrics or logs that funnel here; one set of dashboards for the whole stack, including GPU utilization on the inference box, which matters for capacity planning.

**Guardrails & safety.** **Presidio** (MIT; originated at Microsoft and still commonly called "Microsoft Presidio," though it's actively transitioning to independent, community governance under the Data Privacy Stack organization — the license and functionality are unaffected, just don't be surprised if the branding shifts) for PII detection/redaction — scan documents and chat inputs/outputs for things like SSNs, account numbers, and names before they're logged or sent to a model, which matters even more when "fully local" is a compliance requirement, not just a preference. **LLM Guard** or **NeMo Guardrails** for input/output filtering — prompt-injection detection, topic restriction, and basic content-safety checks on both what users send in and what the model sends back.

### 4.11 Security hardening beyond the app layer

A few things worth calling out explicitly since "security should be included" was part of the brief:

Host-level firewall (`ufw`/`nftables`) restricting inbound traffic to only Traefik's ports 80/443, with everything else reachable only over an internal Docker/VM network. **Wazuh** (open source SIEM/EDR) if the org wants host and log-based intrusion detection. **Trivy** for scanning container images for known CVEs as part of the deploy pipeline. Regular automated backups of the Postgres volumes, object storage buckets, and OpenBao/secrets store using **Restic** or **BorgBackup**, pushed to a second physical location or at minimum a second disk — self-hosting everything means the org also owns 100% of the disaster-recovery responsibility that a cloud provider would otherwise carry.

---

## 5. Two worked examples

### Example A: "ChatGPT for the company," with RAG over internal documents

1. Documents (policies, contracts, wikis, PDFs) land in object storage (SeaweedFS), either via a watched folder/sync job or direct upload through Open WebUI.
2. An ingestion pipeline (Airflow-scheduled, using LangChain/LlamaIndex loaders) chunks documents, generates embeddings via the embedding model on Ollama/vLLM, and writes vectors to pgvector/Qdrant, tagged with access-control metadata mirrored from Keycloak group membership. Treat keeping that metadata in sync with live group changes (adds/removes) as its own ongoing operational task, not a set-and-forget step — stale metadata (a user removed from a group who still sees stale-indexed content until the next re-sync) is a real data-leak vector, not just a theoretical one.
3. A user logs into Open WebUI via Keycloak SSO. Their query goes to the RAG pipeline (exposed as a LiteLLM-compatible endpoint), which performs retrieval filtered by the user's current group membership, then calls the LLM itself through LiteLLM.
4. The LLM (served by Ollama or vLLM) generates a grounded answer with citations back to source documents. Presidio/LLM Guard scan the exchange; Langfuse logs the full trace for later review.

### Example B: Build-your-own classifier — e.g., "will this customer churn in the next 30 days"

This is a fully self-serve workflow for an internal developer/data scientist, start to finish, entirely inside this stack:

1. **Data.** Historical customer data (usage events, support tickets, billing history, plan changes) lives in PostgreSQL — either the org's operational data landing there directly, or synced in from other internal systems via Airflow.
2. **Labeling & features.** A Dagster/Airflow pipeline builds the training set: for each customer-month in history, compute features (recency/frequency of usage, ticket count, tenure, plan tier, payment failures, etc.) and a label — did that customer churn within the next 30 days, based on what actually happened afterward.
3. **Training.** The developer trains an XGBoost or scikit-learn model against that feature set — locally or in a scheduled pipeline job — and logs every run to **MLflow** (parameters, precision/recall/AUC, the trained artifact itself). Once a run looks good, they assign it the `champion` alias in the MLflow model registry (e.g., `churn-classifier: v7 → @champion`) — current MLflow uses alias-based promotion rather than the older Staging/Production "stage" labels, which MLflow has deprecated.
4. **Serving as an API.** The production model is wrapped in a small **FastAPI** service that loads the current MLflow-registered model and exposes an endpoint other systems can push data to, for example:

   ```python
   # churn-scoring-api/main.py
   from fastapi import FastAPI, Depends
   import mlflow.sklearn

   app = FastAPI()
   model = mlflow.sklearn.load_model("models:/churn-classifier@champion")

   @app.post("/v1/churn/score")
   def score(customer: CustomerFeatures, user=Depends(verify_oidc_token)):
       proba = model.predict_proba([customer.to_row()])[0][1]
       return {"customer_id": customer.customer_id,
               "churn_probability_30d": round(float(proba), 4),
               "model_alias": "champion"}
   ```

   (Note this loads the model via its native `mlflow.sklearn`/`mlflow.xgboost` flavor rather than the generic `mlflow.pyfunc` wrapper — the generic wrapper only exposes `.predict()`, not `.predict_proba()`, so the native flavor is what you actually want when you need probability scores rather than just a class label.)

   This endpoint sits behind Traefik like every other service, so it inherits TLS and can require a Keycloak-issued OIDC token or a scoped API key (issued via Keycloak client credentials) — the CRM or billing system authenticates the same way a human user would, just without a browser in the loop. "Pushing data" to it is simply: an internal app (CRM, billing system, a nightly batch job) `POST`s one customer's feature row — or a batch of them — and gets a churn probability back in the response, either one at a time for real-time use (e.g., flagging a customer the moment a support agent pulls up their account) or in bulk for a nightly scoring run that writes results back to Postgres.
5. **Feeding it new data.** If instead the intent is "the model should retrain on newly arrived data," that's the Airflow pipeline from step 2 — it's schedule-driven (e.g., nightly), pulling whatever new rows landed in Postgres since the last run, rather than a live push endpoint. Both patterns — push-to-score (real-time) and scheduled-retrain (batch) — are standard and both are covered here; which one an org needs depends on whether the requirement is "score this record right now" or "keep the model current over time."
6. **Monitoring it stays good.** Every request to `/v1/churn/score` is logged (Grafana/Prometheus for request volume/latency/errors); **Evidently AI** runs on a schedule comparing recent predictions and incoming feature distributions against the training baseline, flagging drift before precision quietly degrades — so the team finds out from a dashboard, not from a customer complaint.
7. This pipeline shares the same SSO, secrets, observability, and backup infrastructure as the RAG chatbot — it's not a separate stack, just a different set of containers plugged into the same platform layer.

---

## 6. Component summary & licensing reference

| Component | Purpose | License |
|---|---|---|
| Ubuntu Server / Debian | OS | GPL / various FOSS |
| Proxmox VE | Virtualization | AGPLv3 |
| Docker / Docker Compose | Containers | Apache 2.0 |
| k3s | Lightweight Kubernetes (growth path) | Apache 2.0 |
| Traefik | Reverse proxy / TLS | MIT |
| Caddy | Reverse proxy alternative | Apache 2.0 |
| Keycloak | SSO / IdP | Apache 2.0 |
| Authentik | SSO / IdP alternative | MIT (core) — separate proprietary license for `enterprise/` features |
| OpenBao | Secrets management | MPL 2.0 |
| PostgreSQL + pgvector | Relational + vector DB | PostgreSQL License / PostgreSQL License |
| Qdrant | Dedicated vector DB (optional) | Apache 2.0 |
| SeaweedFS | Object storage (S3-compatible) | Apache 2.0 |
| MinIO | Object storage — legacy option, see caveat below | AGPLv3 nominally, but CE repo archived Feb 2026 (no longer maintained upstream) |
| Redis | Cache / queue | AGPLv3 as of Redis 8 (May 2025) |
| Valkey | Cache / queue — BSD alternative to Redis | BSD-3-Clause |
| Ollama | LLM serving (simple) | MIT |
| vLLM | LLM serving (high-throughput) | Apache 2.0 |
| llama.cpp | LLM inference engine | MIT |
| LiteLLM | Unified model API gateway | MIT (core) — separate license for `enterprise/` features |
| Open WebUI | Chat UI | Custom "Open WebUI License" (BSD-3 base + branding clause) — not OSI-approved as of v0.6.5+ |
| LibreChat | Chat UI — fully OSI-licensed alternative | MIT |
| LangChain / LlamaIndex | RAG/agent frameworks | MIT |
| Haystack | RAG framework alternative | Apache 2.0 |
| LangGraph | Agent orchestration | MIT |
| Langflow | Low-code agent builder | MIT |
| Flowise | Low-code agent builder — announced end-of-life 2026, not recommended for new builds | Apache 2.0 (core) / separate license for `enterprise/` |
| n8n | Workflow automation | Sustainable Use License (source-available, not OSI — verify fit) |
| scikit-learn / XGBoost / LightGBM | Classical ML | BSD-3-Clause / Apache 2.0 / MIT |
| MLflow | ML experiment tracking/registry | Apache 2.0 |
| Evidently AI | Production ML model drift/performance monitoring | Apache 2.0 |
| Airflow | Workflow/pipeline scheduling | Apache 2.0 |
| Dagster | Pipeline orchestration alternative | Apache 2.0 |
| FastAPI | Model serving APIs | MIT |
| Prometheus / Grafana / Loki | Metrics, dashboards, logs | Apache 2.0 / AGPLv3 / AGPLv3 |
| Langfuse | LLM observability | MIT (core) |
| Presidio | PII detection/redaction | MIT |
| LLM Guard / NeMo Guardrails | Prompt/output safety filtering | MIT / Apache 2.0 |
| Wazuh | SIEM / host intrusion detection | GPLv2 |
| Trivy | Container vulnerability scanning | Apache 2.0 |
| Restic / BorgBackup | Backups | BSD-2-Clause / BSD-3-Clause |

*Redis's license went through real churn — BSD through 2023, then SSPL/RSAL (source-available, not OSI-approved) in 2024, then back to genuinely open under AGPLv3 with the Redis 8 release in May 2025, largely in response to community pressure and the rise of the **Valkey** fork (BSD-3, Linux Foundation — the fork created during the SSPL period, and still a completely reasonable choice if you'd rather not deal with AGPL's copyleft obligations). MinIO went further in the same direction and further still — reduced CE functionality, then no more binaries, then the repository itself archived within about a year. Open WebUI moved from a clean permissive license to a source-available one with a branding clause. Given that pattern — three separate components in this one document, within about eighteen months — treat "check the current license and maintenance status before you build on it" as a standing practice for this whole stack, not a one-time check at build time. Where this document names a fully OSI-licensed alternative alongside a source-available pick (LibreChat next to Open WebUI, LangGraph or Langflow next to n8n), that's there specifically so the org can make that swap without redesigning anything else.*

---

## 7. Suggested rollout phasing

**Phase 1 — Platform foundation.** OS, Docker, Traefik, Keycloak, Postgres, object storage, backups. Get SSO and one internal app (even something simple) working end to end before adding AI components.

**Phase 2 — Chat + RAG pilot.** Ollama with a small model, Open WebUI, LiteLLM, ingest one document set into pgvector. Prove the RAG use case with a small user group.

**Phase 3 — Harden and scale inference.** Add GPU hardware if usage justifies it, move to vLLM, add Langfuse/guardrails, expand document ingestion pipelines, roll out to the full org.

**Phase 4 — Add classical ML.** Bring in MLflow, Airflow/Dagster, and the first classification/prediction use case, sharing the platform layer already built.

**Phase 5 — Mature ops.** OpenBao for secrets, Wazuh/Trivy for security posture, and — only if the org has genuinely outgrown 2 servers — evaluate migrating orchestration to k3s.

---

## 8. Honest tradeoffs of going fully open source and self-hosted

This is worth stating plainly rather than glossing over: this stack trades convenience for control. There is no vendor SLA — the org's own team owns uptime, patching, and disaster recovery. Model quality on a locally-hosted 8B–70B model, even a good one, will generally trail the current frontier hosted models (GPT/Claude/Gemini-class) on the hardest reasoning tasks, though it's often more than sufficient for internal RAG, classification, and structured business tasks. And the up-front engineering time to wire all of these components together correctly is real, and scales with how much of this document you're building at once. A minimal working pilot — Phase 1–2 from §7: SSO plus one app, a small chat model, RAG over a single document set — is a realistic multi-week effort for a small team already comfortable with Docker, Keycloak, and basic LLM tooling. Reaching Phase 5 maturity — hardened security, full guardrails, a production classical-ML pipeline with drift monitoring, tested disaster recovery — is realistically a multi-month program, not a multi-week one, and that's before any compliance validation (audit trail review, data-retention policy, penetration testing) the org's industry requires, which this document doesn't scope at all. Comparable self-hosted internal-platform builds (identity + data + serving + observability, similar shape to this stack even when the specific components differ) commonly run 6–12 months to reach something a whole org actually relies on. The recurring cost after that point is genuinely just hardware, power, and upkeep — but don't let "weekend project" enter anyone's head for the full build.

What the org gets in exchange: no per-token or per-seat costs at scale, no data ever leaving the building, full control over model choice and update timing, and no dependency on any single vendor's continued existence or pricing decisions.
