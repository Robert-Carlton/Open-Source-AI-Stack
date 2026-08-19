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

**Open source means the license, not just the price.** Every component below ships under a genuine OSI-approved license (Apache 2.0, MIT, AGPLv3, MPL, etc.) with no required paid tier to reach the functionality described here. Where a project has both an open core and a paid/hosted edition, this document specifies the open-source edition.

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
| 7 | Data platform | Relational + vector + object + cache storage | PostgreSQL (+pgvector), MinIO, Redis | Qdrant, Milvus |
| 8 | Model serving / inference | Run LLMs and embedding models locally | Ollama or vLLM, llama.cpp | TGI, LocalAI |
| 9 | AI application & orchestration layer | RAG, agents, chat UI, ML pipelines | Open WebUI, LiteLLM, LangChain/LlamaIndex, MLflow, Airflow | Flowise, Dagster, n8n |
| 10 | Observability & guardrails | Metrics, logs, LLM tracing, PII/safety filtering | Prometheus, Grafana, Loki, Langfuse, Presidio | Wazuh (SIEM) |

Everything below expands on why each pick was made and how the pieces connect.

---

## 3. Hardware & topology

### How much hardware do you actually need?

The single biggest variable in this stack is whether a server has a GPU. It changes which inference engine you use, which models are practical, and how many concurrent users you can support.

**CPU-only path.** Entirely viable for SMB workloads that are moderate in volume: internal Q&A over company documents, ticket classification, churn/lead scoring, light summarization. Use `llama.cpp`/Ollama with quantized 3B–14B models (Llama 3.x, Qwen2.5, Phi-4, Mistral-Nemo class). Expect single-digit to low-teens tokens/second per request and plan for a handful of concurrent chat users, not dozens. Classical ML (scikit-learn/XGBoost) for classification and prediction is CPU-native and unaffected by this choice — it runs well regardless.

**GPU-accelerated path (recommended if budget allows).** One server with a 24GB+ VRAM GPU (RTX 4090, RTX 6000 Ada, L40S, or a used A6000) opens up 13B–70B-class models at usable multi-user latency via vLLM, and makes embedding generation for RAG close to instant. This is the difference between "a chatbot the whole company can use during business hours" and "a proof of concept."

You don't have to choose up front — Ollama and vLLM both speak an OpenAI-compatible API, so the application layer (Open WebUI, LiteLLM, your RAG code) doesn't need to change when you add a GPU later. Start CPU-only, prove out the use cases, add the GPU box when usage justifies it.

### 1-server topology

Everything on one machine, sized around 8+ cores, 64–128GB RAM, NVMe storage, GPU optional. Good for pilots, single-department deployments, or orgs under ~50 concurrent users.

```
Server A (single box)
├── Proxmox (optional) or bare Ubuntu
└── Docker Compose stack:
    Traefik → Keycloak, Open WebUI, LiteLLM, Ollama/vLLM,
              PostgreSQL+pgvector, MinIO, Redis, Airflow,
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
├── MinIO (object storage)
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

**Keycloak** (Apache 2.0, a CNCF/Red Hat-backed project) is the recommended identity provider: full OIDC and SAML support, fine-grained role/group management, MFA (TOTP, WebAuthn/passkeys), and the broadest compatibility list of any open source IdP — every tool in this stack (Grafana, MinIO, Open WebUI, Airflow, etc.) speaks OIDC to it natively. It's Java-based and a bit heavier to run than the alternative, but it's the most "enterprise-ready" and best-documented choice, which matters when this is the single point every user authenticates through.

**Authentik** is a very credible alternative — lighter weight, a more modern UI, Python-based, and its forward-auth "outpost" pattern is arguably a cleaner fit for a Traefik-fronted homelab-style deployment. Either is a legitimate default; Keycloak is favored here for its maturity and breadth of pre-built integrations, which reduces the amount of custom OIDC wiring an SMB IT team has to do by hand.

Practical setup: one Keycloak realm for the org, groups mapped to roles (e.g., `analytics-users`, `ai-admins`, `finance-viewers`), and every downstream app configured for OIDC/group-claim-based authorization rather than its own local user table. This is what "run all activities locally, one login" actually means in practice.

### 4.6 Secrets management

**OpenBao** (the community/Linux-Foundation fork of HashiCorp Vault, MPL 2.0, created after Vault's 2023 license change) stores database credentials, API keys, and TLS certs, and injects them into containers at runtime instead of sitting in plaintext `.env` files. For a 1–2-server deployment this can feel like overkill early on — a reasonable minimum viable version is git-encrypted secrets via **SOPS + age**, graduating to OpenBao once the number of services and credential-rotation needs grows.

### 4.7 Data platform

- **PostgreSQL** as the relational backbone — application state, Keycloak's user store, Airflow/MLflow metadata, and business data all live here. Rock solid, and every other tool in this list supports it as a backend.
- **pgvector** extension turns that same Postgres instance into a capable vector store for RAG embeddings — for SMB-scale document collections (up to a few million chunks) this is genuinely enough, and it means one database to back up instead of two. If retrieval volume/latency outgrows it, add **Qdrant** (Apache 2.0, Rust, excellent open source vector DB) as a dedicated service — it's a drop-in swap since most RAG frameworks abstract the vector store behind an interface.
- **MinIO** (AGPLv3) provides S3-compatible object storage for documents, model artifacts, MLflow experiment outputs, and file uploads — so any tool expecting "an S3 bucket" works fully on-prem.
- **Redis** for caching, session state, and lightweight job/queue needs (e.g., Celery-backed background tasks).

### 4.8 Model serving / inference layer

This is the layer that runs the actual LLMs and embedding models.

- **Ollama** — the easiest on-ramp. One command pulls and serves a model, exposes an OpenAI-compatible API, and handles both GPU and CPU-only hosts gracefully. Best fit for low-to-moderate concurrency (a handful of simultaneous users) and for teams who want the least operational overhead.
- **vLLM** — the production-grade choice once you have a GPU and real concurrent usage. Its PagedAttention memory management delivers dramatically higher throughput under load (roughly an order of magnitude more aggregate tokens/second than Ollama under multi-user load, per current benchmarks) at the cost of being GPU-first and a bit more involved to configure.
- **llama.cpp** underlies both of the above and is also a fine direct choice for CPU-only or embedded/edge deployments where you want maximum control over quantization and memory footprint.

Recommended models to start with (all open-weight, commercially usable licenses — verify the specific license per model/version before deployment): Llama 3.x or Qwen2.5 in the 8B–14B range for CPU/small-GPU hosts, and a 32B–70B class quantized model for larger GPU hosts. Use a dedicated embedding model (e.g., `nomic-embed-text` or `bge-large`) rather than reusing the chat model for embeddings — purpose-built embedding models are smaller, faster, and better at the retrieval task.

**LiteLLM** (MIT) sits in front of Ollama/vLLM as a unified OpenAI-compatible gateway/proxy — it gives you one API endpoint, per-user/per-team API keys, rate limiting, cost/usage tracking, and the ability to add a cloud model as a fallback later without touching application code. Every app above this layer (Open WebUI, custom RAG code, agents) talks to LiteLLM, not directly to the inference engine.

### 4.9 AI application & orchestration layer

This is where the "ChatGPT-like interface" and the classification/prediction tooling actually live.

**Open WebUI** (BSD-3-Clause) is the recommended chat frontend — it's the closest genuinely open source equivalent to the ChatGPT UI: chat history, multi-model switching, file upload with built-in RAG, per-user OIDC login (via Keycloak), team/workspace support, and a plugin ("tools"/"functions") system for calling out to other services. It talks to models through LiteLLM.

For **RAG** beyond what Open WebUI's built-in retrieval offers — more control over chunking strategy, hybrid search, re-ranking, multi-source ingestion pipelines — build the pipeline with **LangChain** or **LlamaIndex** (both MIT), pulling from PostgreSQL/pgvector or Qdrant, and expose the result either as a LiteLLM-compatible endpoint or as an Open WebUI "tool." **Haystack** (Apache 2.0) is a strong alternative RAG framework if the team prefers its more pipeline-oriented API.

For **agentic workflows** (multi-step reasoning, tool use, task automation) — **LangGraph** (part of the LangChain ecosystem) for code-first agent graphs, or **n8n** (fair-code/Sustainable Use License — note: not OSI-approved for all uses, verify against the org's license requirements) or **Flowise** (Apache 2.0) for low-code workflow building that lets non-engineers wire up AI-assisted business processes (e.g., "summarize this inbound email, classify its intent, draft a reply, route for approval").

For **classification and prediction** (the "other algorithms" beyond LLMs — churn prediction, lead scoring, fraud/anomaly detection, demand forecasting): this is classical ML, not LLM territory, and it's the highest-ROI, lowest-compute part of the stack for most SMBs. **scikit-learn** and **XGBoost/LightGBM** for models, **MLflow** (Apache 2.0) for experiment tracking and a model registry, and **Airflow** or **Dagster** (Apache 2.0) to schedule the retraining/scoring pipelines. Package trained models behind a simple **FastAPI** service (or MLflow's own model-serving) so the rest of the stack can call them over HTTP just like any other internal API. This is a fully supported, self-contained workflow — a developer building something like a churn classifier never has to leave this stack. See §5, Example B for the concrete build/deploy/serve walkthrough, including exposing a scoring endpoint other internal systems can push data to.

### 4.10 Observability, guardrails & model performance monitoring

This layer answers two distinct questions the org will ask constantly: "is the system healthy," and "are the models actually still good." Both are covered:

**LLM usage & performance.** **LiteLLM** (from the model-serving layer) logs every request per user/team/app — token counts, cost estimate, latency, error rate — and can enforce per-user/per-team rate limits or budgets out of the box. **Langfuse** (MIT core) sits alongside it for deeper tracing — every prompt/completion, full retrieval context for RAG calls, latency and token usage broken down per user/app/model, and side-by-side comparison when you swap models (e.g., "did the new model answer these test questions better than the old one"). Together these are the "which model is being used, how much, how fast, how well" logging layer for LLMs specifically.

**Classical ML model performance.** A separate concern, covered by **MLflow** (training-time: every run's hyperparameters and metrics — accuracy/precision/recall/AUC — plus full version history of what was deployed when) and **Evidently AI** (Apache 2.0) for *production* monitoring — it compares live prediction data against training data to catch data drift and performance decay (e.g., "the churn model's precision has quietly dropped over the last month because customer behavior shifted"), with pre-built dashboards for exactly this. Without something like Evidently, a deployed classifier is a black box once it leaves the training notebook — this is the piece that keeps it honest over time.

**Infrastructure health.** **Prometheus + Grafana + Loki** — the standard open source metrics/dashboards/logs trio for the underlying containers/hosts (CPU, memory, GPU utilization, disk, error logs). Every component above exposes Prometheus metrics or logs that funnel here; one set of dashboards for the whole stack, including GPU utilization on the inference box, which matters for capacity planning.

**Guardrails & safety.** **Microsoft Presidio** (MIT) for PII detection/redaction — scan documents and chat inputs/outputs for things like SSNs, account numbers, and names before they're logged or sent to a model, which matters even more when "fully local" is a compliance requirement, not just a preference. **LLM Guard** or **NeMo Guardrails** for input/output filtering — prompt-injection detection, topic restriction, and basic content-safety checks on both what users send in and what the model sends back.

### 4.11 Security hardening beyond the app layer

A few things worth calling out explicitly since "security should be included" was part of the brief:

Host-level firewall (`ufw`/`nftables`) restricting inbound traffic to only Traefik's ports 80/443, with everything else reachable only over an internal Docker/VM network. **Wazuh** (open source SIEM/EDR) if the org wants host and log-based intrusion detection. **Trivy** for scanning container images for known CVEs as part of the deploy pipeline. Regular automated backups of the Postgres volumes, MinIO buckets, and OpenBao/secrets store using **Restic** or **BorgBackup**, pushed to a second physical location or at minimum a second disk — self-hosting everything means the org also owns 100% of the disaster-recovery responsibility that a cloud provider would otherwise carry.

---

## 5. Two worked examples

### Example A: "ChatGPT for the company," with RAG over internal documents

1. Documents (policies, contracts, wikis, PDFs) land in MinIO, either via a watched folder/sync job or direct upload through Open WebUI.
2. An ingestion pipeline (Airflow-scheduled, using LangChain/LlamaIndex loaders) chunks documents, generates embeddings via the embedding model on Ollama/vLLM, and writes vectors to pgvector/Qdrant, tagged with access-control metadata (which Keycloak group can see this document).
3. A user logs into Open WebUI via Keycloak SSO. Their query goes to LiteLLM, which routes to the RAG pipeline; retrieval is filtered by the user's group membership before results ever reach the model.
4. The LLM (served by Ollama or vLLM) generates a grounded answer with citations back to source documents. Presidio/LLM Guard scan the exchange; Langfuse logs the full trace for later review.

### Example B: Build-your-own classifier — e.g., "will this customer churn in the next 30 days"

This is a fully self-serve workflow for an internal developer/data scientist, start to finish, entirely inside this stack:

1. **Data.** Historical customer data (usage events, support tickets, billing history, plan changes) lives in PostgreSQL — either the org's operational data landing there directly, or synced in from other internal systems via Airflow.
2. **Labeling & features.** A Dagster/Airflow pipeline builds the training set: for each customer-month in history, compute features (recency/frequency of usage, ticket count, tenure, plan tier, payment failures, etc.) and a label — did that customer churn within the next 30 days, based on what actually happened afterward.
3. **Training.** The developer trains an XGBoost or scikit-learn model against that feature set — locally or in a scheduled pipeline job — and logs every run to **MLflow** (parameters, precision/recall/AUC, the trained artifact itself). Once a run looks good, they promote it in the MLflow model registry (e.g., `churn-classifier: v7 → Production`).
4. **Serving as an API.** The production model is wrapped in a small **FastAPI** service that loads the current MLflow-registered model and exposes an endpoint other systems can push data to, for example:

   ```python
   # churn-scoring-api/main.py
   from fastapi import FastAPI, Depends
   import mlflow.pyfunc

   app = FastAPI()
   model = mlflow.pyfunc.load_model("models:/churn-classifier/Production")

   @app.post("/v1/churn/score")
   def score(customer: CustomerFeatures, user=Depends(verify_oidc_token)):
       proba = model.predict_proba([customer.to_row()])[0][1]
       return {"customer_id": customer.customer_id,
               "churn_probability_30d": round(float(proba), 4),
               "model_version": "v7"}
   ```

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
| Authentik | SSO / IdP alternative | MIT |
| OpenBao | Secrets management | MPL 2.0 |
| PostgreSQL + pgvector | Relational + vector DB | PostgreSQL License / PostgreSQL License |
| Qdrant | Dedicated vector DB (optional) | Apache 2.0 |
| MinIO | Object storage | AGPLv3 |
| Redis | Cache / queue | RSALv2/SSPLv1 (community edition — verify current terms) |
| Ollama | LLM serving (simple) | MIT |
| vLLM | LLM serving (high-throughput) | Apache 2.0 |
| llama.cpp | LLM inference engine | MIT |
| LiteLLM | Unified model API gateway | MIT |
| Open WebUI | Chat UI | BSD-3-Clause |
| LangChain / LlamaIndex | RAG/agent frameworks | MIT |
| Haystack | RAG framework alternative | Apache 2.0 |
| LangGraph | Agent orchestration | MIT |
| Flowise | Low-code agent builder | Apache 2.0 |
| n8n | Workflow automation | Sustainable Use License (source-available, not OSI — verify fit) |
| scikit-learn / XGBoost / LightGBM | Classical ML | BSD-3-Clause / Apache 2.0 |
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

*Redis's license changed in 2024; some deployments use the community fork **Valkey** (BSD-3, Linux Foundation) instead — verify current terms before relying on this component, and swap to Valkey if strict OSI compliance is required. Similarly, n8n's license is source-available rather than a standard OSI license — Flowise or LangGraph are the fully OSI-licensed alternatives if that distinction matters for the org's policy.*

---

## 7. Suggested rollout phasing

**Phase 1 — Platform foundation.** OS, Docker, Traefik, Keycloak, Postgres, MinIO, backups. Get SSO and one internal app (even something simple) working end to end before adding AI components.

**Phase 2 — Chat + RAG pilot.** Ollama with a small model, Open WebUI, LiteLLM, ingest one document set into pgvector. Prove the RAG use case with a small user group.

**Phase 3 — Harden and scale inference.** Add GPU hardware if usage justifies it, move to vLLM, add Langfuse/guardrails, expand document ingestion pipelines, roll out to the full org.

**Phase 4 — Add classical ML.** Bring in MLflow, Airflow/Dagster, and the first classification/prediction use case, sharing the platform layer already built.

**Phase 5 — Mature ops.** OpenBao for secrets, Wazuh/Trivy for security posture, and — only if the org has genuinely outgrown 2 servers — evaluate migrating orchestration to k3s.

---

## 8. Honest tradeoffs of going fully open source and self-hosted

This is worth stating plainly rather than glossing over: this stack trades convenience for control. There is no vendor SLA — the org's own team owns uptime, patching, and disaster recovery. Model quality on a locally-hosted 8B–70B model, even a good one, will generally trail the current frontier hosted models (GPT/Claude/Gemini-class) on the hardest reasoning tasks, though it's often more than sufficient for internal RAG, classification, and structured business tasks. And the up-front engineering time to wire all of these components together correctly is real — this is a multi-week build for a competent small team, not a weekend project, even though the recurring cost afterward is just hardware and power.

What the org gets in exchange: no per-token or per-seat costs at scale, no data ever leaving the building, full control over model choice and update timing, and no dependency on any single vendor's continued existence or pricing decisions.
