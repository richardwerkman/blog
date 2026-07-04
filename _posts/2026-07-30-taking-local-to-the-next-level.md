---
title: "My Journey Running AI Locally: Taking Local to the Next Level"
date: 2026-07-30 10:00:00 +0000
categories: [AI, Local AI, Machine Learning]
tags: [local-ai, llm, vllm, lm-studio, m4-pro, dell-max-pro]
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

## The limit

I feel like I have reached the limits of what I can do with my current setup. While the M4 Pro is a powerful chip, running large models like Qwen3.6-27B locally still presents challenges in terms of memory and processing power. I have noticed that while the model performs well for most tasks, there are certain complex queries that can cause slowdowns or even crashes.

What determines the speed of the model:

- **Compute**: The M4 Pro is a capable chip, but it has its limits. Running multiple processes or heavy workloads can lead to performance degradation. For LLM inference, compute determines how quickly the model can process each token. A FLOP (floating-point operation) is a single mathematical calculation, and a PFLOP represents one trillion FLOPS per second. During text generation, each token requires billions of FLOPs to compute the probability distribution over the vocabulary. More compute means faster token generation — the difference between a responsive assistant and one that takes seconds per word. The M4 Pro's Neural Engine delivers 38 TOPS (trillion operations per second) for AI workloads, while the GPU provides the bulk of FP16/BF16 compute for model inference.
- **RAM Speed**: The speed of the RAM can also impact the performance of the model. Faster RAM can lead to quicker data access and processing times. LLMs are famously memory-bandwidth-bound during inference. The model weights must be loaded from memory for every single token generated. On Apple Silicon, the unified memory architecture means the GPU shares the same memory pool as the CPU, so memory bandwidth is the shared bottleneck. When bandwidth is saturated, token generation slows down regardless of how much compute is available. This is why memory bandwidth (measured in GB/s) is often a better predictor of local LLM throughput than raw TFLOPS.

Where does the M4 Pro stand in all this?

| Spec | M4 Pro (14" MacBook Pro) |
|------|--------------------------|
| CPU | 14 cores (6 performance + 8 efficiency) |
| GPU | 20 cores |
| Neural Engine | 16-core, 38 TOPS |
| Unified Memory | Up to 48 GB |
| Memory Bandwidth | 200 GB/s |
| FP16/BF16 Compute (GPU) | ~2 TFLOPS |

The M4 Pro is an impressive chip for a laptop, but the 200 GB/s memory bandwidth and 48 GB memory ceiling are the real constraints for local AI. The 2 TFLOPS of FP16 compute is modest compared to server-grade GPUs, which is why larger models feel sluggish.

## Hosting a server

Ofloading the model to a server can help alleviate some of the limitations of running it locally. By hosting the model on a server, I can take advantage of more powerful hardware and resources, allowing for faster processing and better performance.

Some advantages of hosting a server include:
- **More powerful hardware**: The Dell Max Pro GB10 has more powerful GPUs and more memory, allowing for faster processing and better performance.
- **Sharing resources**: Hosting a server allows multiple users to access the model simultaneously, making it more efficient for collaborative work. Also you will need less powerful hardware on your local machine, since the heavy lifting is done on the server.
- **Offloading compute**: By offloading the compute to a server, I can free up resources on my local machine for other tasks and save on battery life.

Some of the downsides of hosting a server include:
- **Cost**: Hosting a server can be expensive, the hardware nowadays is getting more expensive and the electricity costs can add up over time.
- **Connection**: To access the server, I need a stable internet connection, which loses a key benefit of running the model locally. If the connection is slow or unstable, it can lead to delays and interruptions in processing.

I got my hands on a Dell Max Pro GB10, a server that I can use to host my AI models. The Dell Max Pro GB10 is equipped with more powerful hardware than my MacBook Pro:

| Spec | M4 Pro (MacBook Pro) | Dell MAX Pro GB10 |
|------|---------------------|-------------------|
| CPU | 14-core (6P + 8E) Apple Silicon |  |
| GPU | 20-core integrated Apple GPU | NVIDIA Blackwell GB10 |
| AI Accelerator | 16-core Neural Engine, 38 TOPS | 6100 NVIDIA Cuda Cores |
| System Memory | 48 GB unified | 128 GB unified |
| Memory Bandwidth | 273 GB/s | 273  GB/s |
| FP16/BF16 Compute | 38 TFLOPS | 1000 TFLOPS |
| Form Factor | 16" laptop, portable | mini pc, less portable |
| Power | ~66W TDP | up to 280W TDP |

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
- **200+ model architectures**: Supports Llama, Qwen, Mixtral, DeepSeek, and many more

### The trade-offs

vLLM is more complex to set up than LM Studio. It requires Linux (though Apple Silicon support is improving), demands more operational knowledge, and has a steeper learning curve. For my Dell MAX Pro server though, it's the right tool: I can run a 70B parameter model across two L40S GPUs with quantization, getting responsive token generation that my MacBook simply cannot match. The setup effort pays off immediately in capability.