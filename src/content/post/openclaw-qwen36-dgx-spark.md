---
title: "OpenClaw + Qwen3.6, Running Locally on DGX Spark"
description: "Setting up OpenClaw on a DGX Spark with Qwen3.6-35B-A3B running fully locally on vLLM. MTP speculative decoding gets it to 93 tok/s, and the speed turned out to be the easy part."
publishDate: "2026-10-13"
tags: ["ai", "agents", "inference", "nvidia"]
---

I set up a local agent on my DGX Spark: OpenClaw on top, Qwen3.6-35B-A3B served by vLLM underneath. Short version: 92.6 to 93.0 tok/s single stream with MTP speculative decoding, 62.3 without it. Speculation buys about 1.5x. The reason it works at all is the same one from my Gemma 4 benchmarks. Decode is a memory bandwidth problem, and a model that activates roughly 3B parameters per token does not move much memory.

The workload I have in mind is polling: check a page or a file every few minutes and do a small thing when something changes, all day. On a cloud model that burns paid tokens forever. Locally it burns electricity I am already paying for. It also decides what the agent can touch. A 35B local model in an open harness does not get my email or my accounts. It works with files I give it and the public web, and anything it drafts waits for my approval. That is the whole trust model.

- **Model**: Qwen3.6-35B-A3B, NVIDIA's NVFP4 build
- **Serving**: vLLM v0.28.0 in Docker, FP8 KV cache, 262K context
- **Agent**: OpenClaw 2026.10.3, local endpoint only
- **Benchmarks**: single stream, 1024 token generations, reasoning tokens counted

---

The recipe is NVIDIA's playbook, with two forced changes. Its GPU memory setting of 0.4 leaves too little KV cache at 262K context and the engine refuses to start; 0.5 works. Docker GPU access needs `--gpus all`, because the CDI flag the docs prefer does not work here.

| Config | Output tok/s |
| --- | --- |
| NVFP4, no speculation | 62.3 |
| NVFP4, MTP-3 | **92.6 to 93.0** |

Three warm runs, all within half a token of each other. A 512 token spot check later gave 86.7, so call real usage 85 to 95.

The number floating around for this config is roughly 105 tok/s. I got 93. That is about ten percent under, and under the 100 bar I had set for myself. It is also three times what I called unusable in the Gemma post, and for short mechanical polling generations I doubt the gap matters.

What I have not measured is quality. I haven't run SWE-bench on this setup, or any task suite of my own. The vendor numbers for this model sit in the low 70s on SWE-bench Verified, and vendor numbers are vendor numbers. Until I have my own data, it only gets work I can verify afterwards.

The agent itself passed a basic smoke test: it summarized a file from its workspace, knew it was running on the local model, and still worked after a vLLM restart.

For now, my advice is the NVFP4 weights with MTP on, and expect about 90 rather than 105. The speed question is settled. The trust question is not.
