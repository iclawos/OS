# ICLAW OS Technical Whitepaper

## A Native Operating System for the Age of Artificial Intelligence

**Version**: 1.0  
**Release Date**: April 2026  
**Status**: Concept Design & Architecture Draft

 [iclawmini.com](https://iclawmini.com/)


## Abstract

For the past six decades, the computer world has been built on the **determinism of bytes**. The rise of LLMs brings a paradigm shift: the fundamental unit changes from bytes to **tokens**, computation shifts from deterministic to probabilistic, and architecture transforms from "CPU + kernel" to "LLM + Agent". ICLAW OS is an **AI-native operating system** that fully embraces this new paradigm from kernel to user interface. This whitepaper outlines its design philosophy, core architecture, information flow model, and engineering roadmap.


## Table of Contents

1. [Paradigm Shift: From Bytes to Tokens](#1-paradigm-shift-from-bytes-to-tokens)

2. [Design Philosophy & Architecture Overview](#2-design-philosophy--architecture-overview)

3. [Kernel Design: AI-Driven Single-Threaded Master Loop](#3-kernel-design-ai-driven-single-threaded-master-loop)

4. [The Two Engines: Context/Memory Engine & Tool/Skill Engine](#4-the-two-engines-contextmemory-engine--toolskill-engine)

5. [Processes & Scheduling: Agents as First-Class Citizens](#5-processes--scheduling-agents-as-first-class-citizens)

6. [Memory & Context Management: Hierarchical Token Storage](#6-memory--context-management-hierarchical-token-storage)

7. [Device & I/O: Multimodal Tokenized Drivers](#7-device--io-multimodal-tokenized-drivers)

8. [Security Model: Guardrails as System Calls](#8-security-model-guardrails-as-system-calls)

9. [Information Flow Model: The Token Loop](#9-information-flow-model-the-token-loop)

10. [Hardware Specifications Recommendations](#10-hardware-specifications-recommendations)

11. [Engineering Implementation Roadmap](#11-engineering-implementation-roadmap)

12. [Conclusion & Outlook](#12-conclusion--outlook)


## 1. Paradigm Shift: From Bytes to Tokens

> "LLM = CPU, Agent = operating system kernel." — Andrej Karpathy

**Traditional computer**: CPU processes **bytes** → kernel schedules processes → OS serves users  
**AI era**: LLM processes **tokens** → Agent orchestrates tasks → AI OS serves users

| Dimension | Byte Era | Token Era |
| - | - | - |
| Basic unit | Byte (deterministic) | Token (probabilistic) |
| Computation | Precise, deterministic | Statistical, probabilistic |
| Bottleneck | CPU frequency, storage speed | Context window, token transfer |
| State storage | Distributed (client + server) | Centralized (local Agent kernel) |


The core thesis of ICLAW OS: **Large language models run in the cloud or on clusters; only Agent scheduling logic runs locally**. The local machine is no longer a compute center but an **Agent command center**.


## 2. Design Philosophy & Architecture Overview

ICLAW OS is built around **one foundation, two engines, and a five-layer architecture**.

### 2.1 One Foundation

- **LLM = CPU**: Large models in the cloud or clusters provide statistical reasoning capability — the system's "processor".

- **Agent = Kernel**: The local Agent kernel handles task orchestration, context management, and tool dispatching.

### 2.2 Two Engines

- **Context/Memory Engine**: Manages tokenized short-term and long-term memory, supporting hierarchical storage and efficient retrieval.

- **Tool/Skill Engine**: Abstracts file, network, and device capabilities into atomic tools for Agents to invoke.

### 2.3 Five-Layer Architecture

```
┌─────────────────────────────────────────┐  
│         Interaction Layer (AI Shell / UI) │  
├─────────────────────────────────────────┤  
│        Agent Orchestration Layer (Kernel) │  
├─────────────────────────────────────────┤  
│  Context/Memory Engine │ Tool/Skill Engine │  
├─────────────────────────────────────────┤  
│       Device Abstraction Layer (Multimodal) │  
├─────────────────────────────────────────┤  
│            Hardware Layer (CPU/RAM/Network) │  
└─────────────────────────────────────────┘
```


## 3. Kernel Design: AI-Driven Single-Threaded Master Loop

Drawing from Claude Code's **single-threaded master loop** design, the ICLAW OS kernel is an event-driven AI scheduler:

```
while (true) \{  
    task = fetch\_next\_task();        // Get task from token queue  
    context = load\_context(task);    // Load relevant context window  
    result = call\_llm(context);      // Invoke cloud LLM reasoning  
    if (result.has\_tool\_calls()) \{  
        execute\_tools(result.tool\_calls);  
    \} else if (result.is\_complete()) \{  
        output\_result(result);  
    \} else \{  
        push\_subtask(result.next\_step);  
    \}  
\}
```

The kernel does **not** run large models; it runs **Agent logic** only, responsible for:

- Task queue management (priority, dependencies, concurrency)

- Context window allocation and reclamation

- Token stream compression and decompression

- Tool invocation execution and permission checking


## 4. The Two Engines: Context/Memory Engine & Tool/Skill Engine

### 4.1 Context/Memory Engine

- **Three-tier memory system** (inspired by OpenClaw):

  - L0: Real-time workspace (current conversation, sub-millisecond)

  - L1: Agent cache (recent sessions, millisecond)

  - L2: Long-term storage (vector database, millisecond retrieval)

  - L3 (extended): Cloud-hosted compressed history storage

- **Shared prompt caching**: Borrowing Claude Code's `subagent` fork model, multiple Agents can share precomputed KV caches.

- **Self-healing memory**: Periodically detects "context entropy" and automatically compresses or forgets low-value tokens.

### 4.2 Tool/Skill Engine

- **Pluggable tool system**: Each tool is an atomic capability (read file, execute bash, HTTP request, LSP completion, etc.).

- **Tool registration & discovery**: Kernel maintains a tool manifest supporting dynamic loading.

- **Standard tool interface**:

- ```
interface Tool \{  
  name: string;  
  description: string;  
  parameters: JSONSchema;  
  execute(input: any): Promise\<TokenStream\>;  
\}
```

- **Tool invocation as system call**: Agents invoke tools through a standardized ABI; permissions are managed uniformly by the kernel.


## 5. Processes & Scheduling: Agents as First-Class Citizens

### 5.1 Agent Process Model

Each Agent is an **autonomous execution entity** possessing:

- Unique ID

- Independent context window budget

- Priority and scheduling policy

- State (ready / running / waiting for tool / sleeping)

### 5.2 Intelligent Scheduler

Drawing from research such as SchedCP, the scheduler is driven by a small LLM:

- **Dynamically learns workload characteristics** (task type, token consumption, latency sensitivity)

- **Intelligently allocates "compute budget"** (i.e., LLM call frequency and context window size)

- **Achieves application awareness**: foreground interactive Agents get priority; background data-analysis Agents run when idle.


## 6. Memory & Context Management: Hierarchical Token Storage

"Memory" in the token era is the **context window**. ICLAW OS implements a hierarchical context architecture:

| Tier | Name | Medium | Typical Size | Latency |
| - | - | - | - | - |
| L0 | Real-time workspace | RAM | 8K-32K tokens | \<1ms |
| L1 | Agent cache | RAM | 32K-128K tokens | \<10ms |
| L2 | Long-term storage | Local vector DB | Unlimited | \<100ms |
| L3 | Cloud extension | Cloud storage | Unlimited | 1s+ |


**Context compression protocol**: When L0+L1 approaches capacity, the kernel requests the cloud LLM to compress a segment of historical tokens into a **summary token** (a special token representing an entire segment). Summary tokens can be nested, forming a **token tree**.


## 7. Device & I/O: Multimodal Tokenized Drivers

### 7.1 Multimodal Driver Model

Traditional drivers provide byte streams (audio PCM, image RGB). ICLAW OS drivers provide **tokenized perception information**:

- Camera driver → outputs token descriptions of objects/scenes

- Microphone driver → outputs speech recognition token sequences

- Touchpad driver → outputs gesture intent tokens

### 7.2 Cross-Device Unified Scheduling

Through system-level APIs, Agents can transparently access all connected devices (phones, tablets, IoT). Example:

```
agent.call\_device("my\_phone", "camera.capture", \{"query": "capture the document on the desk"\})
```

Returns tokenized document content, not image bytes.


## 8. Security Model: Guardrails as System Calls

### 8.1 Kernel-Level Guardrails

All tool invocations by Agents must pass through the **guardrail engine** for inspection:

- Allow/deny list rules

- User-preset security policies (e.g., "do not delete files")

- Anomaly detection (e.g., excessive sensitive tool calls in a short period)

### 8.2 Fine-Grained Permission Control

Each tool corresponds to a permission. The first time an Agent calls a tool, the kernel prompts the user for authorization (similar to Android permission model). Users can grant "once", "this session", or "permanently".

### 8.3 Privacy by Design

- All session state, context, and user preferences are **stored only locally**

- Tokens uploaded to the cloud are **anonymized** (e.g., replacing entity names, hashing sensitive IDs)

- Cloud LLMs are stateless and do not retain any user data


## 9. Information Flow Model: The Token Loop

Information flows as **probabilistic token streams** among four nodes:

```
          ┌─────────────┐  
          │  Cloud LLM  │ (Statistical reasoning)  
          └──────▲──────┘  
                 │ Completion Tokens  
                 │  
    ┌────────────┴────────────┐  
    │                         │  
Prompt Tokens            Prompt Tokens  
    │                         │  
    ▼                         │  
┌─────────┐               ┌───┴────┐  
│  Local  │◄──Structured──►│Website│  
│Computer │   (Tool call)   │(Tool) │  
│ (Agent) │               └────────┘  
└────┬────┘  
     │ Intent/Result  
     ▼  
┌─────────┐  
│Mobile App│ (Projection terminal)  
└─────────┘
```

**Key characteristics**:

1. **Token as sole currency**: Any bytes entering the Agent are immediately tokenized.

2. **Probabilistic throughout**: LLM outputs carry weights; Agents choose among multiple probabilistic paths.

3. **Websites degrade to tool fields**: Provide only APIs, no UI.

4. **Mobile apps as Agent projections**: Only collect input and render output.

5. **Local as sole state holder**: Cloud is stateless.


## 10. Hardware Specifications Recommendations

Recommended configurations for ICLAW OS local client (Agent command center):

| Use Case | CPU | RAM | Storage | GPU | Network |
| - | - | - | - | - | - |
| Learning / Trial | 2 cores | 8GB | 20GB SSD | Not needed | 20 Mbps |
| Daily development (single Agent) | 4 cores | 16GB | 50GB SSD | Not needed | 100 Mbps |
| Complex project (multi-Agent parallel) | 8 cores | **32GB** | 100GB NVMe | Not needed | 100 Mbps+ |
| Production multi-Agent cluster | 8+ cores | **64-96GB** | 200GB+ NVMe | Optional (local small model) | 500 Mbps+ |


> 32GB is the comfortable baseline because the combined KV cache and context window requirements of multiple concurrent Agents are extremely high.


## 11. Engineering Implementation Roadmap

### Phase 1: Kernel Prototype (AIOS Kernel)

- Study architectures of AIOS, OpenClaw, and Claude Code (leaked code)

- Build minimal token task queue + single-threaded master loop on Linux base

- Experiment: Integrate Qwen-2.5-3B as a loadable kernel module (LKM) into kernel space (optional)

### Phase 2: Tools & Protocols

- Implement Gateway and pluggable tool system (reference OpenClaw)

- Define AI-native application ABI: system call interface between Agent and kernel

### Phase 3: Memory & System Services

- Integrate local vector database (e.g., LanceDB, Qdrant)

- Implement three-tier memory system (L0/L1/L2)

### Phase 4: Interaction Interface

- Develop AI Shell: natural language commands, chain-of-thought visualization, task management panel

- Design intention-centric UI paradigm

### Phase 5: Security & Distribution

- Implement kernel-level guardrail engine

- Build Agent app store and sandbox mechanism


## 12. Conclusion & Outlook

ICLAW OS is not a patch on existing operating systems, but a **complete paradigm shift from bytes to tokens**. It will:

- Treat large models as reasoning CPUs (cloud/cluster)

- Treat Agents as scheduling kernels (local)

- Treat tokens as the universal data unit

- Treat probabilistic as a first-class citizen

In the future, each user's digital world will consist of a **private token tree**, with ICLAW OS as its root node. We invite developers and researchers to participate in this operating system reconstruction.

> "When the underlying unit of data changes, everything above must change." — Andrej Karpathy


**Appendices**

- A. Glossary

- B. References (AIOS, SchedCP, OpenClaw, Claude Code architecture analysis)

- C. Contribution Guidelines


*This whitepaper is based on public materials and architectural reasoning. ICLAW OS is currently in the concept design phase.*

