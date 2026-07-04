---
title: "My Journey Running AI Locally: Taking Local to the Next Level"
date: 2026-07-30 10:00:00 +0000
categories: [AI, Local AI, Machine Learning]
tags: [local-ai, llm, vllm, lm-studio, m4-pro, dell-max-pro, mtp, speculative-decoding]
layout: post
comments: true
---

After weeks of experimenting with running AI models locally on my MacBook Pro I can share some more of my experiences and insights. I have enjoyed my setup a lot. I could for example keep running my model in the train without a stable internet connection! That felt like freedom. But I was curious, how far can I take this? How far can I push my local setup? Can I run even larger models locally? Can I run models that are more capable than Qwen3.6-35B-A3B? Let's find out!

## The Setup

Like described in my previous post my setup consists of:
1. **LM Studio** for model management and JIT loading of models
2. **Qwen3.6-35B-A3B** for fast, capable responses, vision capabilities, and excellent tool-calling
3. **Qwen3.6-27B** for more complex tasks that require more reasoning and understanding of code (but a lot slower than Qwen3.6-35B-A3B)
4. **MLX optimization** for maximum performance on my M4 Pro
5. **Open Code** as the agent harness, for smooth integration and execution of tasks

What have I used this setup for? I have used it for a variety of tasks, including:
- Writing this blog site
- Adding simple features to Stryker.NET
- Refactoring small, self-contained modules
- Generating unit tests for straightforward functions
- Answering technical questions and explaining code

While it worked great on those simple tasks, for more complex tasks I have found that Qwen3.6-35B-A3B is not capable. Some examples of tasks that are too complex for this model include:
- Resolving merge conflicts in large codebases with many interdependent files
- Implementing complex features that require deep understanding of a codebase
- Debugging subtle, hard-to-reproduce bugs across multiple layers
- Architectural refactoring that touches many interconnected components

Some other downsides of this setup include:
- **Memory Usage**: The model can consume a significant amount of RAM, which can lead to slowdowns or crashes if the system runs out of memory.
- **Battery Life**: Running large models locally can be power-intensive, leading to shorter battery life on laptops.

## How LLMs Work

Before diving into hardware limits, it helps to understand what actually happens when you send a prompt to an LLM. Inference happens in two distinct phases, each with very different hardware requirements.

### Prefill (Prompt Processing)

When you send a prompt, the model first processes the entire input in the **prefill** phase. All tokens in your prompt are processed in parallel through the network's layers. The model computes attention scores between all token pairs and builds the KV cache — a memory structure that stores the contextual understanding of every token for use during generation.

Prefill is **compute-bound**. The heavy matrix multiplications across all layers dominate, and the GPU's raw TFLOPS determine how quickly the prompt is processed. More compute means faster prompt understanding. For short prompts this phase is nearly instantaneous, but for long contexts (tens of thousands of tokens) it can take noticeable time. The key hardware specs here are:

- **GPU Compute (TFLOPS)**: Raw floating-point throughput drives matrix multiplication speed
- **VRAM/Capacity**: The entire prompt plus the KV cache must fit in memory

### Decode (Token Generation)

After prefill, the model enters the **decode** phase — generating the response one token at a time. This is the autoregressive nature of LLMs: each new token depends on all previous tokens. For every single token, the model must:

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

Understanding these two phases explains a lot about the local AI experience. When you paste a long document and hit enter, there's a pause (prefill), then tokens start streaming (decode). If tokens feel slow, it's almost always a memory bandwidth issue, not a compute issue. And it explains why throwing more compute at the problem often yields diminishing returns — you're bottlenecked by how fast data moves, not how fast it's processed.

## The limit

I feel like I have reached the limits of what I can do with my current setup. While the M4 Pro is a powerful chip, running large models like Qwen3.6-27B locally still presents challenges in terms of memory and processing power. I have noticed that while the model performs well for most tasks, there are certain complex queries that can cause slowdowns or even crashes.

Understanding the prefill vs. decode split makes these limits concrete. With 48 GB of unified memory and 200 GB/s bandwidth, I can run models up to about 35B parameters in 4-bit quantization. Anything larger either swaps to disk (crippling performance) or crashes outright. And during decode, that 200 GB/s bandwidth caps my theoretical throughput no matter how efficient the inference engine is.

Where does the M4 Pro stand in all this?

| Spec | M4 Pro (14" MacBook Pro) |
|------|--------------------------|
| CPU | 14 cores (6 performance + 8 efficiency) |
| GPU | 20 cores |
| Neural Engine | 16-core, 38 TOPS |
| Unified Memory | Up to 48 GB |
| Memory Bandwidth | 273 GB/s |
| FP16/BF16 Compute (GPU) | ~2 TFLOPS |

The M4 Pro is an impressive chip for a laptop, but the 273 GB/s memory bandwidth and 48 GB memory ceiling are the real constraints for local AI. The 2 TFLOPS of FP16 compute is modest compared to server-grade GPUs, which is why with a larger model like Qwen3.6-27B, the prefill phase is painfully slow. Even with optimizations like MLX, it lacks the raw horsepower to process large prompts quickly. And during decode, the memory bandwidth is the bottleneck, limiting token generation speed.

## Multi Token Prediction

If decode is memory-bandwidth-bound, is there any way to get more tokens per second without upgrading hardware? Enter **Multi Token Prediction** (MTP), also known as speculative decoding or speculative sampling.

### How It Works

The core insight behind MTP is simple but powerful: instead of generating one token at a time, generate several candidate tokens in parallel, then verify them all in one pass.

Here's the process:
1. A small **draft model** (or a distilled version of the target model) quickly proposes K candidate tokens
2. The **target model** verifies all K tokens in a single forward pass
3. Accepted tokens are appended; if any token is rejected, generation continues from the last accepted token

The magic is in step 2. Because the target model processes all K candidates in one pass (using the same memory read of the full model weights), you get up to K tokens for the cost of roughly one decode step. If the draft model is accurate, you can approach K× speedup.

### Real-World Speedup

The actual speedup depends on the **acceptance rate** — how often the draft model's guesses match what the target model would have chosen. In practice:

- **High acceptance** (>80%): You get close to the theoretical K× speedup. With K=6, that's nearly 6 tokens per decode step
- **Moderate acceptance** (50-80%): Still meaningful speedup, typically 2-3×
- **Low acceptance** (<50%): The overhead of running the draft model may not be worth it

Acceptance rate depends heavily on the relationship between draft and target models. Models trained with native MTP (like DeepSeek v3 and Qwen3) have built-in draft heads that achieve 60-80% acceptance, making them ideal candidates. For unrelated model pairs, acceptance can drop significantly.

The kicker? MTP is not available for MLX format models. MLX is optimized for single-token decode, and the draft model approach doesn't fit into its architecture. If you want to use MTP, you need to run models in GGUF or GPTQ/AWQ formats. If I want to utalize the power of MTP, I have to switch to GGUF which is a lot slower on my M4 Pro. I tested this with Qwen3.6-35B-A3B in GGUF format and the speedup was not as good as I hoped. The prefill phase took longer than without MTP and the decode phase was about as fast as MLX without MTP. The trade-off isn't worth it.

### Native MTP in Modern Models

The latest generation of models, including Qwen3.6 and DeepSeek v4, have MTP built into their architecture. Instead of relying on a separate draft model, these models have auxiliary "draft heads" trained alongside the main model. This approach is more efficient because:

- The draft head is tiny compared to the full model, adding minimal memory overhead
- It's trained specifically to predict what the main model would output, maximizing acceptance rates
- No need to load a second model — everything runs in the same forward pass

## Hosting a server

Ofloading the model to a server can help alleviate some of the limitations of running it locally. By hosting the model on a server, I can take advantage of more powerful hardware and resources, allowing for faster processing and better performance.

Some advantages of hosting a server include:
- **More powerful hardware**: The Dell Max Pro GB10 has more powerful GPUs and more memory, allowing for faster processing and better performance.
- **Sharing resources**: Hosting a server allows multiple users to access the model simultaneously, making it more efficient for collaborative work. Also you will need less powerful hardware on your local machine, since the heavy lifting is done on the server.
- **Offloading**: By offloading the model to a server, I can free up resources on my local machine for other tasks and save on battery life.

Some of the downsides of hosting a server include:
- **Cost**: Hosting a server can be expensive, the hardware nowadays is getting more expensive and the electricity costs can add up over time.
- **Connection**: To access the server, I need a stable internet connection, which loses a key benefit of running the model locally. If the connection is slow or unstable, it can lead to delays and interruptions in processing.

I got my hands on a Dell Max Pro GB10 (thanks Willem!), a server that I can use to host my AI models. The Dell Max Pro GB10 is equipped with more powerful hardware than my MacBook Pro:

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

The main advantage of the Dell MAX Pro GB10 is its ability to run larger models with more parameters, which can lead to better performance and more accurate results. The server's powerful GPUs and ample memory allow for faster processing, but the memory bandwidth is still a limiting factor. The Dell MAX Pro GB10 has exactly the same memory bandwidth as the M4 Pro, which means that while it can handle larger models, it may still face bottlenecks when generating tokens. However, the increased compute power and memory capacity of the Dell MAX Pro GB10 make it a great choice for hosting AI models.

## VLLM

vLLM is a fast, open-source library for LLM inference and serving, originally developed at UC Berkeley. It has become the de facto standard for self-hosted LLM serving, powering deployments at companies and research labs worldwide.

### Why vLLM over LM Studio?

LM Studio is fantastic for local experimentation on a single machine, but vLLM is built for serious inference workloads. The key differentiator is **PagedAttention** — vLLM's innovation for managing the KV cache. Traditional LLM servers pre-allocate contiguous memory for each request's attention state, leading to significant fragmentation and wasted memory. PagedAttention treats the KV cache like virtual memory, allocating it in non-contiguous blocks. This alone can improve throughput by up to 2.4x while supporting much longer context lengths.

### Key features

- **Continuous batching**: Processes multiple requests simultaneously instead of waiting for each one to finish, dramatically improving throughput
- **Prefix caching**: Reuses computed attention states for repeated prompts (like system instructions), cutting redundant computation
- **Quantization support**: Runs models in FP8, INT8, INT4, GGUF, and GPTQ/AWQ formats, fitting larger models into available VRAM
- **Multi-GPU support**: Splits models across multiple GPUs using tensor and expert parallelism, enabling models that wouldn't fit on a single GPU
- **OpenAI-compatible API**: Drop-in replacement for OpenAI's API, making integration with tools like Open Code straightforward
- **Speculative decoding**: First-class support for MTP and speculative sampling, including native draft heads in models like Qwen3 and DeepSeek v3
- **200+ model architectures**: Supports Llama, Qwen, Mixtral, DeepSeek, and many more

### The trade-offs

vLLM is more complex to set up than LM Studio. It requires Linux (though Apple Silicon support is improving), demands more operational knowledge, and has a steeper learning curve. For my Dell MAX Pro server though, it's the right tool: I can run a 70B parameter model across two L40S GPUs with quantization, getting responsive token generation that my MacBook simply cannot match. The setup effort pays off immediately in capability.

## Coming up

The coming weeks I will be testing vLLM on my Dell MAX Pro server and comparing it to my local setup. I will be running a variety of models, including Qwen3.6-35B-A3B, Qwen3.6-27B, and DeepSeek v4, to see how they perform in terms of speed, accuracy, and resource usage. I will also be experimenting with different use cases for these models. Let's see how far I can push the limits of local AI and what new capabilities I can unlock with a server-based setup. Stay tuned for more insights and findings in my next post!