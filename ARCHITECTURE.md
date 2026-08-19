# GitHub Documentation - ARCHITECTURE.md
## System Architecture & Data Flow Blueprint (ARCHITECTURE.md)
This document provides an exhaustive breakdown of the multi-layered system architecture for the low-cost, open-source AI infrastructure stack. By separating system responsibilities into individual components, this structure ensures long-term modularity, horizontal scalability, and strict data sovereignty.
------------------------------
## 1. Architectural Core Principles
To keep the platform cost-efficient and production-ready for small to mid-sized businesses, the entire infrastructure follows three mandatory structural design principles:

* Decoupling of Compute and Orchestration: The user interfaces and orchestration frameworks operate independently from raw hardware model runners. This allows compute infrastructure to be scaled up or down dynamically without modifying client applications.
* Unified API Abstraction Gateway: Every foundational model backend is mapped and routed through a single gateway proxy layer. This forces all internal downstream business tools to use a standardized, uniform connection interface.
* Strict Network Isolation (Air-Gapping Ready): Every single data stream, prompt sequence, document index, and model computation log is kept strictly within localized network networks, preserving data residency guarantees.

------------------------------
## 2. Component Execution Table
The execution flow moves top-down, passing client queries seamlessly down through abstraction layers to reach the dedicated hardware:

| Execution Level | Layer Designation | Primary Technology | Core Function & Operational Role |
|---|---|---|---|
| Level 1 | User Interface (UI) | Open WebUI | Captures user queries, runs multi-user permissions, routes user sessions, and displays real-time text streaming outputs. |
| Level 2 | Application Middleware | LangChain / LlamaIndex / CrewAI | Manages multi-agent execution states, builds system prompts, injects memory context, and manages workflow loops. |
| Level 3 | Semantic Memory | PostgreSQL + pgvector / Qdrant | Indexes and stores unstructured enterprise data as mathematical vector embeddings for rapid semantic query matching. |
| Level 4 | Gateway & Proxy | LiteLLM | Acts as a central traffic cop to unify load-balancing, access tokens, and performance metrics across different inference endpoints. |
| Level 5 | Inference Engine | Ollama / vLLM | Compiles mathematical neural weights, handles key-value caching, and optimizes execution directly on the GPU VRAM. |
| Level 6 | Compute Infrastructure | Local Hardware nodes / Ray clusters | Provides physical graphics hardware processing, system memory pools, and containerized runtimes. |

------------------------------
## 3. End-to-End System Pipelines
The open-source stack coordinates two foundational data pipelines to execute enterprise workflows safely:
## Pipeline A: Corporate Document Ingestion (Knowledge Base Sync)
This offline background pipeline transforms static, unstructured multi-format files into searchable semantic memory:

   1. Document Parsing: Local files (PDFs, Markdown assets, text wikis) are extracted via LangChain document loaders.
   2. Deterministic Chunking: Raw text streams are segmented into clean, predictable lengths (e.g., 512 or 1024 tokens) using specialized character-splitting algorithms to protect semantic boundaries.
   3. Vector Transformation: Text blocks are forwarded to a local model running on Ollama (such as nomic-embed-text) to create corresponding high-dimensional mathematical coordinates.
   4. Database Retention: The text chunks, along with metadata pointers, are saved into the pgvector or Qdrant storage schema for immediate production use.

## Pipeline B: Runtime User Query & RAG Synthesis
This synchronous execution pipeline runs instantly whenever an authorized user submits a new prompt:

   1. User Request Input: An authenticated employee types a prompt into the Open WebUI browser dashboard.
   2. Context Retrieval: The Orchestration layer vectorizes the input query, queries the vector storage system via an internal lookup, and pulls the most relevant matching text fragments.
   3. Context Injection: The orchestration framework drops these verified document fragments directly into an augmented system prompt context template.
   4. Gateway Transmission: The structured prompt package is transmitted to LiteLLM, which load-balances the payload across available local vLLM or Ollama computational engines.
   5. Token Generation & Stream: The inference hardware executes the context-rich prompt and streams text chunks chronologically back to Open WebUI to display the response instantly.


