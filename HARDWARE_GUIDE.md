# GitHub Documentation - HARDWARE_GUIDE.md
## Hardware Procurement & VRAM Sizing Guide (HARDWARE_GUIDE.md)
This document provides local artificial intelligence infrastructure procurement guidelines for small to mid-sized businesses. Because local model execution is strictly bounded by hardware memory capacity, selecting physical graphics hardware directly dictates which model families your organization can host natively. [1, 2] 
------------------------------
## 1. The Core Baseline VRAM Sizing Formula
When hosting open-source language models locally, model weights must fit entirely within your Graphic Processing Unit's onboard Video RAM (VRAM) or system Unified Memory to prevent severe performance degradation. The absolute baseline allocation is determined by this sizing formula: [1, 2, 3, 4, 5] 

Minimum VRAM Capacity = (Model Parameters in Billions * Quantization Bits / 8) + KV Cache Overhead


* Quantization Factor: Running models at native 16-bit precision requires roughly 2GB of VRAM per billion parameters. Applying 4-bit precision quantization (Q4_K_M) compresses this footprint by roughly 75% with a negligible 1-3% degradation in overall response reasoning quality. [4, 6, 7, 8, 9] 
* Key-Value (KV) Cache Overhead: Every concurrent user or extended document retrieval context window consumes extra memory allocation blocks. Allocate a protective buffer of 1GB to 4GB of VRAM exclusively for context processing overhead. [4, 6, 8, 10, 11] 

------------------------------
## 2. Infrastructure Procurement Tiers
Find your company's concurrent operational scale or target model size to determine your exact hardware specification track: [1, 2] 

| Procurement Tier | Target Model Profile | Recommended Hardware Track | Estimated System Cost Hardware Profile |
|---|---|---|---|
| Tier 1: Entry Developer Workstation | Small Coding & Chat Models (e.g., Llama 3.1 8B, Qwen 2.5 Coder 7B) | 1x NVIDIA RTX 3060 12GB or RTX 4060 Ti 16GB | $300 - $500 baseline card upgrade for existing office desktop computers. |
| Tier 2: Production Team Server | Medium Reasoning & RAG Models (e.g., Gemma 27B, Qwen 32B) | 1x Dedicated Workstation with a used or new NVIDIA RTX 3090 / 4090 / 5080 (24GB VRAM) | $1,500 - $2,500 standalone dedicated tower acting as an internal LAN server. |
| Tier 3: Sovereign Enterprise Lab | Large Comprehensive Models (e.g., Llama 3 70B, DeepSeek Variants) | Apple Silicon Mac Studio with M4 Max / Ultra (64GB, 128GB, or 192GB Unified Memory) | $2,500 - $5,000+ total turnkey device. Zero cluster assembly complexity. |
| Tier 4: Datacenter Concurrent Cluster | High-Throughput Serving (70B+ FP16 Parameters or 100+ concurrent connections) | Multi-GPU Dedicated Nodes (e.g., 2x NVIDIA H100 80GB or RTX PRO 6000 clusters) | Enterprise server rack environment. High electrical consumption and custom server room cooling required. |

------------------------------
## 3. Hardware Ecosystem Selection Trade-offs
Choosing between an NVIDIA-based CUDA workstation or an Apple Silicon Unified Memory ecosystem requires weighing distinct architectural performance behaviors: [11, 12, 13, 14] 
## Platform A: NVIDIA Graphics Infrastructure

* The Structural Advantage: Native 4th-generation Tensor Cores feature deep hardware-accelerated FP8 execution pipelines, yielding maximum token-per-second velocity for localized operations. Extremely stable runtime setups across deep production engines like vLLM. [4, 15, 16, 17, 18] 
* The Sizing Constraint: Consumer-grade hardware models hard-cap at 24GB of physical VRAM per individual graphics card. Scaling past this boundary requires configuring multi-GPU links, which increases physical tower power consumption and splits context compilation graphs. [11, 14, 19, 20, 21] 

## Platform B: Apple Silicon Unified Memory Architecture (UMA)

* The Structural Advantage: System memory maps dynamically as one massive, pooled buffer shared simultaneously between processing units. This permits a single desktop Mac Studio to load a massive 70B parameter model in its entirety, completely avoiding multi-card server infrastructure overhead. [3, 11, 14, 22, 23] 
* The Sizing Constraint: Peak execution velocity is lower compared to specialized datacenter hardware chips. To maintain high response speeds, optimization engines must leverage Apple's native machine-learning backends (such as Ollama's automatic MLX runtime integration). [11, 12, 14, 24] 

------------------------------
## 4. Mandatory Hardware Pre-Flight Checklist
Before finalizing hardware purchases for an on-site AI stack deployment, verify these final system parameters:

   1. System RAM Multiplier: Standard system motherboard RAM should scale to roughly 1.5x or 2x the capacity of your total active GPU VRAM to ensure rapid model weight caching and system swap stability. [3, 15, 18, 23] 
   2. PCIe Lane Thresholds: Motherboards for multi-GPU builds must support adequate PCIe Gen4 or Gen5 lanes (such as AMD Threadripper platforms) to prevent communication bottlenecks between separate cards. [15, 18] 
   3. Storage Performance: Model checkpoints are exceptionally heavy files (a 70B parameter model spans roughly 40GB to 140GB depending on quantization). Use dedicated NVMe M.2 SSD drives exclusively to ensure responsive initialization boot times. [1, 2, 15, 18] 


[1] [https://www.promptquorum.com](https://www.promptquorum.com/local-llms/local-llm-hardware-guide-2026)
[2] [https://www.youtube.com](https://www.youtube.com/watch?v=T7WF9okSfrw)
[3] [https://www.youtube.com](https://www.youtube.com/shorts/vdkMNy5LPho)
[4] [https://eastondev.com](https://eastondev.com/blog/en/posts/ai/20260528-ollama-hardware-guide/)
[5] [https://vrlatech.com](https://vrlatech.com/vrla-tech-workstations-ollama/)
[6] [https://www.digitalocean.com](https://www.digitalocean.com/community/conceptual-articles/vllm-gpu-sizing-configuration-guide)
[7] [https://lyceum.technology](https://lyceum.technology/magazine/how-much-vram-for-70b-model/)
[8] [https://docs.vllm.ai](https://docs.vllm.ai/en/v0.9.0/getting_started/installation/index.html)
[9] [https://sesamedisk.com](https://sesamedisk.com/llama-3-1-70b-on-rtx-3090-nvme-to-gpu/)
[10] [https://www.youtube.com](https://www.youtube.com/watch?v=QKdKcFjjZhE&t=136)
[11] [https://www.thinkdifferent.blog](https://www.thinkdifferent.blog/blog/the-truth-about-mac-vs-nvidia-for-ai-i-tested-both/)
[12] [https://ksingh7.medium.com](https://ksingh7.medium.com/i-almost-bought-an-rtx-5090-then-apples-unified-memory-changed-my-mind-83eb964e930b)
[13] [https://www.youtube.com](https://www.youtube.com/watch?v=Zp7CvD6xJYg)
[14] [https://jamesm.blog](https://jamesm.blog/ai/mac-studio-local-llm-guide/)
[15] [https://vrlatech.com](https://vrlatech.com/vrla-tech-workstations/vllm/)
[16] [https://gigagpu.com](https://gigagpu.com/rtx-3090-vs-rtx-4090-ai/)
[17] [https://itctshop.com](https://itctshop.com/can-i-run-llama-3-70b-on-rtx-4090/)
[18] [https://docs.vllm.ai](https://docs.vllm.ai/en/latest/getting_started/installation/)
[19] [https://www.youtube.com](https://www.youtube.com/watch?v=sR8sJ2mybQU)
[20] [https://www.reddit.com](https://www.reddit.com/r/LocalLLaMA/comments/1g7tsey/rtx_4090_3090_for_70b_llms_will_the_3090_hog/)
[21] [https://hardwarehq.io](https://hardwarehq.io/best-gpu-for-llama)
[22] [https://www.kunalganglani.com](https://www.kunalganglani.com/blog/running-local-llms-2026-hardware-setup-guide)
[23] [https://www.youtube.com](https://www.youtube.com/watch?v=zjG9TkfWd0o)
[24] [https://ai-tldr.dev](https://ai-tldr.dev/learn/local-open-models/running-models-locally/run-llms-on-mac/)
