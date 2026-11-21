# Projects

Six projects, each one builds on the last. Each has a scope, the stretch goals worth doing, and the acceptance criteria that count as "done". Back to the [roadmap](README.md).

---

## P1: SGEMM ladder (Phase 1, ~3 weeks)

Hand-written FP32 matmul, optimized step by step, each step profiled. This one kernel teaches most of CUDA performance.

**Steps** (each a separate kernel, benchmarked):
1. Naive (one thread per output element)
2. Coalesced global memory access
3. Shared-memory tiling
4. 1D, then 2D register blocking (several outputs per thread)
5. Vectorized loads (`float4`), transposed A tile
6. Warp tiling
7. Double buffering with `cp.async`
8. Stretch: FP16/BF16 tensor-core version with WMMA/MMA, compared against cuBLAS HGEMM

**Acceptance:**
- [ ] Correct against cuBLAS for square and non-square shapes
- [ ] ≥ 80% of cuBLAS SGEMM at M=N=K=4096
- [ ] A GFLOP/s table for every step, plus the Nsight Compute limiter (memory, latency, compute) for each
- [ ] Write-up: why each step helped, in roofline terms

## P2: Fused kernels (Phase 1, ~4 weeks)

The kernels that dominate a transformer outside GEMM, in CUDA and in Triton.

**Scope:**
- Online softmax (single pass), RMSNorm, fused RoPE
- FlashAttention-style forward pass: tiled Q·Kᵀ, online softmax, ·V, no materialized attention matrix
- Decode attention over a long KV cache (split-K / flash-decoding)

**Acceptance:**
- [ ] Matches PyTorch to within tolerance in FP16/BF16
- [ ] Attention kernel memory use stays linear in sequence length
- [ ] Bandwidth-bound kernels reach ≥ 70% of peak memory bandwidth
- [ ] CUDA vs Triton comparison: lines of code, time to write, speed

## P3: `spinfer`, an LLM inference engine in C++/CUDA (Phase 2, ~10 weeks)

A small, readable serving engine for Llama-family models, built the way `modern_exchng_cpp` was built: fixed pools, pinned threads, measured end to end.

**Architecture:**
```
HTTP (OpenAI-compatible) → request queue (SPSC) → scheduler thread (pinned)
    → batch builder → GPU worker (CUDA graphs) → detokenize → stream out
                    ↘ block allocator (paged KV) ↗
```

**Milestones:**
1. Batch-1 generation with a contiguous KV cache, reusing P1/P2 kernels or cuBLAS
2. Paged KV cache: block table, fixed-size block pool, no per-request `cudaMalloc`
3. Continuous batching: requests join and leave at every iteration
4. Chunked prefill so long prompts don't stall decode
5. INT8 then INT4 weight-only quantization
6. Prefix caching with a radix tree
7. CUDA graphs for decode steps
8. Load generator with Poisson arrivals and realistic prompt/output length distributions

**Acceptance:**
- [ ] Serves Llama-3.2-1B (and one 7–8B model on a large enough GPU) with greedy output matching the reference
- [ ] Metrics: TTFT, TPOT, tokens/s, p50/p99 latency against request rate
- [ ] A comparison table against llama.cpp and vLLM on the same GPU and workload
- [ ] KV memory waste below 5% under mixed-length load
- [ ] Monitored with triangulator during load tests (scheduler thread, CPU-side stalls)

## P4: Upstream performance PR (Phase 3, 6–8 weeks)

**Process:**
1. Build vLLM (or SGLang / llama.cpp) from source and pin a benchmark (e.g. `benchmark_serving`) on fixed hardware
2. Profile: Nsight Systems for GPU gaps and launch overhead; `py-spy`, `perf` and triangulator for scheduler, tokenizer and CPU-side stalls
3. Pick a target: an open perf issue, a hot path in the scheduler or block manager, a kernel for a missing shape or dtype, CPU overhead between decode steps
4. Reproduce, fix, measure, submit, then respond to review

**Acceptance:**
- [ ] At least one merged PR with a measured improvement
- [ ] A public write-up with profiles before and after

**Where CPU-side expertise pays off:** at high batch sizes and small models, the gap between GPU kernels is often CPU work (scheduling, sampling, detokenization, Python). That is the same thread-level latency problem triangulator was built for.

## P5: Tensor parallelism in `spinfer` (Phase 4, ~4 weeks)

**Scope:**
- Megatron-style split: column-parallel QKV/up projections, row-parallel output/down projections, one all-reduce per block
- NCCL for collectives; also implement a simple ring all-reduce yourself to understand it
- Measure all-reduce time against compute time per layer

**Acceptance:**
- [ ] Output identical to single-GPU
- [ ] Scaling efficiency reported for 1 → 2 (→ 4) GPUs, with the communication share per step
- [ ] `nccl-tests` bandwidth numbers for the rented node, compared with what `spinfer` actually achieves

## P6: Cache-aware request router (Phase 4, ~4 weeks)

A router in front of several `spinfer` (or vLLM) replicas.

**Scope:**
- Policies: round-robin, least-outstanding-tokens, prefix-cache affinity (hash of prompt prefix → replica), hybrid
- Health checks, draining, failover
- Stretch: router metadata replicated with libvsr; deterministic simulation of replica failures and network faults, as in libvsr's simulator

**Acceptance:**
- [ ] On a shared-system-prompt workload, prefix-affinity routing beats round-robin on TTFT and throughput, with numbers
- [ ] Losing a replica mid-load causes no lost requests beyond the ones in flight
- [ ] A fault-injection test suite
