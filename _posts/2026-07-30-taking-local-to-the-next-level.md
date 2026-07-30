---
title: "My Journey Running AI Locally: Taking Local to the Next Level"
date: 2026-07-30 10:00:00 +0000
categories: [AI, Local AI, Machine Learning]
tags: [local-ai, llm, vllm, lm-studio, m4-pro, dell-max-pro, mtp, speculative-decoding]
layout: post
comments: true
---

After weeks of experimenting with running AI models locally on my MacBook Pro I can share some more of my experiences and insights. I have enjoyed my setup a lot! I could for example keep working in the train without a stable internet connection! That felt like freedom. But I was curious, how far can I take this? How far can I push my local setup? Can I run even larger models locally? Can I run models that are more capable than Qwen3.6-35B-A3B? Let's find out!

Outline:
experimented with current setup, MTP, prefix-caching
Not available for MLX models in llama.cpp (and thus LM Studio)
MLX studio promises to support all this, but it crashes often on my M4 Pro due to memory usage spikes

Instead I decided to focus on getting the best performance out of my current setup. Using less tokens increases speed a lot, since 11 tokens per seconds is the max I can get.
With RTK and ThinkingCap I can get a lot more done with less tokens. I have been experimenting with different models and found that Qwen3.6-27b-ThinkingCap is the best model for my hardware.

Outcome:
- Qwen3.6-35B-A3B is the best model for easy tasks.
- Qwen3.6-27b-ThinkingCap is the best model for my hardware, with some tweaks.

Next up:
I'm experimenting with a Dell Max Pro GB10 server to see if I can run even larger models and improve performance. Stay tuned for my next post where I will share my findings and insights from this new setup.

PART 2:
vllm is dificult to set up. I couldn't get it to work on my Dell Max Pro GB10 server. A lot of errors. In the end I decided to use LM Studio on the server as well. I can run larger models on the server, but the performance is not as good as I hoped. GPT-OSS-120B and Laguna-S-2.1 seem less capable than Qwen3.6-27B-ThinkingCap. I will continue to experiment with different models and setups to see if I can improve performance and capabilities. 

So, is the GB10 server worth it? For now, I think the answer is no. The performance and capabilities of the models I can run on the server are not significantly better than what I can achieve with my M4 Pro. Mainly because in the 120b parameters class there are no new models.

## The Setup

Like described in my previous post my setup consists of:
1. **LM Studio** for model management and JIT loading of models
2. **Qwen3.6-35B-A3B** for fast, capable responses, vision capabilities, and excellent tool-calling
3. **Qwen3.6-27B** for more complex tasks that require more reasoning and understanding of code (but a lot slower than Qwen3.6-35B-A3B)
4. **MLX optimization** for maximum performance on my M4 Pro
5. **Open Code** as the agent harness, for smooth integration and execution of tasks

What have I used this setup for? I have used it for a variety of tasks, including:
- Creating this blog site
- Adding [simple features](https://github.com/stryker-mutator/stryker-net/pull/3670) to Stryker.NET
- Generating unit tests for straightforward functions
- Answering technical questions and explaining code

While it worked great on those simple tasks, for more complex tasks I have found that Qwen3.6-35B-A3B is not capable. Some examples of tasks that are too complex for this model include:
- Resolving merge conflicts in large codebases with many interdependent files
- Generating slide decks with visualizations and data
- Implementing complex features in Stryker.NET
- Debugging subtle, hard-to-reproduce bugs across multiple layers

Some other downsides of this setup include:
- **Long Load Times**: While the tokens per second are impressive, the initial load time can be minutes, which is not ideal for quick tasks.
- **Battery Life**: Running large models locally can be power-intensive, leading to shorter battery life on laptops and can be hot on your lap.

I wanted to know if I could solve these issues by some smarter software optimizations. I have been researching and experimenting with different approaches, and I have found some interesting insights that I want to share in this post.

## How LLMs Work

Before diving into hardware limits, it helps to understand what actually happens when you send a prompt to an LLM. Why can it take minutes before you see a response? Inference happens in two distinct phases, each with very different hardware requirements.

### Prefill (Prompt Processing)

When you send a prompt, the model first processes the entire input in the **prefill** phase. All tokens in your prompt are processed in parallel through the network's layers. The model computes attention scores between all token pairs and builds the KV cache, a memory structure that stores the contextual understanding of every token for use during generation.

Prefill is **compute-bound**. The heavy matrix multiplications across all layers dominate, and the GPU's raw TFLOPS determine how quickly the prompt is processed. More compute means faster prompt understanding. For short prompts this phase is nearly instantaneous, but for long contexts (tens of thousands of tokens) it can take noticeable time. The key hardware specs here are:

- **GPU Compute (TFLOPS)**: Raw floating-point throughput drives matrix multiplication speed
- **VRAM/Capacity**: The entire prompt plus the KV cache must fit in memory

When running large models like Qwen3.6-27B, the prefill phase can take a long time on my M4 Pro. Processing the system prompt in OpenCode takes a few seconds. But when I send a prompt as a continuation on a 200k token conversation, the prefill phase can take minutes. This is because the model has to process all 200k tokens to build the KV Cache, which is a lot of compute.

### Decode (Token Generation)

After prefill, the model enters the **decode** phase, generating the response one token at a time. This is the autoregressive nature of LLMs: each new token depends on all previous tokens. For every single token, the model must:

1. Read the full set of model weights from memory
2. Compute attention against the existing KV cache
3. Produce a probability distribution over the vocabulary
4. Sample the next token
5. Append it to the KV cache and repeat

Decode is **memory-bandwidth-bound**. The bottleneck isn't how fast the GPU can compute — it's how fast weights can be streamed from memory. For each token, the entire model (billions of parameters) must be read from memory. On a 35B parameter model in 4-bit quantization, that's roughly 17.5 GB read per token. At 200 GB/s bandwidth, you're looking at a theoretical ceiling of roughly 11 tokens per second, regardless of how much compute you have.

This is why memory bandwidth is the single most important spec for local LLM performance. You can have teraflops of compute sitting idle while the memory bus is maxed out streaming weights. The key hardware specs for decode are:

- **Memory Bandwidth (GB/s)**: The dominant factor — directly limits tokens/second
- **Memory Capacity (GB)**: Determines which models you can run at all
- **Compute (TFLOPS)**: Secondary — only matters once bandwidth is no longer the bottleneck

### Why This Matters

Understanding these two phases explains a lot about the local AI experience. When you paste a long document and hit enter, there's a pause (prefill), then tokens start streaming (decode). If tokens feel slow, it's almost always a memory bandwidth issue, not a compute issue. And it explains why throwing more compute at the problem often yields diminishing returns. For tokens per second the bottleneck is memory bandwidth, not compute power.

## Software Optimizations

There are a few optimizations that can help squeeze more performance out of local LLMs.

### Prompt caching

During prefill, the model computes attention scores between all tokens in the prompt. If you send the same prompt multiple times, the model can cache the KV cache for that prompt and skip re-computation. This is especially useful when using an agent harness that always sends the same system prompt. LM Studio does not currently support prompt caching. This is why the prefill phase can feel slow when using LM Studio, even for repeated prompts.

This will be especially important for long-running conversations. If you have a 200k token conversation and you want to continue it, the model has to re-process all 200k tokens to build the KV cache. With prompt caching, the model could skip this step and start generating immediately.

### Multi Token Prediction

If decode is memory-bandwidth-bound, is there any way to get more tokens per second without upgrading hardware? Enter **Multi Token Prediction** (MTP), also known as speculative decoding or speculative sampling.

The core insight behind MTP is simple but powerful: instead of generating one token at a time, generate several candidate tokens in parallel, then verify them all in one pass.

Here's the process:
1. A small **draft model** (or a distilled version of the target model) quickly proposes K candidate tokens
2. The **target model** verifies all K tokens in a single forward pass
3. Accepted tokens are appended; if any token is rejected, generation continues from the last accepted token

The magic is in step 2. Because the target model processes all K candidates in one pass (using the same memory read of the full model weights), you get up to K tokens for the cost of roughly one decode step. If the draft model is accurate, you can approach K× speedup. 

The actual speedup depends on the **acceptance rate**. This is how often the draft model's guesses match what the target model would have chosen. In practice:

- **High acceptance** (>80%): You get close to the theoretical K× speedup. With K=6, that's nearly 6 times the tokens per second.
- **Moderate acceptance** (50-80%): Still meaningful speedup, typically 2-3 times the tokens per second.
- **Low acceptance** (<50%): The overhead of running the draft model may not be worth it

Acceptance rate depends heavily on the relationship between draft and target models. Models trained with native MTP (like DeepSeek v3 and Qwen3) have built-in draft heads that achieve 60-80% acceptance, making them ideal candidates. For unrelated model pairs, acceptance can drop significantly.

The kicker? MTP is not available for MLX format models. MLX is optimized for single-token decode, and the draft model approach doesn't fit into its architecture. If you want to use MTP, you need to run models in GGUF formats. If I want to utalize the power of MTP, I have to switch to GGUF which is a lot slower on my M4 Pro. I tested this with Qwen3.6-35B-A3B in GGUF format and the speedup was not as good as I hoped. The prefill phase took longer than without MTP and the decode phase was about as fast as MLX without MTP. The trade-off isn't worth it.

I found a tool that makes MTP available for MLX models: [MTPLX](https://www.mtplx.com/). It promises to bring the benefits of MTP to MLX models, but I found it to be buggy and unreliable. It crashed my MacBook multiple times. I will keep an eye on this project, but for now, MTP is not a viable option for my local setup.

### Reaching the limits

I came to the conclusion that I have reached the limits of what I can do with my hardware with current software. Hopefully llama.ccp (and thus LM Studio) will support MTP with MLX and prompt caching in the future, but for now, I have to look for other ways to improve performance. I have reached the limits of what I can do with my current setup. 

## Hosting a server

So what is the next step? I got my hands on a Dell Max Pro GB10 (thanks Willem!), a server that I can use to host my AI models. The Dell Max Pro GB10 is equipped with more powerful hardware than my MacBook Pro:

| Spec | M4 Pro (MacBook Pro) | Dell MAX Pro GB10 |
|------|---------------------|-------------------|
| CPU | 14-core Arm | 20-core Arm|
| GPU | 20-core integrated Apple GPU | NVIDIA Blackwell GB10 (6144 CUDA Cores) |
| NPU | 16-core Neural Engine, 38 TOPS | - |
| System Memory | 48 GB unified | 128 GB unified |
| Memory Bandwidth | 273 GB/s | 273  GB/s |
| FP16/BF16 Compute | 13 TFLOPS | 30 TFLOPS |
| Form Factor | 16" laptop, portable | mini pc, less portable |
| Power | ~66W TDP | 140W TDP |

The main advantage of the Dell MAX Pro GB10 is its ability to run larger models with more parameters, which can lead to better performance and more accurate results. The server's powerful GPU allows for faster prefill, but the memory bandwidth is still a limiting factor. The Dell MAX Pro GB10 has exactly the same memory bandwidth as the M4 Pro, which means that while it can handle larger models, it may still face bottlenecks when generating tokens. However, the increased compute power and memory capacity of the Dell MAX Pro GB10 make it a great choice for hosting AI models.

## Coming up

The coming weeks I will be setting up vLLM on the Dell MAX Pro server and comparing it to my local setup. I will be running a variety of models, to see how they perform in terms of speed, accuracy, and resource usage. I will also be experimenting with different use cases for these models. Let's see how far I can push the limits of local AI and what new capabilities I can unlock with a server-based setup. Stay tuned for more insights and findings in my next post!