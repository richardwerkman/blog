---
title: "My Journey Running AI Locally: Is It Worth It?"
date: 2026-06-13 10:00:00 +0000
categories: [AI, Local AI, Machine Learning]
tags: [local-ai, llm, mlx, ollama, lm-studio, m4-pro]
---

# Running AI on Your Own Hardware: A Wild Ride

So you're thinking about running AI locally? Welcome to the club! After spending countless hours tinkering with models, dealing with mysterious errors, and watching my laptop fan work overtime, I've learned a thing or two. Let me share my journey—the good, the bad, and the surprisingly fast.

## Why Go Local Anyway?

Before we dive into the technical deep end, let's talk about **why** anyone would choose to run AI locally when ChatGPT is just a browser tab away.

- Token Costs Are Rising

Remember when API calls were cheap? Yeah, those days are getting blurrier in the rearview mirror. If you're using AI heavily for coding, writing, or experimentation, those costs add up faster than you can say "context window."

### 🔒 Privacy Matters

Not everything you work on belongs in someone else's data center. Whether it's proprietary code, sensitive documents, or just the principle of the thing—keeping your data on your machine is worth something.

### 🌐 Always Available

No internet? No problem. Rate limits? Not today, Satan. When you're running locally, the only limit is your hardware (and maybe your electricity bill).

### 📅 The 9-Month Gap

Here's the kicker: open-weight models are typically about 9 months behind their closed-source cousins. But you know what? Nine months ago, frontier models were still pretty darn good. And if this trend continues, in 9 months we might be running Opus-class models on consumer hardware. Now *that's* exciting!

### 💻 Hardware Is Getting More Accessible

The barrier to entry is shrinking. You don't need a server farm anymore—modern laptops with unified memory are surprisingly capable.

## Terminology 101: Let's Decode the Jargon

Before we get into the juicy details, let's clarify some terms that get thrown around:

### Full Weight vs. Active Parameters

Think of a model's **full weight** as its total capacity—like a car's engine size. **Active parameters**, on the other hand, are what's actually being used during inference. Some models use mixture-of-experts (MoE) architecture, where only a subset of parameters activate for each request. It's like having an 8-cylinder engine but only firing 2-4 cylinders at a time for efficiency.

### Quantization: 4-bit vs. 8-bit

**Quantization** is basically compression for AI models. Instead of storing each parameter with high precision (think: storing "3.14159265359" vs just "3.14"), you reduce the precision to save memory.

- **8-bit quantization**: Better quality, needs more RAM
- **4-bit quantization**: More compressed, faster, but slightly lower quality

The quality difference is often negligible for everyday tasks, making 4-bit a sweet spot for consumer hardware.

### MLX vs. GGUF

These are different formats/frameworks for running models:

- **MLX**: Apple's framework optimized for Apple Silicon, taking full advantage of unified memory
- **GGUF**: Cross-platform format that works everywhere (think: the MP3 of AI models)

## My Setup: The Hardware

I'm running all this on an **M4 Pro with 48GB of unified RAM**. Is it overkill for casual browsing? Absolutely. Is it perfect for AI experimentation? *Chef's kiss.*

The unified memory architecture is a game-changer—the GPU and CPU share the same memory pool, which means you can load larger models than you'd expect.

## The Tools of the Trade

I've tested four main runners for local inference. Here's how they stack up:

| Tool | Pros | Cons | Best For |
|------|------|------|----------|
| **LM Studio** | Beautiful UX, HuggingFace model browser, JIT loading, vscode integration | Some MLX compatibility issues | General use, beginners |
| **Ollama** | Simple CLI, great for automation, solid stability | Limited model format support | Scripts, automation |
| **Llama.cpp** | Maximum control, bleeding edge features | Steeper learning curve | Power users |
| **MLX Server** | Native Apple Silicon optimization | Apple-only, smaller ecosystem | Maximum performance on Mac |

### My Favorite: LM Studio

I keep coming back to **LM Studio** because it just *works*. The UI is clean, downloading models from HuggingFace is a breeze, and the just-in-time model loading means the model only spins up when I actually need it from VS Code. 

![LM Studio Interface](/assets/img/posts/lm-studio-screenshot.png)
*LM Studio's clean interface makes model management actually enjoyable*

The main caveat? MLX support is still a bit wonky. Hopefully, future updates will iron this out.

## Battle of the Models

I put several models through their paces for real coding work. Here's the roster:

| Model | Parameters | Download Size | Speed (tok/s) | Verdict | HuggingFace Link |
|-------|------------|---------------|---------------|---------|------------------|
| **GPT-OSS-20B** | 20B (less active) | ~12GB (4-bit) | ~15 | ⭐⭐⭐⭐ Surprisingly capable | [Link](https://huggingface.co/collections/gpt-oss) |
| **Qwen3.6-35B-A3B** | 32B (3B active) | ~18GB (4-bit) | ~18 | ⭐⭐⭐ Fast but loops | [Link](https://huggingface.co/Qwen) |
| **Qwen3.6-27B** | 27B full | ~16GB (4-bit) | ~6 | ⭐⭐ Too slow | [Link](https://huggingface.co/Qwen) |
| **Gemma-4-27B-A4B** | 27B (4B active) | ~16GB (4-bit) | ~20 | ⭐⭐⭐ Fast, tool issues | [Link](https://huggingface.co/google/gemma-2-27b) |
| **Gemma-4-31B** | 31B full | ~18GB (4-bit) | ~5 | ⭐⭐ Too slow | [Link](https://huggingface.co/google/gemma-4-31b) |

### The Surprise Winner: GPT-OSS-20B

For an older model of this size, GPT-OSS-20B punched way above its weight class. I used it to generate this entire blog website, and it only took 15 minutes! It's proof that you don't always need the latest and greatest.

### The Fast but Flawed: Qwen3.6-35B-A3B

Qwen3.6-35B-A3B with its MoE architecture was blazingly fast and had excellent tool-calling capabilities. The problem? It kept getting stuck in thinking loops, when the AI decides to ponder the meaning of existence instead of just writing the code. I tried various temperature and repetition penalty settings, which helped a bit, but the loops persisted.

### The Tortoise: Full Weight Models

Both Qwen3.6-27B and Gemma-4-31B were painfully slow on my hardware, even with MLX optimization. At 5-6 tokens per second, you could make a coffee while waiting for a response. It takes minutes to even process the system prompt, so you will not see a response until much later. I even saw timeouts in Copilot. Not ideal for interactive coding. Even a smaller model like Qwen3.5-9B was fairly slow.

## The Agent Experience

I also experimented with different AI coding agents:

- **Kilo Code**: Interesting but limited
- **OpenCode**: Decent open-source option
- **GitHub Copilot** (via OAICopilot plugin): My daily driver
- **Claude Code**: Excellent but requires API access

**My winner**: GitHub Copilot with local models via the OAICopilot plugin. Why? I'm already familiar with Copilot's interface, it integrates seamlessly with VS Code, and I can switch between local and paid models effortlessly. Best of both worlds!

However, I did notice that some models, like Gemma-4 and GPT-OSS, struggled with tool-calling in Copilot. They often failed to execute code or verify changes unless I was very explicit in my prompts.

![image: Copilot with Local Model](/assets/images/Screen%20Recording%202026-06-13%20at%2016.39.48.mov)

The retry button will often show its face when the model fails to call tools properly. It's a reminder that local models aren't perfect yet.

## A Thought Experiment: Models vs. Agents

Here's something I've been pondering: How much of the progress we've seen in recent years is from better **model capabilities** versus better **agent capabilities**?

Think about it—modern agents have:
- Sophisticated tool-calling mechanisms
- Better instruction sets and prompting strategies
- Improved context management
- Smarter error recovery

If agents continue to improve, we might not need ever-more-powerful models. A smaller, well-orchestrated model with great tools might outperform a larger model with poor tooling. This has huge implications for local AI—you don't need the absolute best model if your agent framework is solid.

## Hard-Won Lessons

After weeks of tinkering, here's what I learned:

### 1. Don't Chase the Bleeding Edge

Use models that have been out for a few months. The tools and frameworks need time to catch up. That shiny new model released yesterday? It might not work properly with your runner of choice.

### 2. Thinking Loops Are Real

Smaller models can get stuck in infinite thinking loops. Combat this with:
- **Repeat penalty**: Penalizes repetitive output
- **Higher temperature**: Boosts creativity and breaks patterns

### 3. Smaller Models Are Lazy

I'm used to agents that automatically verify their code changes by running builds and tests. Smaller models like Gemma-4 and GPT-OSS often skip verification unless you're *very* explicit. You can't assume they'll do the smart thing—you need to spell it out.

### 4. Explicit Instructions Win

With frontier models, you can be vague and they'll figure it out. With smaller local models, precision matters. "Fix the bug" might not work, but "Fix the null pointer exception on line 42 by adding a null check" probably will.

## The Bottom Line

Should you run AI locally on your macbook? Not quite yet.

**Don't expect miracles.** After considerable tinkering, I got a simple workflow running. But it took trial and error, patience, and accepting that local models aren't frontier models. You can expect to run into some frustrating moments. Models that are supposed to support tool-calling might not do it reliably. Some models will get stuck in loops. It's part of the journey.

**The future is bright.** If open-weight models continue tracking 9 months behind frontier models, we'll soon have Sonnet or Opus-class models running locally. Of course, you'll need serious hardware for that—just like current DeepSeek v4 models need 500+ GB of RAM. But the trajectory is clear. Thanks to quantization and better optimization, the hardware requirements are becoming more manageable.

**My sweet spot**: For simple-to-moderate coding tasks, a 20-30B parameter model running on 48GB of RAM sometimes works. MLX optimization is a must for larger models. For more complex tasks, you're better off with a smaller, well-optimized model that can call tools effectively.

**The really exciting prospect**: In 9 months, if we can run a Sonnet-class model locally on 48GB of RAM? That would be a game-changer.

## What's Next?

I'll keep watching for new models and better tooling. The local AI space is evolving rapidly, and every month brings new possibilities.

If you're considering going local:
1. Start with LM Studio for ease of use
2. Try GPT-OSS-20B or similar ~20B parameter models
3. Be patient with setup and tuning
4. Keep your expectations realistic
5. Enjoy the privacy and cost savings!

Have you tried running AI locally? Hit me up on [GitHub](https://github.com/richardwerkman) and share your experiences. I'd love to hear what's working (or not working) for you!

---

*Posted from my M4 Pro, where the fans are finally quiet again* 🎉
