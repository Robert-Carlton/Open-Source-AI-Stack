# GitHub README - Open Source AI Stack
## Low-Cost, Fully Open-Source AI Stack for SMBs (2026 Blueprint)
This repository provides a comprehensive architectural blueprint for small to mid-sized businesses (SMBs) looking to deploy a fully sovereign, enterprise-grade Artificial Intelligence stack without incurring prohibitive per-token licensing fees or sacrificing data privacy.
By leveraging mature open-source technologies under permissive commercial licenses (such as Apache-2.0 and MIT), businesses can host their own foundational models, document search systems, and automated agent pipelines either locally or via highly economical, on-demand compute infrastructure.
------------------------------
## 🏗️ Architectural Overview
The stack is designed with a layered approach, ensuring a clean separation of concerns between raw compute orchestration, model serving, internal data vectorization, and user-facing applications.

| Execution Layer | Layer Name | Associated Technologies | Architectural Data Flow Role |
|---|---|---|---|
| Level 1 | User Interface (UI) | Open WebUI / Enterprise Chat | Captures incoming user prompts, manages session history, and displays real-time streaming tokens. |
| Level 2 | Application & Orchestration | LangChain / LlamaIndex / CrewAI | Intercepts queries, manages multi-agent states, and handles system prompt compilation. |
| Level 3 | Data & Vector Storage | PostgreSQL + pgvector / Qdrant | Hosts the corporate knowledge base, storing and indexing semantic text embeddings. |
| Level 4 | Inference & Gateway | LiteLLM / Ollama / vLLM | Unifies backend endpoints, handles API routing rules, and runs the core LLM execution engine. |
| Level 5 | Compute Infrastructure | Local Hardware / Kubernetes / Ray | Orchestrates the physical hardware nodes, VRAM pools, and baseline container runtimes. |

------------------------------
## 📊 Core Tech Stack Components

| Layer | Component / Tool | Open Source License | Primary SMB Advantage |
|---|---|---|---|
| User Interface | Open WebUI | MIT | Seamless, ChatGPT-like multi-user environment with full self-hosted identity management and direct integration with local pipelines. |
| Agentic Orchestration | LangChain, LlamaIndex, CrewAI | MIT / Apache-2.0 | Standardized abstractions for chaining LLMs together, executing complex data retrieval workflows (RAG), and running multi-agent squads. |
| Data & Retrieval | PostgreSQL with pgvector | PostgreSQL License / MIT | Allows utilizing existing database hardware and engineering knowledge to store embeddings, reducing infrastructural sprawl. |
| Specialized Vector Storage | Qdrant | Apache-2.0 | Ultra-fast semantic search capabilities with a low memory footprint and fully accessible free tiers. |
| Inference Serving | Ollama | MIT | Single-command installation engine for local model optimization, managing hardware memory pools automatically. |
| Production Inference | vLLM | Apache-2.0 | High-throughput, distributed serving framework for production setups requiring simultaneous request processing. |
| Model Management / Gateway | LiteLLM | MIT | Acts as a central proxy to unify distinct model backends under a single OpenAI-compatible API dashboard. |
| Underlying Models | Llama 3 / 3.1, Mistral, Qwen | Permissive / Commercial-use friendly | Highly capable foundation models reaching near-parity with premium closed-source alternatives at zero licensing expense. |

------------------------------
## 💻 Hardware Requirements & Cost Efficiency
Running an open-source AI stack eliminates variable token usage fees, shifting the financial model entirely to fixed hardware or hosting utility costs. Businesses can choose between two main infrastructure paths:
## 1. 100% Private Local Deployment

* Target Hardware: A standard workstation PC equipped with a mid-range consumer GPU (e.g., NVIDIA RTX series with 16GB+ VRAM) or an Apple Silicon Mac Studio with unified memory.
* Capacity: Comfortably serves 8B to 13B parameter models (such as Llama 3 or Qwen) for local team requests.
* Software Footprint: Ollama + Open WebUI running in docker containers.
* Estimated Software Cost: $0.

## 2. Cloud-Hybrid or Dedicated Server Bursting

* Target Hardware: Single-node dedicated server hosting an enterprise accelerator card, managed through lightweight container orchestrators like Kubernetes or distributed frameworks like Ray.
* Capacity: Dynamically scales to support ultra-dense models (70B+ parameters) or specialized workloads during peak production hours.
* Consolidation Principle: To maximize operational efficiency, avoid multi-vendor API fragmentation. Route all applications through a unified abstraction proxy like LiteLLM to maintain one set of access credentials and clear usage visibility.

------------------------------
## 🚀 Quick-Start Deployment: Internal Corporate RAG
Follow these steps to deploy a localized, privacy-compliant Document Q&A setup on local hardware or an internal server:
## Step 1: Install the Local Inference Engine
Pull and launch the model runner using a containerized environment or direct execution:

# Example via local setup
curl -fsSL https://ollama.com | sh
ollama run llama3

## Step 2: Spin Up the Shared User Interface
Launch Open WebUI alongside your model deployment to give your team an accessible web panel:

docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main

## Step 3: Establish the Vector Memory Layer
Equip your database instance with pgvector or spin up a standalone instance of Qdrant to host your company's knowledge base embeddings.
## Step 4: Wire the Orchestration Pipeline
Use a Python framework like LangChain or LlamaIndex to parse local files, generate vectorized indices, and route questions to your self-hosted inference gateway.
------------------------------
## 🔒 Security, Compliance & Governance
When deploying self-hosted AI stacks within a business infrastructure, always implement the following baseline protocols:

   1. Supply Chain Documentation: Utilize vendor-neutral platforms like MLflow to track your internal experiments, evaluate models systematically, and generate appropriate Software Bills of Materials (SBOMs).
   2. Access Controls: Protect user-facing instances behind secure corporate reverse proxies, requiring OAuth2 or internal corporate directory verification.
   3. Data Residency: Ensure that all processing loops remain entirely contained within local networks or strictly audited private clouds to seamlessly satisfy data privacy regulations.



