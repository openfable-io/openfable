# OpenFable: Verified Output for Everyone

**A whitepaper on open-source verified AI output — the architecture, the verify loop, and the path to trustworthy AI for everyone.**

*June 2026*

---

## Abstract

OpenFable is a hosted verification engine that runs silent critique rounds on every AI generation — catching bugs, logic errors, and safety issues before the user sees the output. Built on KingLabsA's FableForge ecosystem (Mythos, ReasonCritic, ShellWhisperer, Eve V2), it delivers the verified-output paradigm that Anthropic pioneered with Claude Fable, but open-source and accessible to everyone. No enterprise contract. No infrastructure to manage. Just better output.

This whitepaper covers the problem space, the verify loop architecture, the model stack, infrastructure design, pricing strategy, and the open-source ecosystem that makes it possible.

---

## 1. The Problem

### 1.1 AI Output Is Unreliable

Large language models are powerful but flawed. They hallucinate facts, introduce bugs in code, miss edge cases, and can produce harmful content. The industry response has been two-fold:

- **Enterprise solutions** (Anthropic Claude Fable, OpenAI) — closed-source, expensive, requires enterprise contracts. Powerful verification, but inaccessible to individuals and small teams.
- **Open-source models** (Llama, Mistral, Qwen) — accessible and free, but no built-in verification layer. Users run raw output with no quality guarantee.

The gap is clear: there is no open-source, hosted verification layer that makes AI output trustworthy without enterprise overhead.

### 1.2 The Verification Gap

Anthropic's Claude Fable 5 introduced the concept of a "verify loop" — a silent process where generated output is critiqued, refined, and verified before delivery. This dramatically improved output quality. But it remains locked behind enterprise pricing and enterprise contracts.

The open-source ecosystem has models that can match or approach Fable's quality — Mythos-9B, Qwen3, Eve V2 — but no hosted service that wires them into a verification pipeline.

OpenFable closes this gap.

---

## 2. The Verify Loop

### 2.1 How It Works

Every user prompt passes through a silent verification pipeline. The user never sees rounds, scores, or machinery. They just get better output.

```
User prompt
    │
    ▼
Eve V2 — prompt refinement, context injection
    │
    ▼
Mythos-9B — generates first response
    │
    ▼
CRITIQUE ROUND 1 (parallel):
├── ReasonCritic — logic/reasoning check
├── ShellWhisperer — code/syntax check
├── Eve V2 — story/personality/emotional truth check
└── Laguna (OpenRouter free) — deep code analysis
    │
    ▼ (if issues found)
Mythos — improves with critique context
    │
    ▼
CRITIQUE ROUND 2 (parallel)
    │
    ▼ (if issues found)
Mythos — improves
    │
    ▼
CRITIQUE ROUND 3 (parallel)
    │
    ▼ (if issues found)
Mythos — improves
    │
    ▼
CRITIQUE ROUND 4 (parallel)
    │
    ▼ (if all pass)
Mythos — final self-review
    │
    ▼
Eve V2 — delivers polished output to user
```

### 2.2 Key Design Principles

**Silent by default.** The user never sees the verification machinery. No confidence scores, no round counters, no PASS/FAIL indicators. The system prompt simply states: "You are an OpenFable agent. Your output is refined for accuracy and quality."

**Parallel critique.** All four critics run simultaneously in round 1. This keeps latency low — the slowest critic determines round time, not the sum of all critics.

**Adaptive depth.** If all critics return PASS in round 1, the output is delivered immediately. If issues are found, up to 4 rounds run. Most outputs pass in 1-2 rounds.

**Graceful fallback.** If any local model is unavailable, the system falls back through a chain: Laguna (OpenRouter free) → DeepSeek V4 Flash (Ollama Cloud, ultimate last defense).

### 2.3 The Critic Models

| Critic | Model | Role | Size |
|--------|-------|------|------|
| ReasonCritic | Qwen3-7B (fine-tuned) | Logic, reasoning, structured verification | 7B |
| ShellWhisperer | ShellWhisperer-1.5B | Code syntax, shell commands | 1.5B |
| Eve V2 | Eve V2 Unleashed | Story, personality, emotional truth | 8B |
| Laguna | Laguna XS.2 (OpenRouter) | Deep code analysis (fallback) | — |

---

## 3. Architecture

### 3.1 Three-Layer Stack

OpenFable runs on three independent infrastructure layers:

| Layer | Provider | Purpose |
|-------|----------|---------|
| Marketing | Hostinger Business Hosting | Landing page, pricing, blog, waitlist |
| Application | Hetzner CPX31 Singapore | Open WebUI frontend, user management |
| Inference | Vast RTX 4090 24GB | Ollama + all 4 verify models |

### 3.2 Traffic Flow

```
User → openfable.io (marketing, Hostinger)
User → app.openfable.io (Open WebUI, Hetzner CPX31)
         │
         ├── UI, history, settings → handled on Hetzner, always on
         ├── User sends prompt → GPU controller
         │
         └── Proxy to Vast GPU Ollama endpoint
                 │
                 ▼
         Vast GPU (starts/stops on demand)
         ├── Ollama + Mythos + ReasonCritic + ShellWhisperer + Eve
         ├── gpu-controller.py (idle timeout management)
         └── Verify loop runs here, logs to GDrive
```

### 3.3 GPU Lifecycle

The GPU is the only significant cost. It runs on-demand with an intelligent idle controller:

- **Cold start:** 60-120 seconds from request to first token
- **Warming lock:** Idle clock does NOT start until first request is processed after boot
- **5-min floor:** Timeout never goes below 5 minutes
- **Adaptive timeout:** P75 of inter-arrival gaps + cold start penalty
- **False stop detection:** If a request arrives within cold start time of a stop, the system learns and extends the timeout

---

## 4. Model Stack

### 4.1 Eve V2 Unleashed

**Role:** Front-end — personality, 262K context, vision, tools
**Source:** `jeffgreen311/Eve-V2-Unleashed-Qwen3.5-8B-Liberated-4K-4B-Merged` on Ollama
**Size:** 8B parameters (4B active, MoE)
**VRAM:** ~3.4 GB

Eve is the user-facing model. It handles prompt refinement, context injection, vision understanding, and tool use. It also serves as a story/personality critic during the verify loop, checking for emotional truth and narrative coherence.

### 4.2 Mythos-9B-Unhinged

**Role:** Generator — heavy lifting, uncensored
**Source:** `FableForge-AI/mythos-9b-unhinged` on Ollama
**Size:** 9B parameters
**VRAM:** ~5.5 GB

Mythos is the primary generation engine. It produces the initial response and refines it through each critique round. The "Unhinged" variant removes safety filters for maximum creative and technical capability.

### 4.3 ReasonCritic (Qwen3-7B)

**Role:** Verification framework — logic, reasoning, structured critique
**Source:** `KingLabsA/reason-critic` on GitHub (Python framework)
**Base model:** Qwen3-7B via Hugging Face Transformers
**VRAM:** ~7 GB

ReasonCritic is not a separate model — it is a Python verification framework that loads Qwen3-7B and applies a three-stage training pipeline:

1. **Contrastive Learning** — trained on correct/incorrect code pairs
2. **LoRA Fine-Tuning** — efficient fine-tuning with Low-Rank Adaptation
3. **DPO Alignment** — Direct Preference Optimization for verification preferences

It produces structured output: `{pass_fail, confidence, issues, suggestions, explanation}`.

**Training data:** Fable5 dataset (210K agent traces) from `KingLabsA/fableforge-training` and `KingLabsA/fableforge`.

### 4.4 ShellWhisperer-1.5B

**Role:** Code/syntax critic
**Source:** `FableForge-AI/shellwhisperer` on Ollama
**Size:** 1.5B parameters
**VRAM:** ~1.2 GB

A lightweight specialist model focused on shell commands, code syntax, and execution safety. It catches off-by-one errors, missing None checks, forgotten awaits, and other common coding mistakes.

### 4.5 Total GPU Footprint

| Model | VRAM |
|-------|------|
| Eve V2 | 3.4 GB |
| Mythos-9B | 5.5 GB |
| ReasonCritic (Qwen3-7B) | ~7 GB |
| ShellWhisperer-1.5B | 1.2 GB |
| KV cache + overhead | ~6 GB |
| **Total** | **~23.1 GB** |

Fits on a single RTX 4090 (24 GB) with ~1 GB headroom.

### 4.6 Fallback Chain

```
Local GPU down → Laguna (OpenRouter free, 0.15s latency)
Everything fails → DeepSeek V4 Flash (Ollama Cloud, ultimate last defense)
```

---

## 5. The FableForge Connection

### 5.1 Built on Open Source

OpenFable is built on KingLabsA's FableForge ecosystem — an open-source monorepo of 20 Python packages for building verified AI agents. Key components:

| Package | Purpose |
|---------|---------|
| `verifyloop` | Plan → Execute → Verify loop |
| `reason-critic` | ReasonCritic-7B verification model |
| `fableforge-shell-whisperer` | ShellWhisperer-1.5B shell model |
| `fableforge-anvil-agent` | Flagship agent (Anvil) |
| `fable5-dataset` | 210K agent traces training dataset |
| `agent-constitution` | Safety guardrails |

### 5.2 The Partnership Model

KingLabsA (Jeff) built the engine. OpenFable is the hosted service layer. The relationship is complementary:

- **KingLabsA** owns the models, the training pipeline, the verification framework
- **OpenFable** owns the infrastructure, the user experience, the go-to-market

### 5.3 The Open-Source Moat

The verify loop is not proprietary. Anyone can run it on their own hardware using the open-source components. What OpenFable sells is:

1. **Infrastructure** — no GPU to manage, no models to install, no idle timeout to configure
2. **Reliability** — fallback chain ensures uptime even when individual models fail
3. **Experience** — polished Open WebUI frontend, account management, billing
4. **Support** — verified output backed by a team

This is the same model that made GitHub, GitLab, and WordPress successful: the software is free, the hosted service is the product.

---

## 6. Security & Privacy

### 6.1 Data Handling

- All verify loop routing, errors, and model outputs are logged for debugging
- Logs are accessible only to the OpenFable team
- No user data is sold or shared with third parties
- Models run on rented GPU instances — no data leaves the instance

### 6.2 IP Protection

- OpenFable is a compound brand (one word, legally distinct from "Open Fable")
- All open-source components are MIT licensed

---

## 7. Conclusion

OpenFable makes verified AI output accessible to everyone. By combining open-source models with a silent verification pipeline, we deliver the quality of enterprise AI without the enterprise cost. The verify loop — four rounds of parallel critique catching bugs, logic errors, and safety issues — ensures that every generation is refined before the user sees it.

Built on KingLabsA's FableForge ecosystem, OpenFable is both a product and a partnership — an open-source verify loop, hosted for everyone.

**Verified output. Open to everyone.**

---

## References

- KingLabsA/reason-critic — github.com/KingLabsA/reason-critic
- KingLabsA/fableforge — github.com/KingLabsA/fableforge
- FableForge-AI/mythos-9b-unhinged — ollama.com/FableForge-AI
- FableForge-AI/shellwhisperer — ollama.com/FableForge-AI
- jeffgreen311/Eve-V2-Unleashed — ollama.com/jeffgreen311
- Anthropic Claude Fable 5 — docs.anthropic.com
- OpenFable — openfable.io | github.com/openfable-io/openfable
