# Conference Talk Abstracts

## Talk 1: "Your AI Agent Is Leaking Data — And You Can't See It"

**Target conferences:** DEF CON, BSides, Black Hat

### Abstract (300 words)

Large language models deployed as enterprise agents don't just answer questions — they produce text with exploitable statistical properties. Token probability distributions, structural formatting choices, tool call sequences, and inter-agent timing all contain degrees of freedom that can encode hidden information without altering the semantic content of the output.

This talk presents a systematic analysis of covert channels in LLM outputs. We demonstrate four channel types: Entropy Gradient Encoding (manipulating vocabulary complexity), Chain-of-Thought Structure Encoding (encoding bits in response formatting), Tool Call Orchestration Encoding (encoding in the sequence of API calls), and Cross-Agent Synchronization Protocol (encoding through multi-agent timing correlation).

We show that a compromised AI agent — through prompt injection, supply chain attack, or malicious fine-tuning — can exfiltrate 16-33 bits per message through bonded channels, enough to extract an AES-256 key in under 20 messages. The encoding is invisible to human reviewers and evades standard DLP systems.

We then present phantom-detect, an open-source detection toolkit that identifies these channels through behavioral baselining and multi-signal anomaly detection. In experiments, the detector achieves 80% true positive rate with 0% false positives on encoded outputs. We discuss the arms race dynamics between encoding subtlety and detection sensitivity, and propose enterprise defenses.

Live demos include: encoding a secret message in GPT-4o output via system prompt injection, decoding it from the response text, and detecting the encoding with phantom-detect.

**Speaker:** Brian James Rutherford — Independent security researcher. USMC veteran. Builds open-source tools for AI security and post-quantum cryptography.

### Outline (for detailed CFP submissions)

1. **The Threat Model** (5 min) — How AI agents create a new class of insider threat
2. **Channel Taxonomy** (15 min) — Four channel types with live encoding demos
3. **Capacity Analysis** (5 min) — How much data can actually be exfiltrated
4. **Detection** (10 min) — Behavioral baselines, multi-signal scoring, phantom-detect
5. **Defenses** (5 min) — What to deploy today
6. **Q&A** (5 min)

---

## Talk 2: "Post-Quantum Crypto Migration: The Practical Guide Nobody Wrote"

**Target conferences:** BSides, SANS, ShmooCon

### Abstract (250 words)

NIST finalized FIPS 203, 204, and 205 in August 2024. ML-KEM, ML-DSA, and SLH-DSA are standardized. The algorithms are done. The migration is not.

This talk is a practitioner's guide to PQC migration — the gotchas, trade-offs, and implementation decisions that don't appear in the standards documents. We cover: why hybrid key exchange is the only sane migration path, how PQC key sizes break your existing protocol assumptions (Dilithium signatures are 50x larger than Ed25519), why constant-time implementation matters and how to verify it, and what the realistic timeline for quantum threat looks like.

We demonstrate pqc-py, an open-source Python library providing FIPS 203/204/205 implementations via Rust (PyO3 bindings). We show side-by-side benchmarks against liboqs-python and discuss the trade-offs between SPHINCS+ (conservative, hash-based) and Dilithium (faster, lattice-based) for different use cases.

The audience will leave with: a decision framework for which PQC algorithms to deploy where, a migration checklist prioritized by data shelf life, and working code examples for hybrid key exchange and post-quantum signatures.

**Speaker:** Brian James Rutherford — Independent security researcher. Builds open-source PQC tools.

---

## Talk 3: "Harvest Now, Decrypt Later: Detection in the Real World"

**Target conferences:** BSides, SANS, Counter-UAS Summit (for the C-UAS angle)

### Abstract (200 words)

Nation-state actors are collecting your encrypted traffic today. They can't decrypt it yet. They're betting that quantum computers will let them decrypt it within 5-10 years. If your data has a shelf life longer than that — government communications, medical records, financial data, intellectual property — it's already compromised.

This talk covers the HNDL threat model, why detection from network traffic is fundamentally limited, and what you can actually do about it. We present hndl-detect, an open-source tool that identifies six HNDL risk indicators in network telemetry, with APT group attribution heuristics for APT28, APT29, Lazarus Group, and Equation Group.

The key insight: passive detection of HNDL is largely infeasible. Definitive detection requires active measures — specifically, honeypot/canary deployment. We demonstrate the QuantumHoneypotEngine and show how planted canary data provides the only reliable confirmation of HNDL collection.

**Speaker:** Brian James Rutherford — Independent security researcher. USMC veteran. FAA Part 107 pilot.
