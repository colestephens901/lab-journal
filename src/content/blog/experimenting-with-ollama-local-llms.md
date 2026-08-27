---
title: "Experimenting with Ollama and Local LLMs in My Homelab"
description: "What I learned running Ollama and Open WebUI on CPU-only homelab hardware, comparing local models against an RTX 4080 desktop, and planning a dedicated GPU-backed AI platform."
pubDate: "2026-08-26"
heroImage: "../../assets/blog/ollama-local-llm-homelab.png"
heroImageAlt: "Local AI in the homelab featuring Ollama and Open WebUI with CPU and GPU accelerated inference."
heroImageCaption: "My local AI experiments evolved from CPU-only inference on homelab hardware to testing larger models with a dedicated RTX 4080 GPU."
type: "project-writeup"
tags:
  - Ollama
  - Open WebUI
  - Local AI
  - Linux
  - Proxmox
  - GPU
  - Self-Hosting
featured: false
draft: false
---

One of the projects I had wanted to experiment with for a while was running a large language model entirely inside my own homelab.

Cloud-hosted AI services are extremely capable, but running a model locally introduces a completely different set of questions.

How much hardware does it actually need?

How usable is inference without a GPU?

How much does model size affect responsiveness?

How much VRAM is enough?

And perhaps most importantly for a homelab: does it make sense to dedicate hardware to local AI at all?

I decided to find out by deploying **Ollama** with **Open WebUI** and experimenting with several different models across two very different systems.

The project started with CPU-only inference in the homelab and eventually expanded into testing much larger models on my desktop with an NVIDIA RTX 4080.

The difference between those environments taught me far more about local AI hardware requirements than simply installing Ollama ever could.

## Why Run an LLM Locally?

My interest in local AI was less about replacing every cloud AI service and more about understanding the infrastructure required to run one myself.

Running an LLM locally provides several interesting advantages:

- Model inference happens on hardware I control
- Prompts can remain inside my own environment
- Models remain available without depending on an external API
- Different models can be tested easily
- The system can potentially integrate with other homelab services
- It provides hands-on experience with AI workloads and hardware acceleration

It also creates an interesting infrastructure problem.

Unlike many of the services I run, an LLM can consume a significant amount of CPU, memory, GPU compute, and VRAM.

That made this project as much about **hardware sizing and resource management** as it was about AI.

## Building the Local AI Stack

I chose two primary applications for the project:

- **Ollama** for downloading and running local language models
- **Open WebUI** for providing a browser-based interface to those models

At a high level, the stack is simple:

```text
Web Browser
     │
     ▼
 Open WebUI
     │
     ▼
   Ollama
     │
     ▼
Local Model
     │
     ▼
CPU / GPU
```

Ollama handles the actual model runtime while Open WebUI provides a much more convenient interface for interacting with it.

This separation also makes the platform flexible.

Open WebUI can provide the user-facing application while Ollama handles whichever models I decide to load underneath it.

## Starting Small

My first experiments focused on relatively small models that had a reasonable chance of running on CPU-based homelab hardware.

Models I experimented with during this stage included:

- `qwen3.5:4b`
- `llama3.2:3b`
- `qwen3:4b`
- `qwen3:1.7b`
- `nomic-embed-text`

The smaller language models were useful for proving that the platform worked.

`qwen3.5:4b` became one of the better-quality options in the initial environment, while `llama3.2:3b` provided a lighter alternative when responsiveness mattered more than model capability.

I also installed `nomic-embed-text` for experimenting with embeddings and potential retrieval-augmented generation workflows.

At this point, the goal was not to run the largest model possible.

It was simply to establish a functional local AI platform and understand how the workload behaved.

## Running Ollama on pve2

The first serious home for the AI stack was `pve2`.

This gave me a system inside the homelab where I could dedicate resources to Ollama and Open WebUI without relying on my main desktop.

The major limitation was obvious, however:

**there was no dedicated GPU available for inference.**

That meant Ollama relied primarily on CPU resources and system memory.

For smaller models, this was workable.

The system could load the model, generate responses, and provide a completely self-hosted AI experience.

That alone was useful because it demonstrated that a dedicated GPU is not an absolute requirement for experimenting with local LLMs.

But "it runs" and "it feels fast" are very different things.

## CPU-Only Inference

CPU-only inference was probably the most important first lesson from the project.

Smaller quantized models were usable, but response generation was noticeably slower than the AI services I was accustomed to using.

The difference becomes easier to understand when comparing the two workloads.

A typical service in my homelab might spend most of its time waiting for:

```text
Network Traffic
      │
      ▼
Application
      │
      ▼
Database / Storage
      │
      ▼
Response
```

An LLM inference workload looks very different:

```text
Prompt
  │
  ▼
Model Parameters
  │
  ▼
Heavy Computation
  │
  ▼
Token Generation
  │
  ▼
Response
```

Generating every response requires substantial computation.

The CPU could perform that computation, but it was not particularly fast at it.

This made small models acceptable for experimentation while also making it very clear why GPUs dominate local AI discussions.

## Model Size Changed Everything

One thing that became obvious very quickly was that model size matters enormously.

A small model might be perfectly usable on limited hardware.

Moving up several billion parameters can dramatically change:

- Memory requirements
- Model loading time
- Generation speed
- CPU utilization
- GPU VRAM requirements
- Overall responsiveness

That meant choosing a model was not simply a question of:

> Which model gives the best answers?

It was really:

> Which model gives the best answers that this hardware can run comfortably?

That tradeoff became much clearer when I moved the experiment away from CPU-only hardware.

## Testing Local LLMs on My Desktop

To understand what dedicated GPU acceleration would change, I tested Ollama on my desktop.

That machine is in an entirely different performance class from the CPU-only homelab environment.

It uses an **Intel Core i9-13900K** along with an **NVIDIA RTX 4080**.

The RTX 4080 gave Ollama access to dedicated GPU compute and significantly more memory bandwidth than CPU-only inference.

The difference was immediately noticeable.

Instead of waiting for responses to slowly generate token-by-token, models that fit comfortably within GPU memory felt substantially more responsive.

This was the moment where local AI stopped feeling like a technical demonstration and started feeling like something I could realistically use.

## Moving Beyond Small Models

With the RTX 4080 available, I experimented with much larger models than I had been comfortable running on the original CPU-only setup.

The models I tested included models in roughly three different classes:

```text
9B Model
   │
   ▼
Fast and responsive

14B Model
   │
   ▼
Larger, more capable,
still very usable

27B Model
   │
   ▼
Much heavier hardware requirements
```

The 9B-class models ran extremely well.

They were significantly more capable than the smallest models I had started with while still maintaining excellent responsiveness on the GPU.

The 14B-class models were especially interesting.

They provided a noticeable improvement in capability while still performing well enough that interacting with them felt natural.

For my testing, this became one of the most appealing model sizes.

It represented a strong middle ground between model quality and hardware requirements.

## Testing Gemma 3 27B

I also experimented with **Gemma 3 27B**.

This was where another major lesson appeared:

**having a powerful GPU does not mean every model will fit comfortably on it.**

The RTX 4080 is a very capable GPU, but larger models can still exceed its available VRAM depending on model size and quantization.

Once a model no longer fits completely within VRAM, part of the workload may need to spill into system memory or otherwise rely more heavily on the CPU.

The result is a significant performance change.

Conceptually, the ideal path looks like:

```text
Model
  │
  ▼
GPU VRAM
  │
  ▼
GPU Compute
  │
  ▼
Fast Inference
```

When the model becomes too large:

```text
Large Model
    │
    ├─────────────► GPU VRAM
    │
    └─────────────► System RAM
                         │
                         ▼
                     CPU / PCIe
                         │
                         ▼
                  Slower Inference
```

Gemma 3 27B could run, but it exposed the limits of the hardware much more clearly.

This taught me that **VRAM capacity can be just as important as raw GPU performance** when selecting hardware for local AI.

## The RTX 4080 Changed My Expectations

Running Ollama on the desktop changed my expectations for what a permanent local AI deployment should feel like.

The CPU-only system proved that local LLMs were possible.

The GPU-backed system showed what they could feel like when the hardware was properly matched to the workload.

The comparison was roughly:

```text
pve2
CPU-Only
   │
   ├── Small models: usable
   ├── Larger models: increasingly slow
   └── Great for experimentation


Desktop
RTX 4080
   │
   ├── Small models: extremely fast
   ├── 9B models: very responsive
   ├── 14B models: strong balance
   └── 27B models: VRAM becomes limiting
```

That experience made one thing clear:

If I wanted local AI to become a permanent service rather than an occasional experiment, I eventually wanted dedicated GPU acceleration inside the homelab.

## Why Not Just Use the Desktop?

The obvious solution would be to simply leave Ollama running on my gaming desktop.

Technically, that would work.

Operationally, I do not think it is the best long-term design.

My desktop serves a completely different purpose and is not intended to function as an always-on infrastructure server.

A permanent service should ideally live on infrastructure designed to remain available independently of whether my desktop is powered on, rebooting, gaming, or being worked on.

The separation I want is:

```text
Desktop
Gaming / Workstation
        │
        │ Not a permanent dependency
        ▼
────────────────────────────

Homelab
Always-On Infrastructure
        │
        ▼
Local AI Service
```

That pushed the project toward a longer-term hardware plan.

## Future Plan: GPU Acceleration on pvenas

My eventual goal is to add a dedicated GPU to `pvenas` and use that system to provide GPU-backed AI capacity inside the homelab.

`pvenas` already serves as one of the larger compute platforms in the environment, making it a more natural location for future GPU workloads than my desktop.

The goal would be to create an architecture closer to:

```text
Client Devices
      │
      ▼
  Open WebUI
      │
      ▼
    Ollama
      │
      ▼
Virtual Machine / Container
      │
      ▼
GPU Passthrough
      │
      ▼
Dedicated GPU
      │
      ▼
Local LLM
```

There are still decisions to make around the exact GPU.

The desktop testing taught me that the choice should not be based purely on gaming performance or raw compute.

For an LLM server, I care heavily about:

- VRAM capacity
- Model compatibility
- Power consumption
- Cooling
- Physical size
- Passthrough compatibility
- Performance per dollar

A GPU with more VRAM can sometimes be more useful for this workload than a faster GPU with less memory.

That is something I understood conceptually before the testing, but seeing a larger model run into memory limitations made the tradeoff much more tangible.

## GPU Passthrough Adds Another Layer

Moving the GPU into `pvenas` will also introduce another interesting infrastructure challenge.

Rather than running Ollama directly on a Windows desktop with native GPU access, I will most likely need to expose the GPU to a virtualized workload.

That introduces another stack of dependencies:

```text
Physical GPU
     │
     ▼
Proxmox Host
     │
     ▼
IOMMU / PCIe Passthrough
     │
     ▼
Linux VM
     │
     ▼
NVIDIA Drivers
     │
     ▼
Ollama
     │
     ▼
Model
```

Any one of those layers can become a troubleshooting point.

That makes the future GPU deployment interesting beyond just increasing inference speed.

It becomes another opportunity to work with virtualization, hardware passthrough, Linux drivers, resource allocation, and service architecture.

## Open WebUI Made Local Models Much More Practical

Ollama provides the model runtime, but Open WebUI made the overall system much easier to use.

Instead of interacting with models exclusively through command-line tools or API calls, Open WebUI provides a familiar browser-based chat interface.

The final user-facing stack is essentially:

```text
Browser
   │
   ▼
Open WebUI
   │
   ▼
Ollama API
   │
   ▼
Selected Model
   │
   ▼
Available Hardware
```

That makes switching between models much easier and turns Ollama into something that feels like an actual service rather than just a command-line experiment.

It also creates opportunities for future integrations, including embeddings, retrieval-augmented generation, document search, and potentially other homelab services.

## What Went Well

Several parts of the project worked better than I initially expected.

First, getting a functional local AI environment running did not require specialized hardware.

Small quantized models could run on CPU-based infrastructure, which made it possible to begin experimenting before purchasing any dedicated AI hardware.

Second, Ollama made model management surprisingly straightforward.

Being able to download a model and begin testing it without manually building an inference environment lowered the barrier to experimentation significantly.

Third, Open WebUI provided a clean interface that made comparing models much easier.

Finally, testing on the RTX 4080 showed that local LLM performance can become extremely practical when the workload fits comfortably on the GPU.

## What Was Difficult

The largest challenge was understanding realistic hardware expectations.

Model names do not immediately communicate how a model will behave on a particular system.

Two questions repeatedly became important:

```text
Will the model run?
```

and:

```text
Will the model run well?
```

Those are not the same question.

CPU-only inference demonstrated that a model could technically operate while still being slower than I would want for everyday use.

The 27B testing demonstrated that a powerful GPU could technically run a large model while still encountering limitations when the model exceeded comfortable VRAM capacity.

That distinction changed the way I think about hardware sizing for AI workloads.

## What I Learned

The biggest lesson from this project is that local LLM performance is a balance between **model capability and available hardware**.

There is no universally best model.

The right model depends heavily on the system running it.

My main takeaways were:

- Small quantized models can make CPU-only local AI practical
- CPU inference is useful for experimentation but can become slow quickly
- GPU acceleration dramatically changes the usability of local LLMs
- VRAM capacity is extremely important when choosing models
- Larger models are not automatically better if the hardware cannot run them efficiently
- A 9B-14B model can provide a very attractive balance between capability and responsiveness
- Dedicated server hardware makes more sense than depending permanently on a gaming desktop
- Hardware sizing should consider power, cooling, VRAM, and virtualization compatibility in addition to raw performance

Most importantly, I learned more by running the same type of workload on different hardware than I would have by simply reading model requirements.

## Where the Project Goes Next

The current environment has already accomplished its original goal: giving me hands-on experience running and managing local LLMs.

The next stage is turning the experiment into a more permanent piece of homelab infrastructure.

My longer-term plan is to add dedicated GPU capacity to `pvenas`, provide that GPU to a Linux workload, and move the primary Ollama environment onto hardware designed to remain available with the rest of the lab.

That should give me the best parts of both environments I tested:

```text
Homelab Availability
        +
Dedicated GPU Performance
        +
Larger Local Models
        +
Open WebUI
        +
Infrastructure I Control
```

Once that hardware is installed, the next part of this project will be working through GPU passthrough, driver configuration, model selection, resource monitoring, and determining which models provide the best balance for the available VRAM.

At that point, I will have gone from asking whether a local LLM could run in my homelab to designing infrastructure specifically around running one well.

That progression is exactly why I enjoy projects like this.