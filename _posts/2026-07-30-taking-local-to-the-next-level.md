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
3. **MLX optimization** for maximum performance on my M4 Pro
4. **Open Code** as the agent harness, for smooth integration and execution of tasks

What have I used this setup for? I have used it for a variety of tasks, including:
- Writing this blog site
- Adding simple features to Stryker.NET
- ..

While it worked great on those simple tasks, for more complex tasks I have found that Qwen3.6-35B-A3B is not capable. Some examples of tasks that are too complex for this model include:
- Resolving merge conflicts in large codebases
- Implementing complex features that require deep understanding of a codebase
- ..

Some other downsides of this setup include:
- **Memory Usage**: The model can consume a significant amount of RAM, which can lead to slowdowns or crashes if the system runs out of memory.
- **Battery Life**: Running large models locally can be power-intensive, leading to shorter battery life on laptops.

## The limit

I feel like I have reached the limits of what I can do with my current setup. While the M4 Pro is a powerful chip, running large models like Qwen3.6-27B locally still presents challenges in terms of memory and processing power. I have noticed that while the model performs well for most tasks, there are certain complex queries that can cause slowdowns or even crashes.

What determines the speed of the model:

- **Compute**: The M4 Pro is a capable chip, but it has its limits. Running multiple processes or heavy workloads can lead to performance degradation. <add why compute matters for local AI models and what a PFLOP is>
- **RAM Speed**: The speed of the RAM can also impact the performance of the model. Faster RAM can lead to quicker data access and processing times. <add why ram speed matters for local AI models>

Where does the M4 Pro stand in all this?

<add M4 Pro specs for compute and ram speed>

## Hosting a server

Ofloading the model to a server can help alleviate some of the limitations of running it locally. By hosting the model on a server, I can take advantage of more powerful hardware and resources, allowing for faster processing and better performance.

Enter the Dell Max Pro, a server that I can use to host my AI models. The Dell Max Pro is equipped with more powerful hardware than my MacBook Pro:

<compare the specs of the Dell Max Pro gb10 with the M4 Pro, highlighting the differences in compute power, RAM speed, and other relevant specifications>

## VLLM

<add a section about VLLM and how it can help with running larger models locally or on a server, including its benefits and limitations>