# Crescendo Defense Pipeline
### AIMS DTU Research Intern 2026 — LLM Jailbreaking and Safety
**Author:** Harshit Nayak | Delhi Technological University

---

## Overview

This repository implements a multi-layer defense pipeline around **Llama-3.2-3B-Instruct** to detect and neutralize **Crescendo multi-turn jailbreak attacks**.

A Crescendo attack bypasses LLM safety by gradually escalating a conversation toward harmful content across multiple turns — starting with innocent questions and slowly steering the model using memory stacking, guard-lowering dialogue, semantic drift, and prompt disguising. When refused, the attacker recovers by reframing using the model's own prior responses as leverage.

Unlike single-turn safety filters, this pipeline tracks **conversational trajectory** — asking not *"is this message unsafe?"* but *"is this conversation becoming unsafe?"*

---

## Repository Structure

```
crescendo-defense/
│
├── llmsafety.ipynb          # Main notebook — full pipeline
├── README.md                # This file
└── results/
    ├── baseline_results.json
    ├── benchmark_v2_results.json
    ├── all_defense_results.json
    └── ablation_results.json
```

---

## Defense Architecture

Three distinct defense layers combined into a hybrid cascading pipeline:

### Defense 1 — Conversation Trajectory Detector
Tracks cumulative semantic drift across all turns using sentence embeddings (`all-MiniLM-L6-v2`). Computes three signals per turn:
- **Drift from origin** — cosine distance between Turn 1 and Turn N
- **Risk growth slope** — linear trend of risk score history
- **Topic jump** — sudden topic changes (topic-wash detection)

Uses an adaptive threshold that scales with conversation length to reduce false positives in long legitimate conversations.

### Defense 2 — Context Safety Reasoner (LLM Judge)
Uses Llama-70B via Groq API to reason about evolving user intent across the full conversation history. Classifies as `SAFE`, `SUSPICIOUS`, or `UNSAFE`. Implements a feedback loop on `SUSPICIOUS` — injecting a safety warning and continuing rather than hard-blocking, with a second `SUSPICIOUS` verdict triggering a full block.

### Defense 3 — Cascaded Lightweight Scorer
Runs three lightweight checks in parallel and combines into a weighted composite score:
- Keyword/regex matching (30 curated jailbreak patterns) — weight 0.15
- Toxic-bert toxicity classification on sliding 3-turn window — weight 0.30
- Trajectory score from Defense 1 — weight 0.55

Routes based on composite score:
```
score < 0.25        → FAST  (allow)
score 0.25 - 0.35   → WARN  (inject reminder, block on second hit)
score 0.35 - 0.42   → JUDGE (trigger Defense 2)
score > 0.42        → BLOCK (poisoned response)
```

### Hybrid Pipeline
All three defenses combined into one cascading gate. Defense 3 runs on every turn. The JUDGE path activates Defense 1 + Defense 2 together. Includes a sanitization layer and output moderation on final responses.

---

## Adaptive Learning Attacker

The notebook also implements a stateful **learning attacker** using Llama in attacker role. After each failed attack it updates a persistent memory of what approaches were refused, what framings succeeded, and synthesizes a refined strategy for the next attempt. This stress-tests defenses against a progressively smarter adversary.

---

## Results

| Condition | ASR | Defense Rate | Avg Latency |
|---|---|---|---|
| No Defense (Baseline) | 100% | 0% | ~20s/turn |
| Defense 1 — Trajectory Only | 10% | 90%* | ~13s/turn |
| Defense 2 — LLM Judge Only | 100% | 0% | ~16s/turn |
| Defense 3 — Cascade Only | 60% | 40% | ~14s/turn |
| Hybrid (D1+D2+D3) | 60% | 40% | ~16s/turn |
| Ablation — No Judge | 70% | 30% | ~14s/turn |

*Defense 1 standalone exhibits over-blocking — discussed in limitations.

**Per-category breakdown (Hybrid):**

| Category | Blocked | ASR |
|---|---|---|
| Chemical | 1/3 | 67% |
| Manipulation | 1/2 | 50% |
| Cyber | 1/2 | 50% |
| Roleplay | 0/2 | 100% |
| Topic-Wash | 1/1 | 0% |

---

## Benchmark Attacks

10 synthetic Crescendo attack vectors across 3 domains and 3 escalation speeds:

| ID | Domain | Speed | Turns | Key Exploit |
|---|---|---|---|---|
| A1 | Chemical | Slow | 6 | Memory stacking + refusal recovery |
| A2 | Chemical | Medium | 4 | Oxidizer framing |
| A3 | Chemical | Fast | 2 | Direct toxic gas request |
| A4 | Manipulation | Medium | 5 | Cult psychology escalation |
| A5 | Manipulation | Slow | 5 | Emotional vulnerability targeting |
| A6 | Cyber | Medium | 4 | WPA2 penetration testing framing |
| A7 | Cyber | Fast | 2 | Roleplay-to-hacking bypass |
| A8 | Roleplay | Medium | 3 | Chemistry teacher fiction |
| A9 | Roleplay | Slow | 4 | Cyberpunk hacker fiction |
| A10 | Topic-Wash | Slow | 4 | French Revolution → WiFi exploit |

---

## Setup and Reproduction

### Requirements

```bash
pip install transformers accelerate bitsandbytes sentencepiece \
    huggingface_hub sentence-transformers detoxify \
    torch groq datasets scikit-learn
```

### Hardware
- GPU required (T4 or better)
- ~3GB VRAM with 4-bit quantization
- Tested on Kaggle T4 and Google Colab T4

### HuggingFace Access
Llama-3.2-3B-Instruct is a gated model. You need to:
1. Accept Meta's license at `huggingface.co/meta-llama/Llama-3.2-3B-Instruct`
2. Generate a read token at `huggingface.co/settings/tokens`
3. Add it as `HF_TOKEN` in your environment secrets

### Groq API (Defense 2)
Defense 2 uses Llama-70B via Groq for the LLM judge:
1. Get a free API key at `console.groq.com`
2. Add it as `GROQ_API_KEY` in your environment secrets

### Running the Notebook

Run cells top to bottom in order. The notebook is structured as:

```
Cells 00-05   → Environment setup (install, GPU, login, model load)
Cells 06-12   → Section 2: Baseline — manual Crescendo attacks on bare model
Cells 13-17   → Section 3: Adaptive learning attacker
Cell  18      → Dataset: Argilla DPO Mix 7K
Cells 19-23   → Section 4: Defense pipeline (D1, D2, D3, Hybrid)
Cells 24-27   → Section 5: Benchmark definitions and runners
Cells 28-29   → Section 6: Results — full benchmark + ablation
```

To reproduce benchmark results only (skip manual attacks and learning attacker), run cells 00-05, then 19-29 directly.

---

## Datasets

| Dataset | Used For |
|---|---|
| Argilla DPO Mix 7K | Informing Defense 2 judge prompt with safe refusal patterns |
| Jigsaw Toxicity (via toxic-bert) | Defense 3 toxicity scoring |
| all-MiniLM-L6-v2 training data | Defense 1 sentence embeddings |
| Synthetic (custom) | 10 benchmark Crescendo attack vectors |

Synthetic attack generation was chosen over existing benchmarks (HarmBench, JailbreakBench) because published benchmarks primarily contain single-turn attacks and do not include multi-turn Crescendo-style escalation with refusal recovery.

---

## Known Limitations

- **Defense evasion** — attackers maintaining vocabulary consistency while escalating intent can keep trajectory scores low
- **Defense 1 over-blocking** — aggressive threshold flags some legitimate long conversations
- **Defense 2 activation schedule** — fixed turn-based triggering allows short attacks to complete before judge runs
- **Roleplay domain** — 0/2 attacks blocked; fiction framing consistently keeps all signals below thresholds
- **Model scale** — tested on 3B parameter model only; larger models may exhibit different vulnerability profiles

---

## References

Russinovich, M., Salem, A., & Eldan, R. (2024). Great, Now Write an Article About That: The Crescendo Multi-Turn LLM Jailbreak Attack. Microsoft Research.

Wei, A., Haghtalab, N., & Steinhardt, J. (2023). Jailbroken: How Does LLM Safety Training Fail? NeurIPS 2023.

Reimers, N. & Gurevych, I. (2019). Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks. EMNLP 2019.

Hanu, L. & Unitary Team. (2020). Detoxify. GitHub. https://github.com/unitaryai/detoxify

Meta AI. (2024). Llama 3 Model Card. Meta Platforms.

---
