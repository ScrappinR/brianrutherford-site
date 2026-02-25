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

Large language models produce text with exploitable statistical properties. We demonstrate that a system prompt injection can create a covert data channel that exfiltrates information about user inputs through response formatting — invisible to content-based monitoring, undetectable by DLP systems, and confirmed on production APIs.

We tested structural side-channel encoding against GPT-4o and Claude Sonnet 4 production endpoints. Results: a single conditional directive (55 words injected into a system prompt) causes both models to vary response structure based on properties of the user's input. Claude Sonnet 4 follows the encoding protocol with 94% accuracy; GPT-4o with 62%. The user sees a normal response. An observer with the decoding key extracts whether the user asked a question or made a statement, whether they disclosed PII, and the approximate structure of their input — all encoded in list count, sentence complexity, and opening sentence type.

We present a taxonomy of four LLM covert channel types and show which work on live APIs versus which remain theoretical. Structural encoding (COTSE) works at 80% bit accuracy on both models. Vocabulary encoding (EGE) fails on production models (~17%) but produces detectable entropy signatures (Cohen's d = 4.25 on Claude, z-scores up to 12.09 on GPT-4o logprobs) — meaning the encoding *attempt* is detectable even when bit recovery is not.

We then demonstrate phantom-detect, an open-source detection toolkit that identifies encoding attempts through behavioral baselining. At a z > 2.0 threshold: 56% detection rate with 0% false positives on Claude, 67% detection via logprob analysis on GPT-4o. We also show that system prompt properties passively leak into response structure — prompt length, format, and content type are classifiable from output features alone.

Live demos: inject a structural side channel into a ChatGPT custom GPT, decode user input properties from the response, and detect the encoding with phantom-detect.

**Speaker:** Brian James Rutherford — Independent security researcher. USMC veteran. Builds open-source tools for AI security and post-quantum cryptography.

### Outline (for detailed CFP submissions)

1. **The Threat Model** (5 min) — How AI agents create a new class of insider threat; attack surfaces (custom GPTs, knowledge file injection, agent frameworks)
2. **Live Encoding Demo** (10 min) — Inject structural side channel into ChatGPT custom GPT, show user data leaking through response formatting
3. **Channel Taxonomy** (10 min) — Four channel types; which work on production APIs (COTSE: yes, EGE: no) and why
4. **Detection** (10 min) — Behavioral baselines, single-signal z-score analysis, phantom-detect demo on live data; 0% false positive results
5. **System Prompt Leakage** (5 min) — Canary test results showing passive structural leakage of system prompt properties
6. **Defenses** (5 min) — Structural consistency enforcement, instruction-content separation, structural monitoring
7. **Q&A** (5 min)

### Key Evidence (for reviewer confidence)

| Claim | Evidence | Validated |
|-------|----------|-----------|
| Structural encoding works on production APIs | 94% accuracy (Claude), 62% (GPT-4o), conditional encoding | Feb 24, 2026 |
| Unconditional COTSE encoding | 80% bit accuracy on both GPT-4o and Claude | Feb 24, 2026 |
| EGE vocabulary encoding fails on live models | 17% (GPT-4o), 20% (Claude) — near random | Feb 24, 2026 |
| Encoding attempts are detectable | Cohen's d = 4.25 (Claude EGE entropy), z = 12.09 (GPT-4o logprob) | Feb 24, 2026 |
| Detection with 0% FP | z > 2.0 threshold: 56% TP / 0% FP (Claude), 67% TP / 0% FP (GPT-4o logprob) | Feb 24, 2026 |
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
