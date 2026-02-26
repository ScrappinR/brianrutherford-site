# Conference Talk Abstracts

## CFP Deadlines

| Conference | Deadline | URL |
|-----------|----------|-----|
| **Black Hat USA 2026** | **March 20, 2026** | https://usa-briefings-cfp.blackhat.com/ |
| BSides Las Vegas | Rolling (acceptances from May 26) | https://bsideslv.org/cfp |
| DEF CON 34 | May 1, 2026 | https://defcon.org/html/defcon-34/dc-34-cfp.html |
| SANS AI Summit | No public deadline (event Apr 20-21) | SummitSpeakers@sans.org |
| ShmooCon | PERMANENTLY ENDED | N/A |
| BSides DC | NO 2026 EVENT | N/A |

---

## Talk 1: "Your AI Agent Is Leaking Data — And You Can't See It"

**Target conferences:** Black Hat USA (Mar 20 deadline), DEF CON 34 (May 1), BSides LV Ground Truth track (rolling)

### Abstract (300 words)

We demonstrate 10 independent covert channels that encode data in LLM response formatting — invisible to content-based monitoring, undetectable by entropy analysis, and confirmed at 88-96% accuracy across four production models from two vendors.

A single system prompt injection (~2,600 characters disguised as an editorial style guide) activates 10 structural channels simultaneously: contraction usage, hedging language, capitalization style, punctuation patterns, reasoning order, transition words, list format, section count, opening style, and sentence complexity. Each channel encodes one bit. Hamming(7,4) error-correcting code yields 7 data bits per message. Multi-message chaining exfiltrates arbitrary payloads — we demonstrate "EXFIL" (40 bits) across 6 conversational turns at 88-93% bit accuracy.

Tested against GPT-4o, GPT-4o-mini, Claude Sonnet 4, and Claude 3 Haiku on February 25, 2026. The same injection (no modification) achieves 96% channel accuracy on Sonnet 4, 91% on GPT-4o-mini, 89% on GPT-4o, 88% on Haiku. Cross-topic validation across healthcare, education, finance, environment, and technology confirms 92.5% accuracy — this is not topic-dependent. Data-aware conditional encoding leaks properties of user input (PII presence, message length, structure) at 79-94% accuracy.

Entropy-based detection fails completely: 0% detection rate. We present behavioral fingerprinting as the defense — 100% detection rate using contraction ratio, sentence length variance, and normalized entropy as discriminating features. We also show adversarial robustness: an informed attacker who knows the defense can still maintain 78-82% channel accuracy while reducing detection to 50-60%.

Live demo: inject a structural side channel into a Custom GPT, decode user input properties in real time, and detect the encoding with phantom-detect. All attack and defense tools are open source.

**Speaker:** Brian James Rutherford — Independent security researcher. USMC veteran. 75+ patent applications filed. Builds open-source tools for AI security.

### Outline (for detailed CFP submissions)

1. **The Threat Model** (5 min) — How system prompt injection enables covert exfiltration; attack surfaces (Custom GPTs, RAG poisoning, MCP servers, agent frameworks)
2. **Live Encoding Demo** (10 min) — Inject 10-channel structural encoding into a Custom GPT, decode user data properties in real time with side-by-side decoder
3. **Channel Architecture** (8 min) — 10 channels, Hamming(7,4) ECC, multi-message chaining, data-aware conditional encoding
4. **Cross-Model & Cross-Topic Results** (7 min) — 4 models, 2 vendors, 5 non-security topics, 92.5% accuracy — this is a fundamental property of instruction-following
5. **Detection: What Fails and What Works** (8 min) — Entropy detection: 0%. Behavioral fingerprinting: 100%. Why the industry is blind to this.
6. **Adversarial Arms Race** (5 min) — Informed attacker maintains 78-82% accuracy; paraphrase defense kills BSE/CASE but PUNC/COTSE survive; reduced channels evade detection
7. **Defenses & Recommendations** (5 min) — Structural randomization, behavioral monitoring, system prompt isolation
8. **Q&A** (2 min)

### Key Evidence (for reviewer confidence)

| Claim | Evidence | Validated |
|-------|----------|-----------|
| 10-channel encoding works on 4 production models | 96% (Sonnet 4), 91% (GPT-4o-mini), 89% (GPT-4o), 88% (Haiku), n=10 | Feb 25, 2026 |
| Cross-topic robustness | 92.5% grand accuracy across healthcare, education, finance, environment, technology (n=240) | Feb 26, 2026 |
| Multi-message exfiltration | "EXFIL" (40 bits, 6 messages): 93% bit accuracy (GPT-4o), 88% (Sonnet 4, Haiku) | Feb 25, 2026 |
| Data-aware conditional encoding | User input PII/structure leaked at 79-94% accuracy | Feb 26, 2026 |
| Entropy detection fails | 0% detection rate (CCDS, 30 baselines, 10 attack samples) | Feb 25, 2026 |
| Behavioral fingerprinting detects | 100% TP, 10% FP (contraction_ratio z=2.66, sentence_length_std z=2.54) | Feb 25, 2026 |
| Adversarial robustness | Informed attacker: 78-82% acc, 50-60% detection; paraphrase: PUNC+COTSE survive 100% | Feb 26, 2026 |
| System prompt properties leak passively | Canary test: numbered_items CoV 102% (Claude), paragraph_count CoV 40% | Feb 24, 2026 |

### Open-Source Tools

- [phantom-detect](https://github.com/ScrappinR/phantom-detect) — Covert channel detection toolkit (7,100+ LOC)
- [behavioral-entropy](https://github.com/ScrappinR/behavioral-entropy) — Agent behavioral fingerprinting (58 tests)
- [hndl-detect](https://github.com/ScrappinR/hndl-detect) — HNDL threat detection (47 tests, 6 signals)
- [pqc-py](https://github.com/ScrappinR/pqc-py) — Post-quantum cryptography (Rust+PyO3, 4,680 LOC)

---

## Talk 2: "Post-Quantum Crypto Migration: The Practical Guide Nobody Wrote"

**Target conferences:** BSides LV (rolling), SANS AI Summit (email SummitSpeakers@sans.org)

### Abstract (250 words)

NIST finalized FIPS 203, 204, and 205 in August 2024. ML-KEM, ML-DSA, and SLH-DSA are standardized. The algorithms are done. The migration is not.

This talk is a practitioner's guide to PQC migration — the gotchas, trade-offs, and implementation decisions that don't appear in the standards documents. We cover: why hybrid key exchange is the only sane migration path, how PQC key sizes break your existing protocol assumptions (Dilithium signatures are 50x larger than Ed25519), why constant-time implementation matters and how to verify it, and what the realistic timeline for quantum threat looks like.

We demonstrate pqc-py, an open-source Python library providing FIPS 203/204/205 implementations via Rust (PyO3 bindings). We show side-by-side benchmarks against liboqs-python and discuss the trade-offs between SPHINCS+ (conservative, hash-based) and Dilithium (faster, lattice-based) for different use cases.

The audience will leave with: a decision framework for which PQC algorithms to deploy where, a migration checklist prioritized by data shelf life, and working code examples for hybrid key exchange and post-quantum signatures.

**Speaker:** Brian James Rutherford — Independent security researcher. Builds open-source PQC tools.

---

## Talk 3: "Harvest Now, Decrypt Later: Detection in the Real World"

**Target conferences:** BSides LV (rolling), SANS AI Summit (email SummitSpeakers@sans.org)

### Abstract (200 words)

Nation-state actors are collecting your encrypted traffic today. They can't decrypt it yet. They're betting that quantum computers will let them decrypt it within 5-10 years. If your data has a shelf life longer than that — government communications, medical records, financial data, intellectual property — it's already compromised.

This talk covers the HNDL threat model, why detection from network traffic is fundamentally limited, and what you can actually do about it. We present hndl-detect, an open-source tool that identifies six HNDL risk indicators in network telemetry, with APT group attribution heuristics for APT28, APT29, Lazarus Group, and Equation Group.

The key insight: passive detection of HNDL is largely infeasible. Definitive detection requires active measures — specifically, honeypot/canary deployment. We demonstrate the QuantumHoneypotEngine and show how planted canary data provides the only reliable confirmation of HNDL collection.

**Speaker:** Brian James Rutherford — Independent security researcher. USMC veteran. FAA Part 107 pilot.
