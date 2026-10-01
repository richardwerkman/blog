---
title: "My Journey Running AI Locally: Deepdive into oMLX"
date: 2026-09-11 10:00:00 +0000
categories: [AI, Local AI, Machine Learning]
tags: [local-ai, llm, omlx, qwen, mtp, prefix-caching, stability]
layout: post
comments: true
---

After my experience with vLLM on NVIDIA hardware, I was painfully aware how behind my setup on Apple Silicon was in terms of raw performance. My 11 tokens per second versus the 30 per second I saw on the NVIDIA rig was a stark reminder of the software gap I was dealing with. The memory bandwith is exactly the same, so I should be getting more out of my hardware than I was.

A few months ago, I experimented with **oMLX** for running large language models locally. The project showed real promise. It had a sleek interface, great performance potential, and an ambitious roadmap. But it wasn't ready for prime time. During my testing, it crashed my Mac several times, leaving me frustrated and reaching back to the more stable (but less performant) LM Studio.

I decided to discard it and move on. But recently, I gave it another shot, and I'm genuinely impressed by how far the project has come. What was once a promising but unstable experiment has matured into something I actually use every day now.

## From Crash City to Stability

The transformation has been dramatic. The oMLX team clearly took stability seriously, implementing **robust out-of-memory (OOM) safeguards** that prevent the kind of system-crushing crashes I experienced before. Beyond that, there have been substantial **performance and stability improvements** across the board and the overall user experience has become much smoother and more reliable.

### The Secret Sauce: Paged SSD KV Cache

oMLX doesn't implement traditional PagedAttention like vLLM does, but they've built something even better for Apple Silicon—a **Paged SSD KV Cache** system. Here's how it works: oMLX maintains a two-tier memory hierarchy where hot context lives in fast unified memory while older, less-used KV cache blocks are intelligently spilled to SSD. When you reference a previous conversation prefix, oMLX restores the cached block from disk instead of recomputing it.

The real-world impact is staggering. Time to first token in long-context scenarios drops from 30-90 seconds down to 1-3 seconds. On my M4 Pro, this is the difference between waiting out a bathroom break and getting an instant response. There's also **TurboQuant** for optional 2-8 bit KV cache compression, which you can toggle in the Admin UI. This is another lever for squeezing out more performance and managing memory usage effectively.

It's a brilliant adaptation of PagedAttention's block-based philosophy, but optimized specifically for Mac hardware constraints rather than fighting them.

Where I used to reach for the force quit button within minutes, I now run models for hours without a hiccup.

## Performance That Actually Reaches the Finish Line

The real win is in the numbers. I'm running **Qwen 3.8-27B** with **Multi-Token Prediction (MTP)** and **prefix-caching** enabled, and I'm consistently hitting **20 tokens per second**. But here's the kicker: on long conversations with massive context windows, the prefill no longer takes minutes. The prefix-caching ensures that repeated context is handled intelligently, meaning I'm not waiting around for the model to process the same tokens over and over.

This is the kind of practical, day-to-day improvement that makes local AI actually viable.

## Beyond Raw Performance: The Whole Package

What pushes oMLX from "good enough" to "actually great" is attention to the user experience. A few standout features:

![oMLX Dashboard](/assets/img/posts/omlx-saves-the-day/dashboard.png)

**Dashboard**: The included dashboard gives real-time visibility into what's happening under the hood. You can see token throughput, memory usage, temperature, ssd cache size, and more. There's also a straightforward interface for managing which models you have downloaded, adjusting inference settings, and monitoring system health. It's the kind of visibility that lets you actually optimize your setup rather than guessing.

![oMLX Status Bar Menu](/assets/img/posts/omlx-saves-the-day/menu-bar.png)

**Status Bar Menu**: oMLX lives in your Mac's menu bar with a quick-access interface for loading, unloading, and switching models. It's small but powerful. All the essential controls are just a click away, making it easy to manage your local AI setup without interrupting your workflow.

**JIT Model Loading**: No more waiting for the entire model to load before you can start working. Models load on-demand, which keeps the Mac responsive and doesn't needlessly consume memory when you're switching between different workloads.

**Quantizing**: oMLX supports model quantization, so if you can't find the exact quantization for a model, you can create it yourself. This allows you to optimize models for your specific hardware, balancing performance and memory usage according to your needs. It also allows you to keep the MTP head to benefit from faster token prediction even with quantized models. Most quantizations I find online miss the MTP head so creating your own ensures you don't lose this advantage.

![benchmarking](/assets/img/posts/omlx-saves-the-day/benchmarking.png)

**Benchmarking**: oMLX includes built-in benchmarking tools that let you measure the performance and intelligence of different models and configurations on your hardware. This helps you make informed decisions about which models to use and how to optimize them for your specific setup. And it lets you share your benchmarks with the community to help others make better choices as well. Each benchmark includes the specific settings used, so you can replicate results accurately. This helps squeezing the most out of your hardware and ensures you're getting the best performance possible. And it also helps if you're interested in buying new hardware, as you can see how potential upgrades would impact performance before making a purchase. [Try it for yourself](https://omlx.ai/benchmarks/performance)!

**Edge Model Support**: oMLX is optimized for running models at the edge, meaning you can leverage the newest models immediately on your local machine. For example, support for the Qwen4 architecture was released within days of its official announcement, allowing early adopters to experiment with cutting-edge models without having to use cloud-based solutions.

## The Turning Point

What made me switch back to oMLX wasn't a single feature—it was the combination of stability, performance, and thoughtful design coming together. I went from "this is a cool prototype" to "this is the tool I reach for first" without any of the apps I had been using before feeling slow or clunky by comparison.

If you tried oMLX early and gave up like I did, I'd say it's worth revisiting. The project has matured in all the ways that matter. I no longer recommend LM Studio as the preferred local AI engine. oMLX has proven to be more stable, performant, and thoughtfully designed for the Mac ecosystem, making it my go-to choice for running large language models locally.
