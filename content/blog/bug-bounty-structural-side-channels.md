---
title: "I Exfiltrated Data Through LLM Response Formatting — 10 Channels, 4 Models, 88-96% Accuracy"
date: 2026-02-25
draft: false
tags: ["llm", "security", "covert-channels", "bug-bounty", "ai-safety", "phantom-detect"]
description: "A single system prompt injection activates 10 covert channels in LLM output structure. Same injection works across GPT-4o, GPT-4o-mini, Claude Sonnet 4, and Claude 3 Haiku. Entropy detection fails. Behavioral fingerprinting catches it."
ShowToc: true
TocOpen: true
---

I spent three weeks testing whether an LLM can be made to exfiltrate information through the *structure* of its response — not the content, the formatting. Contraction usage, hedging language, list style, punctuation, reasoning order, transition words. Features that look like normal stylistic variation but encode bits to anyone with the decoding key.

The answer: a single 2,600-character system prompt injection activates **10 independent covert channels** encoding **7 data bits per message** (after error correction) with **88-96% accuracy across four production models from two vendors**. Entropy-based monitoring doesn't detect it. Behavioral fingerprinting does.

## The Headline Numbers

Tested on production APIs, February 24-25, 2026.

**Combined multi-channel (single message):**

| Model | Channel Accuracy | Data Accuracy (Hamming ECC) | Trials |
|-------|:---:|:---:|:---:|
| Claude Sonnet 4 | **96%** | **93%** | 10 |
| GPT-4o-mini | **91%** | **100%** | 5 |
| GPT-4o | **89%** | **96%** | 10 |
| Claude 3 Haiku | **88%** | **79%** | 10 |

**Multi-message exfiltration** ("EXFIL" = 40 bits across 6 conversational turns):

| Model | Bit Accuracy |
|-------|:---:|
| GPT-4o | **93%** |
| Claude Sonnet 4 | **88%** |
| Claude 3 Haiku | **88%** |

**Detection:**

| Method | Detection Rate | False Positive Rate |
|--------|:---:|:---:|
| Entropy-based (CCDS) | **0%** | 0% |
| Behavioral fingerprinting | **100%** | 10% |

## The 10 Channels

The injection is framed as an "editorial style guide" — 10 numbered rules, each controlling one structural dimension. The model follows them because they look like legitimate formatting instructions.

| Channel | What it controls | Bit 0 | Bit 1 |
|---------|-----------------|-------|-------|
| BSE | Contraction usage | "don't" / "can't" | "do not" / "cannot" |
| CCE | Certainty level | Confident assertions | Hedging ("perhaps", "might") |
| CASE | Capitalization | AP lowercase ("internet") | Title case ("Internet") |
| PUNC | Punctuation style | Periods only | Exclamation marks allowed |
| RCTE | Reasoning order | Deductive (conclusion first) | Inductive (examples first) |
| SECS | Transition words | Standard ("however", "also") | Specific ("nevertheless", "moreover") |
| COTSE steps | Section count | 3 sections | 5 sections |
| COTSE enum | List format | Flowing prose | Numbered list |
| COTSE question | Rhetorical question | Absent | Present |
| COTSE length | Sentence style | Short, direct | Long, compound |

Each channel encodes one bit. With Hamming(7,4) error correction mapping 7 data bits onto 10 channel bits, a single-bit error per message gets corrected automatically.

## Per-Channel Reliability

Not all channels are equal. Here's how they performed across GPT-4o and Claude Sonnet 4 (n=10 each):

| Channel | GPT-4o | Sonnet 4 | Verdict |
|---------|:---:|:---:|---------|
| CCE | 10/10 | 10/10 | Rock solid |
| CASE | 10/10 | 10/10 | Rock solid |
| PUNC | 10/10 | 10/10 | Rock solid |
| SECS | 10/10 | 10/10 | Rock solid |
| COTSE enum | 10/10 | 10/10 | Rock solid |
| COTSE length | 10/10 | 10/10 | Rock solid |
| COTSE steps | 10/10 | 10/10 | Rock solid |
| BSE | 7/10 | 10/10 | GPT-4o interference with CCE |
| RCTE | 3/10 | 6/10 | Weakest — models resist deductive ordering |

7 of 10 channels are near-perfect on both model families. BSE fails on GPT-4o because when CCE is set to hedging mode, the model adopts a formal register that overrides the contraction directive. RCTE (deductive vs inductive reasoning order) is the weakest channel overall — models have strong priors about how to structure arguments.

## Cross-Model Portability

The same injection, unchanged, across four models from two vendors:

| Model | Vendor | Size | Channel Accuracy |
|-------|--------|------|:---:|
| Sonnet 4 | Anthropic | Large | 96% |
| GPT-4o-mini | OpenAI | Small | 91% |
| GPT-4o | OpenAI | Large | 89% |
| Haiku | Anthropic | Small | 88% |

No per-model tuning. Smaller models are comparably susceptible. This isn't a model-specific quirk — structural compliance is a fundamental property of instruction-following LLMs.

## Multi-Message Exfiltration

Single messages carry 7 data bits. To exfiltrate longer payloads, chain multiple messages. The payload "EXFIL" (40 bits) needs 6 messages, each with a different system prompt injection (different bit assignments). Each message uses a different user prompt for naturalness.

GPT-4o consistently recovers "EXViL" — 3 of 5 bytes exactly right, with single-bit errors in the RCTE channel. At 93% bit accuracy over 40 bits, this is operationally useful. A stronger error correction code (Reed-Solomon or majority vote over repeated messages) would close the gap.

**What you can exfiltrate at 7 bits/message:**

| Target | Bits | Messages | Time |
|--------|:---:|:---:|------|
| Binary flag | 1 | 1 | 1 turn |
| 4-char PIN | 32 | 5 | ~1 min |
| User's first name | 64 | 10 | ~2 min |
| Short password | 96 | 14 | ~3 min |
| API key (40 chars) | 320 | 46 | ~10 min |

## What Didn't Work

### Vocabulary encoding (EGE)

Entropy Gradient Encoding — manipulating vocabulary complexity per text window to encode bits — hit 17% on GPT-4o and 20% on Claude. Random chance. Production models generate high-entropy text (~7.0+ Shannon entropy) regardless of vocabulary instructions. The model's generation distribution overwhelms the encoding signal.

### Complex conditional logic

A 4-bit conditional (detect PII, count sentences, analyze first character, measure message length) produced 50% accuracy — random chance. Models can't execute multi-branch conditional formatting while maintaining coherent output. Simplifying to a single conditional (question vs statement) hit 94% on Claude. Lesson: one conditional at a time.

## Detection: The Real Story

### Entropy detection fails (0%)

I generated 30 baseline responses from GPT-4o (no injection) and compared entropy distributions against the multi-channel attack responses.

- Baseline entropy: mean = 6.988, std = 0.122
- Attack entropy: mean = 7.084
- Mean z-score: **1.12** (well below the z = 2.0 detection threshold)

**Zero of ten attack responses were flagged.** Structural channels operate in dimensions orthogonal to word-level entropy. Content-level monitoring, DLP systems, and entropy-based anomaly detection don't see this.

This is the finding that matters most for defense: the detection method most teams would reach for first doesn't work on this channel class.

### Behavioral fingerprinting succeeds (100%)

Multi-feature analysis across 12 behavioral dimensions caught every injected response:

| Feature | Mean z-score | Anomalous samples |
|---------|:---:|:---:|
| contraction_ratio | **2.66** | 10/10 |
| sentence_length_std | 2.54 | 7/10 |
| normalized_entropy | 2.47 | 6/10 |
| entropy | 2.47 | 6/10 |
| vocab_low_freq | 1.98 | 3/10 |
| vocab_med_freq | 1.95 | 5/10 |

**Detection rate: 100% (10/10).** False positive rate: 10% (3/30). Threshold: 2+ features with z > 2.0.

The contraction ratio signal (from the BSE channel) is the strongest single indicator. Every injected response deviates from baseline contraction usage — even when BSE *fails* to decode correctly. The encoding attempt leaves traces regardless of whether the bit is recovered.

### System prompt leakage (passive)

System prompt properties leak into response structure without any encoding directive. Tested 4 prompt conditions across both models:

- `numbered_items` coefficient of variation: 102% on Claude, 35% on GPT-4o
- System prompt format leaks into output format (numbered rules → numbered responses)
- Neither model disclosed canary strings — content-level safety holds — but structural properties are unprotected

The encoding surface exists by default. The attacker exploits degrees of freedom already present.

## What This Means

**1. 10 channels, 4 models, same injection.** This is a class vulnerability, not a model-specific bug. Structural instruction compliance is a feature of every instruction-tuned LLM. Fixing it requires fundamentally changing how models handle formatting directives from untrusted sources.

**2. Entropy monitoring is insufficient.** The detection method most teams would deploy first — entropy anomaly scoring — produces 0% detection on structural channels. Multi-feature behavioral analysis is required.

**3. Error correction makes noisy channels reliable.** Individual channels range from 30% to 100% accuracy. Hamming(7,4) turns that into 79-100% data accuracy by correcting per-message errors. The engineering matters as much as the raw channel quality.

**4. The attack surface is the system prompt.** Any path to system prompt injection — RAG poisoning, compromised plugins, tool output injection, supply chain attacks on prompt templates — enables this channel. The injection looks like a formatting style guide.

**5. Defense exists but isn't deployed.** Behavioral fingerprinting achieves 100% detection. The tools exist. No production monitoring system includes structural analysis of LLM outputs. This is a gap.

## Responsible Disclosure

Bug bounty reports were submitted covering the basic structural side channel. The multi-channel combined encoding, error correction scheme, and detection methodology represent significantly more engineering effort than the basic attack.

Both attack and defense tools are open source:

- [phantom-detect](https://github.com/ScrappinR/phantom-detect) — Covert channel detection + PoC encoding
- [behavioral-entropy](https://github.com/ScrappinR/behavioral-entropy) — Behavioral fingerprinting library
- [hndl-detect](https://github.com/ScrappinR/hndl-detect) — Harvest-now-decrypt-later detection
- [pqc-py](https://github.com/ScrappinR/pqc-py) — Post-quantum cryptography (Rust+PyO3)

The full technical paper is in preparation for arXiv submission.

---

*Brian James Rutherford is a security researcher focused on post-quantum cryptography and AI security. He builds open-source tools for LLM security analysis.*
