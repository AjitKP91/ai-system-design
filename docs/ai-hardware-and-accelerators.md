# AI Hardware and the 2026 Accelerator Landscape

> Companion page for the [AI Hardware: GPU, TPU, and LPU](../README.md#ai-hardware-gpu-tpu-and-lpu) section of the AI System Design guide. The main section keeps things at the "CPU vs GPU vs TPU vs LPU" altitude. This page adds the fuller comparison table and the deeper "why" behind inference bottlenecks.

## 1. Why Inference Is Memory-Bound (and When It Isn't)

The single most useful mental model for AI hardware is **arithmetic intensity**: how many math operations (FLOPs) a workload does for every byte it reads from memory.

- **Low arithmetic intensity** (few FLOPs per byte) → the chip spends its time waiting on memory → **memory-bandwidth-bound**.
- **High arithmetic intensity** (many FLOPs per byte) → the chip spends its time computing → **compute-bound**.

The **roofline model** draws this as two lines: a sloped "memory" line and a flat "compute" line. Where they meet is the **ridge point**. A workload to the left of the ridge is limited by memory bandwidth; to the right, by raw compute.

LLM inference straddles the ridge because it has two phases:

| Phase | What happens | Bottleneck |
| --- | --- | --- |
| **Prefill** | All prompt tokens are read and processed in parallel. Lots of math per byte of weights read. | **Compute-bound** |
| **Decode** | Tokens are generated one at a time. At batch size 1, the entire weight set is re-read from memory to produce a single token - almost no math per byte. | **Memory-bandwidth-bound** |

**Batching moves you across the ridge.** When many users are served at once, a single weight read is reused to produce one token for each user in the batch. That raises arithmetic intensity, so decode drifts from memory-bound toward compute-bound. This is why:

- GPUs **batch aggressively** to stay efficient (great for throughput, at some cost to single-user latency).
- LPUs like Groq **skip batching** and instead keep weights in ultra-fast on-chip SRAM, winning on single-user latency.

## 2. Bandwidth vs Capacity: Two Different Memory Limits

The main section talks about memory **bandwidth** (how fast weights can be read). There is a second, independent limit: memory **capacity**.

- **Bandwidth** sets how fast each token is produced.
- **Capacity** sets how much can be held at once - the model weights **plus the KV cache**, which grows with context length and the number of concurrent users.

The KV cache can exhaust HBM capacity before bandwidth ever becomes the problem. This is why long-context and many-user serving are hard, and why techniques like **KV cache compression**, **paged attention**, and **Flash Attention** matter. (All covered in the main guide.)

## 3. The 2026 Accelerator Landscape

Approximate figures; verify against vendor datasheets before using in production decisions.

| Accelerator | Type | Memory / chip | Bandwidth | Best for |
| --- | --- | --- | --- | --- |
| **NVIDIA H100** | GPU | 80 GB HBM3 (94 GB NVL) | 3.35 TB/s (3.9 NVL) | Default training + inference |
| **NVIDIA H200** | GPU | 141 GB HBM3e | 4.8 TB/s | Larger models, more KV cache headroom |
| **NVIDIA B200 (Blackwell)** | GPU | ~180-192 GB HBM3e | ~8 TB/s | Current-gen frontier training/inference |
| **NVIDIA GB200** | Grace CPU + 2x Blackwell | superchip | — | Rack-scale deployments |
| **AMD Instinct MI300X** | GPU | 192 GB HBM3 | 5.3 TB/s | Largest memory; NVIDIA alternative |
| **AMD Instinct MI325X / MI355X** | GPU | 256 GB / 288 GB HBM3e | ~6 / ~8 TB/s | Memory-heavy inference |
| **Google TPU** | ASIC | tens of GB HBM | high | Training/inference on Google Cloud |
| **AWS Trainium2** | ASIC (training) | large HBM | high | Cost-driven training (runs Claude) |
| **AWS Inferentia** | ASIC (inference) | HBM | high | Cost-driven inference on AWS |
| **Groq LPU** | Inference ASIC | ~230 MB SRAM | ~80 TB/s on-chip | Lowest single-user latency |
| **Cerebras WSE-3** | Wafer-scale | tens of GB on-wafer SRAM | ~PB/s | Extreme-latency inference / training |
| **Intel Gaudi 3** | ASIC | ~128 GB HBM2e* | ~3.7 TB/s* | Cheaper NVIDIA alternative |
| **Apple Silicon + MLX** | Unified memory | shared with system RAM | moderate | On-device / local inference |

\* Intel Gaudi 3 figures are approximate and were not independently verified for this page - confirm at intel.com.

## 4. How to Choose

1. **Training or heavy experimentation, many model types:** GPU (NVIDIA default; AMD Instinct as the main alternative) or Google TPU on GCP.
2. **Cost-driven cloud inference at scale:** AWS Inferentia/Trainium or AMD Instinct.
3. **Lowest single-user latency on a fixed model** (chat, voice, agents, reasoning): Groq LPU or Cerebras.
4. **Local / on-device:** Apple Silicon + MLX, or llama.cpp on a GPU/CPU.

The deeper lesson holds across all of them: **once a workload is memory-bound, the win comes from moving data less, not computing more.**

## Sources

- NVIDIA Hopper / H100 / H200: en.wikipedia.org/wiki/Hopper_(microarchitecture), nvidia.com/en-us/data-center/h100/
- NVIDIA Blackwell / DGX B200: nvidia.com/en-us/data-center/dgx-b200/
- AMD Instinct: en.wikipedia.org/wiki/AMD_Instinct
- Groq LPU: groq.com/the-groq-lpu-explained/, newsletter.semianalysis.com/p/groq-inference-tokenomics-speed-but
- Cerebras: en.wikipedia.org/wiki/Cerebras
- AWS Trainium/Inferentia: en.wikipedia.org/wiki/AWS_Trainium
