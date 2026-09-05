---
title: "My Dell Precision Max Pro GB10 Experiment: Chasing Tokens on Linux"
date: 2026-08-22 10:00:00 +0000
categories: [AI, Local AI, Machine Learning]
tags: [local-ai, llm, dell-max-pro, vllm, llama.cpp, lm-studio, lmster, linux, qwen3.8]
layout: post
comments: true
---

Last time I wrote about pushing my M4 Pro to its limits and teasing what might come next. That next thing arrived: a **Dell Precision Max Pro GB10** server, courtesy of Willem. Finally, a machine with enough GPU VRAM to run the models I've been watching from afar.

The goal was simple: run more powerful models and see what kind of tokens-per-second I could squeeze out. What followed was a month of trial, frustration, and eventually, some impressive numbers. Here's the full story.

## The Hardware

The Dell Precision Max Pro GB10 is a workstation-grade server built for AI workloads. It comes with a blackwell-grade GPU and enough VRAM to run 120B class models. However, it is also very similar to my macbook in the sense that it works with Unified Memory with equal bandwith of 273GB p/s. In theory this means I can run models at exactly the same t/s as on my MacBook. But I knew that software support on nvidia hardware is a lot better. But which software stack should I use to squeeze the most performance out of this box?

## Attempt 1: LM Studio Server (LMSTer)

My first instinct was to try **LM Studio Server** (lmster), since I already knew the LM Studio ecosystem from my Mac experiments. I figured the workflow would be familiar, maybe a bit different on Linux, but manageable.

It wasn't. The server version of LM Studio feels a lot less polished.

Downloading models through lmster was a hassle. There is a model browser, but it only offers a handful of models out of the box. Also it doesn't let you pick a quantization variant, just the base model. That way I ended up downloading models in FP16, without an MTP head, so I missed performance. That's why I ended up downloading models manually from HuggingFace anyway. Once a model was loaded, the server worked, but configuration options were limited. Since the Dell box runs on Linux I could enable MTP for the first time (as I explained in my last blog it isn't implemented for MLX in llama.cpp on MacOS). But other improvements were missing... For example I learned about `flash-attention`, giving more performance on large contexts. But the option is missing in LMster. I couldn't tune the model the way I wanted, and that lack of configurability was a dealbreaker for chasing performance.

The result: **20 tokens per second**.

lmster felt like a stepping stone, not a destination. It got a model running, but it wasn't a great experience, and it was clearly less configurable than what llama.cpp offers. Time to move on.

## Attempt 2: Llama.cpp

Next I tried **llama.cpp**, the actual core of LM Studio. It's highly configurable, supports a wide range of quantization formats, and I knew it could push hardware close to its limits on my Mac.

On the GB10, I was hopeful. I loaded Qwen3.8-27B with llama.cpp, enabled features like `-fa` (flash attention), and crossed my fingers.

The result: **20 tokens per second**.

That was... fine. Better than my M4 Pro, sure, but this is a machine that costs many times as much and has vastly more VRAM. 20 tps felt like leaving performance on the table. llama.cpp itself was stable and configurable, but the throughput numbers told me there was more to extract with a different approach.

I also noticed vision wasn't working with the Qwen model like it did in LMster (even with the exact same model image). I turns out a specific vision image needs to be loaded.

## Attempt 3: vLLM

This is where things got complicated — and eventually, interesting.

**vLLM** is designed for high-throughput inference, and on paper, it should be the right tool for squeezing maximum tokens per second out of enterprise hardware. In practice, getting it working on the GB10 was its own mini-project.

### The Setup Struggles

vLLM doesn't just work out of the box. You need to understand which settings matter:

- **Tool calling**: Each model handles tool calling different. vLLM has a setting for a tool calling template. If you forget to set this setting the model will just fail to call tools at runtime.
- **Load times**: Where llama.cpp just takes 30 seconds to load, vLLM takes minutes. It does a lot of optimizing and takes all the RAM it can. This makes "just in time loading" very impractical. Also testing different settings takes a long time, if changing a setting means 5 minutes of loading before the model can be tested.

Every decision had consequences. Get the memory settings wrong and vLLM would consume all available RAM, crash Firefox, and leave you staring at a frozen desktop. I learned this the hard way.

All of these settings weren't clearly documented for the Qwen model. Maybe they are for other models. But it was a lot of trial and error to get this working.

### The Payoff: NVFP4

Once I had vLLM configured correctly, I discovered **NVFP4** quantization. It is Nvidia's native 4-bit format optimized for their GPUs. This isn't a general-purpose quantization like AWQ or GPTQ; it's specifically tuned for Nvidia blackwell hardware, which means the kernels are optimized for the exact architecture in the GB10.

With NVFP4, Qwen3.8-27B on vLLM hit **30 tokens per second**.

That's 50% faster than llama.cpp on the same hardware, and it came after what felt like an unreasonable amount of setup effort.

![vLLM running Qwen3.8-27B on Dell Max Pro GB10](/assets/img/posts/dell-max-pro-gb10/vllm-gb10.png)

*30 tps with Qwen3.8-27B on vLLM with NVFP4.*

## Lessons Learned

After a month of cycling through tools, I've distilled a few hard-won lessons that I wish I'd known before starting:

### 1. Larger model != better inteligence

I tried multiple models on the 128GB of RAM available on this box. I could finally run medium size models (120B parameters) instead of the small models fitting on my Mac. A few models I tried:

- gpt-oss-120b (4-bit)
- laguna-s-120b (4-bit)
- nemotron-3-super-120b (4-bit)
- deepseek-v4-flash (2-bit)

However, I was not quite impressed with these models. It was to be expected of the older GPT OSS model. It is a year old which is an eternatiy in LLM world! But the newer laguna and nemotron models also weren't as good as I expected. I was mainly missing vision capability that I got used to from the Qwen models. Otherwise they seemed to perform on-par with Qwen3.6-27B, with similar t/s.

Here's how they stack up on the benchmarks that matter for coding work. I swapped out MMLU-Pro for the Artificial Analysis Intelligence Index (a composite of nine evaluations spanning coding, science, reasoning, and professional tasks) since it's independently tracked across vendors rather than a single vendor-reported academic score, and added a vision column since that's what I actually missed day-to-day:

| Model | Params (Active) | SWE-bench Pro | SWE-bench Verified | Terminal-Bench 2.1 | AA Intelligence Index | Vision |
|-------|----------------|-----------|-----------|----------------------|----------------------|--------|
| **Qwen3.8-27B** | 27B full | 61.7% | -| 73.0% | 52 | Yes |
| **GPT-OSS-120B** | 117B (5.1B) | 16.2% | 62.4% | — | 24 | No |
| **Poolside Laguna-S** | 118B (8B) | 59.4% | - | 70.2% | — | No |
| **Nemotron-3-Super-120B** | 120B (12B) | - | 60.47% | — | 26 | No |
| **DeepSeek-V4-Flash** | 284B (13B) | 52.3% | 79.0% | 82.7 | 52 | Yes |

\* All figures are vendor-published or Artificial Analysis's own runs, not independently reproduced by me.

The pattern is clear: none of these 120B-class models beat Qwen3.8-27B on the Intelligence Index, despite having 4-9x the parameters. 

I also tried DeepSeek-V4-Flash-Vision. It is an impressive model, even outperforming Qwen3.8-27B, but it is a huge model so I could only fit it on this machine in 2-bit quantization. That made it a bit crippled, however it still outperformed Qwen3.8-27B. And even at 2-bit quantization it took almost all my available RAM (96gb), leaving little room for context. I think one would need at least 2 of these machines to run the model in 4-bit and with enough context length. That made it impractical for my use case.

After Qwen3.8-27B was released it made no sense to use the larger models anymore, since Qwen3.8-27B outperforms them on all benchmarks. The lesson here is that having more RAM, doesn't equal running better models. At least with the current set of models available in this range. Maybe in the future better models in the 120B range will be released. But they seem to be released less frequently than the smaller models. It is logical since the focus lies mainly on letting as many people run their models, and having 128GB of RAM is quite a luxury in 2026.

If you're thinking of buying a machine with 128GB of Unified RAM, think about what kind of models you want to run on there. It might be better to get a machine with less, but faster RAM so you can run the Qwen3.8 model with more performance.

### 2. Use the Correct Quantization for Your Hardware

This was the biggest revelation. **NVFP4** (Nvidia FP4) is the quantization format that matters if you're running on Nvidia hardware. It's not just a smaller number, it's a quantization format with kernels specifically optimized for Nvidia's GPU architecture.

For AMD GPUs, the equivalent is **ROCM FP** (ROCMPF). Using the wrong quantization format means you're running suboptimal kernels, which directly translates to lower tokens per second.

General-purpose quantizations like AWQ, GPTQ, or even standard 4-bit GGUF will work, but they won't extract the maximum performance your hardware can deliver. Know your hardware, then use the quantization format designed for it.

### 3. vLLM Is Powerful But Demanding

vLLM gave me the best performance by a wide margin: 30 tps vs 20 tps from llama.cpp. But the setup complexity was genuinely frustrating. You need to understand memory management, context window trade-offs, and the specific configuration requirements of your model. It's not a "download and run" tool. If you're willing to put in the effort, the performance payoff is substantial. If you want something that just works, llama.cpp (or even lmster for basic use) is the better choice.

## Comparison Summary

| Tool | Setup Effort | Stability | Qwen3.8-27B TPS | Verdict |
|------|-------------|-----------|-----------------|---------|
| **LM Studio Server** | Low | Medium | ~20 | Easy to start, limited control |
| **Llama.cpp** | Medium | High | ~20 | Stable, configurable, but left performance on the table |
| **vLLM** | High | Medium (config-dependent) | ~30 | Best performance, but demands deep understanding of settings |

## Conclusion

The Dell Pro Max GB10 proved it has the raw horsepower to run large models at impressive speeds. The interesting part is that the hardware is pretty similar to my MacBook in terms of Unified Memory bandwidth, but the software ecosystem on Linux, particularly with vLLM and NVFP4, allowed me to extract significantly more performance. This bodes well for the future of Mac, there is a lot of potential for Apple Silicon to catch up in the future, but for now, if you want to run large models at cloud-competitive speeds, Linux (and NVIDIA) with the right software stack is the way to go.

vLLM with NVFP4 gave me the best numbers: 30 tokens per second with Qwen3.8-27B. But getting there took a month of trial and error, multiple failed configurations, and more than a few moments of questioning my life choices when Firefox crashed because vLLM ate all my RAM.

If you're considering this path, my advice is simple: find a cookbook for your specific model, use the quantization format designed for your hardware, and be prepared for a learning curve. The performance is worth it — once you've climbed it.

The hardware is capable. The software ecosystem is catching up. And for the first time, running large local models at cloud-competitive speeds feels within reach, even if the journey there tests your patience.

## Next Up

Now that I've proven the GB10 can deliver solid performance with vLLM, I want to explore whether I can push even further — larger models, longer context windows, and maybe even multi-model setups. The hardware is there. The question is what else I can squeeze out of it.
