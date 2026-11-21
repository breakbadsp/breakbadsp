# Resources

A short list, grouped by phase. Read the ones marked ★ first. Back to the [roadmap](README.md).

## Phase 0: Model internals

- ★ Andrej Karpathy, [Neural Networks: Zero to Hero](https://karpathy.ai/zero-to-hero.html), especially "Let's build GPT"
- ★ [llm.c](https://github.com/karpathy/llm.c): GPT-2 training in plain C/CUDA, good for reading
- Horace He, [Making Deep Learning Go Brrrr From First Principles](https://horace.io/brrr_intro.html): compute vs memory vs overhead
- [How to Scale Your Model](https://jax-ml.github.io/scaling-book/): rooflines, FLOP and memory accounting for transformers (TPU-centred, but the math carries over)

## Phase 1: GPU and CUDA

- ★ Hwu, Kirk, El Hajj, *Programming Massively Parallel Processors* (4th ed.)
- ★ Simon Boehm, [How to Optimize a CUDA Matmul Kernel for cuBLAS-like Performance](https://siboehm.com/articles/22/CUDA-MMM): the template for P1
- ★ [GPU MODE lectures](https://github.com/gpu-mode/lectures) and their Discord community
- NVIDIA [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/) and [Best Practices Guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/)
- [Triton tutorials](https://triton-lang.org/main/getting-started/tutorials/index.html): fused softmax, matmul, FlashAttention
- [CUTLASS](https://github.com/NVIDIA/cutlass) and its CuTe docs, once WMMA feels limiting
- Nsight Compute and Nsight Systems user guides (the "stall reasons" section)

## Phase 2: Inference engines

Papers, in order:
- ★ Orca, continuous batching (OSDI '22)
- ★ [FlashAttention](https://arxiv.org/abs/2205.14135), [FlashAttention-2](https://arxiv.org/abs/2307.08691), [FlashAttention-3](https://arxiv.org/abs/2407.08608)
- ★ [PagedAttention / vLLM](https://arxiv.org/abs/2309.06180) (SOSP '23)
- [SGLang / RadixAttention](https://arxiv.org/abs/2312.07104): prefix caching
- AWQ and GPTQ: weight-only quantization
- Speculative decoding (Leviathan et al.; Medusa; EAGLE)

Code to read:
- [llama.cpp](https://github.com/ggml-org/llama.cpp): C/C++, easiest to follow end to end
- [vLLM](https://github.com/vllm-project/vllm): scheduler, block manager, attention backends
- [SGLang](https://github.com/sgl-project/sglang): radix cache, scheduler
- Nano-scale engines (e.g. nano-vllm, mini-sglang): small enough to read in an evening

## Phase 4: Distributed

- ★ [Megatron-LM](https://arxiv.org/abs/1909.08053): tensor parallelism
- ★ [DistServe](https://arxiv.org/abs/2401.09670) and Splitwise: disaggregated prefill/decode
- [nccl-tests](https://github.com/NVIDIA/nccl-tests) and the NCCL docs
- Mooncake (Moonshot AI): KV-cache-centric serving architecture
- NVIDIA Dynamo and llm-d: production disaggregated serving and routing stacks to compare P6 against

## Phase 5: Training (optional)

- ★ Hugging Face, [The Ultra-Scale Playbook](https://huggingface.co/spaces/nanotron/ultrascale-playbook)
- ZeRO (DeepSpeed) and PyTorch FSDP papers

## Keeping current

- vLLM and SGLang release notes and engineering blogs
- GPU MODE talks
- Tri Dao, Horace He, and the PyTorch team's blogs
