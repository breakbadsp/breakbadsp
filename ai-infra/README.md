# AI infra path: from low-latency trading systems to LLM inference infrastructure

**Goal:** build and speed up LLM inference infrastructure (kernels, serving engines, distributed serving), with public, measured work to show for it.

**Starting point:** C++23/26 and Rust, lock-free queues, pinned cores, `epoll`/`io_uring`, custom allocators, `perf`, consensus ([libvsr](https://github.com/breakbadsp/libvsr)), and thread-level observability ([triangulator](https://github.com/breakbadsp/triangulator)). Stage 1 of most AI infra roadmaps (systems foundations) is already done. The gap is **GPUs and model internals**.

**Budget:** about 8–10 hours a week, roughly 10–12 months. Every phase ends with a repo that has a benchmark next to the code, the same rule as the trading repos.

---

## The map

| Phase | Focus | Weeks | Ships |
|:--|:--|:--:|:--|
| 0 | Model internals and the cost model | 3 | Notes + a C++ CPU forward pass |
| 1 | GPU architecture and CUDA | 8 | **P1** SGEMM ladder, **P2** fused kernels |
| 2 | Inference engine from scratch | 10 | **P3** `spinfer` |
| 3 | Upstream performance work | 6–8 | **P4** a merged perf PR in vLLM / SGLang / llama.cpp |
| 4 | Distributed serving | 8 | **P5** tensor parallelism in `spinfer`, **P6** cache-aware router |
| 5 | Training infra (optional) | open | depends on the target role |

Project specs with acceptance criteria are in [projects.md](projects.md). Reading lists are in [resources.md](resources.md).

---

## Phase 0: Model internals and the cost model (3 weeks)

You can't optimize what you can't do on paper. By the end, you can estimate an LLM's FLOPs, bytes moved and memory footprint without running it.

- [ ] Transformer forward pass: embeddings, RMSNorm, attention (MHA → GQA), RoPE, SwiGLU MLP, sampling
- [ ] Why **prefill is compute-bound and decode is memory-bandwidth-bound**, and what that means for batching
- [ ] KV cache size: `2 × layers × kv_heads × head_dim × bytes × tokens`. Work it out for Llama-3-8B at 8K context
- [ ] Roofline model: arithmetic intensity, ridge point, and where GEMM, attention and decode sit
- [ ] **Build:** a single-threaded C++ forward pass for a tiny Llama-style model (e.g. stories15M), checked token-for-token against a PyTorch reference

**Done when:** you can explain from memory why batch-1 decode on an H100 is far below peak FLOPs.

## Phase 1: GPU architecture and CUDA (8 weeks)

The core gap. Treat the GPU like you treated the CPU cache hierarchy: measure first, then reason.

- [ ] Execution model: SMs, warps, SIMT, divergence, occupancy, latency hiding
- [ ] Memory hierarchy: HBM → L2 → shared memory/L1 → registers; coalescing; bank conflicts
- [ ] Profiling: **Nsight Systems** (timeline) and **Nsight Compute** (per-kernel roofline, stall reasons)
- [ ] Tensor cores: WMMA/MMA, then CUTLASS/CuTe concepts
- [ ] Async copies (`cp.async`, TMA on Hopper), pipelining, warp specialization
- [ ] Triton: write the same kernels in Triton and compare effort against speed
- [ ] **P1** SGEMM ladder → **P2** fused softmax, RMSNorm and FlashAttention-style forward pass

**Done when:** P1 reaches ≥ 80% of cuBLAS for FP32 SGEMM at 4096², and you can read a Nsight Compute report and name the limiter.

**Trading-systems overlap:** coalescing works like cache-line-aware layout, shared memory is a software-managed L1, and warp divergence is the branch mispredict of SIMT.

## Phase 2: Inference engine from scratch (10 weeks)

Build a small engine before reading vLLM. Once you've hit the problems yourself, the vLLM codebase is easy to follow.

- [ ] Weight loading (safetensors, mmap), tokenizer integration
- [ ] KV cache: contiguous first, then **paged** (block tables, a block allocator)
- [ ] Scheduler: static batching → **continuous batching** (iteration-level scheduling)
- [ ] Prefill/decode split, chunked prefill
- [ ] Weight-only quantization (INT8, then INT4 / AWQ-style)
- [ ] Prefix caching (radix-tree reuse, the SGLang idea)
- [ ] CUDA graphs for decode to remove launch overhead
- [ ] OpenAI-compatible HTTP endpoint and a load generator
- [ ] **P3** `spinfer`

**Done when:** `spinfer` serves Llama-3.2-1B with continuous batching and paged KV, and has a published TTFT / TPOT / throughput / p99 table against llama.cpp and vLLM on the same GPU.

**Trading-systems overlap:** the request path is an order path. Gateway → queue → pinned scheduler thread → GPU worker is the same pipeline as `modern_exchng_cpp`. Tail latency, batching against latency, and allocation-free hot paths are the same problems.

## Phase 3: Upstream performance work (6–8 weeks, then ongoing)

The strongest hiring signal is a merged performance PR in a project people run in production.

- [ ] Set up vLLM (or SGLang / llama.cpp) from source with a reproducible benchmark
- [ ] Profile the GPU side with Nsight Systems, and the CPU side (scheduler loop, detokenization, Python overhead) with `py-spy`, `perf` and **triangulator**
- [ ] Read open perf issues, pick one, reproduce it, measure it
- [ ] **P4** land the PR, and write it up with before/after numbers

**Done when:** at least one perf PR is merged upstream.

## Phase 4: Distributed serving (8 weeks)

- [ ] Collectives: all-reduce, all-gather, reduce-scatter; ring vs tree; NCCL and `nccl-tests`
- [ ] Interconnects: NVLink, PCIe, InfiniBand/RoCE, RDMA basics, GPUDirect
- [ ] Parallelism for inference: tensor, pipeline, expert (MoE), data
- [ ] Disaggregated prefill/decode and KV-cache transfer between nodes
- [ ] Request routing: prefix-cache affinity, load-aware placement, autoscaling signals
- [ ] **P5** 2-GPU tensor parallelism in `spinfer` → **P6** cache-aware router

**Done when:** `spinfer` runs a model split across 2 GPUs with measured scaling efficiency, and P6 beats round-robin routing on a shared-prefix workload.

**Trading-systems overlap:** libvsr experience (replication, failure handling, deterministic simulation) carries over directly to router state, failover and fault-injection testing of a serving fleet.

## Phase 5: Training infra (optional)

Only if the target role needs it: FSDP/ZeRO, activation checkpointing, Megatron-style 3D parallelism, checkpointing at scale, failure detection on large clusters. Start from the Ultra-Scale Playbook in [resources.md](resources.md).

---

## Hardware

- **Phases 0–3:** one NVIDIA GPU. A consumer RTX card (Ampere or newer) covers almost everything. Rent a datacenter GPU (A100/H100) by the hour when you need tensor-core or HBM numbers that match production.
- **Phase 4:** rent a multi-GPU node for focused sessions. Script the setup so a session starts benchmarking within minutes.

## Rules for every project

1. A benchmark next to the code, with hardware, driver/CUDA version and commands written down.
2. A correctness check against a reference (cuBLAS, PyTorch, llama.cpp) before any speed claim.
3. A Nsight report or flame graph backing each optimization.
4. A short write-up: what was slow, why, what changed, the numbers.

## Progress log

| Date | Phase | Milestone | Link |
|:--|:--|:--|:--|
| 2026-11 | — | Path written | this file |
