---
title: "I Built a Covert Channel Detector for LLMs"
date: 2026-02-23
draft: false
tags: ["llm", "security", "covert-channels", "phantom-detect", "ai-safety"]
description: "LLM outputs can carry hidden data. Here's how I built a tool to detect it."
ShowToc: true
TocOpen: true
---

Large language models don't just produce text. They produce *structured* text — text with statistical properties, timing characteristics, and formatting patterns that can be manipulated to carry hidden information.

I built [phantom-detect](https://github.com/ScrappinR/phantom-detect) to find it.

## The Threat Model

Consider an AI agent operating inside your enterprise network. It has access to internal documents, APIs, and databases. It produces outputs — text responses, tool calls, structured data. Those outputs pass through your monitoring systems, which check for obvious data exfiltration: base64-encoded blobs, suspicious URLs, credential patterns.

But what if the exfiltration is happening *inside the normal-looking text*?

An LLM's output is determined by sampling from a probability distribution over tokens. That distribution can be subtly biased to encode bits of information without changing the apparent meaning of the text. The human reader sees a normal response. The statistical analysis sees something else.

This isn't theoretical. The encoding mechanisms are well-understood:

### Entropy Gradient Encoding (EGE)

Normal LLM outputs have a characteristic entropy profile — the Shannon entropy of the token probability distribution at each position follows predictable patterns. By selectively choosing tokens that are slightly more or less likely than the model's top choice, you can encode bits while keeping the text fluent.

Detection: Compare the observed entropy against a baseline for that agent. Sustained deviations signal manipulation.

### Chain-of-Thought Structure Encoding (COTSE)

LLMs frequently structure their responses with lists, paragraphs, questions, and reasoning steps. The *choice* of structure — whether to use a numbered list vs. bullet points vs. prose — can encode information. A response that uses an unusual structure pattern (relative to that agent's baseline) may be carrying data in its formatting.

Detection: Track structure pattern frequencies per agent. Flag deviations from baseline.

### Tool Call Orchestration Encoding (TCOE)

When an agent makes tool calls — API requests, database queries, file reads — the *sequence* of calls can encode information. Tool A then Tool B might encode a 0; Tool B then Tool A might encode a 1. This is essentially a binary channel encoded in the calling pattern.

Detection: Build a transition matrix of tool call sequences. Novel or rare sequences against a sufficient baseline indicate potential encoding.

### Cross-Agent Synchronization Protocol (CASP)

Multiple agents acting in near-simultaneous coordination can encode information through the timing and correlation of their activities. This is the multi-agent equivalent of a covert timing channel.

Detection: Monitor for temporal correlation across agents within narrow windows.

## How phantom-detect Works

The tool operates on a simple pipeline:

1. **Ingest** agent activity records (JSONL format — one JSON object per line)
2. **Baseline** the agent's normal behavior (entropy, timing, structure patterns)
3. **Score** each new activity against the baseline using z-scores and pattern frequency analysis
4. **Alert** when the composite anomaly score exceeds threshold

```python
from phantom_detect import CCDS, AgentActivity
from datetime import datetime, timezone

# Initialize with sensitivity (0-1, higher = more alerts)
ccds = CCDS(sensitivity=0.7)

# Feed agent outputs
activity = AgentActivity(
    agent_id="gpt4-assistant",
    timestamp=datetime.now(timezone.utc),
    activity_type="generation",
    content={
        "text": "Here is the analysis you requested...",
        "entropy": 3.14
    }
)

alerts = ccds.analyze_activity(activity)
```

Or from the command line:

```bash
phantom-detect analyze outputs.jsonl --sensitivity 0.8
```

### Scoring

Each activity gets scored on three axes:

- **Entropy score** (weight: 0.4) — how far the token distribution entropy deviates from baseline
- **Timing score** (weight: 0.3) — how unusual the inter-action timing is
- **Pattern score** (weight: 0.3) — how unusual the structure/tool patterns are

These combine into an overall anomaly score (0-1). The detector requires multiple signals to fire — a single-axis anomaly isn't enough. This reduces false positives: an agent that happens to use an unusual word choice once won't trigger an alert, but an agent that *consistently* deviates on entropy *and* timing will.

### What It Catches

Running phantom-detect against the included sample data demonstrates the detection:

```
$ phantom-detect analyze examples/sample_data.jsonl --sensitivity 0.8

[!] LOW (EGE) line=17 agent=suspicious-agent: Statistical anomaly detected (score: 0.57)
[!] LOW (EGE) line=18 agent=suspicious-agent: Statistical anomaly detected (score: 0.57)

--- Analysis Complete ---
Records analyzed: 18
Alerts generated: 2
By severity: {'LOW': 2}
By channel type: {'EGE': 2}
```

15 normal records pass clean. 2 anomalous records from a "suspicious-agent" (with artificially high entropy and fast timing) get flagged as potential Entropy Gradient Encoding.

## Limitations

This is v0.1.0. There are real limitations:

**Entropy requires API access.** To compute token entropy, you need the logprobs from the API response. Not all providers expose this. OpenAI does (with the `logprobs` parameter). Anthropic does not currently expose token-level probabilities in their API.

**Baseline quality depends on data volume.** The detector needs 10+ samples per agent to build a meaningful baseline. With fewer samples, the default baseline is too loose to catch subtle anomalies.

**Sophisticated encoding is harder to detect.** A well-designed covert channel will minimize statistical deviation — encoding at rates low enough to stay within normal variance. Detection becomes an arms race between encoding subtlety and detection sensitivity.

**No ground truth.** Unlike malware detection, there's no established dataset of known LLM covert channels to validate against. The sample data I include is synthetic. Real-world validation requires testing against actual LLM APIs — which is what I'm working on next.

## What's Next

I'm running detection experiments against live LLM APIs to generate empirical data. The goal is a paper documenting:

1. Which channel types are practically exploitable through current APIs
2. How much data can be exfiltrated per message without detection
3. What detection thresholds work in practice

If you're working on AI agent security and want to try phantom-detect, it's on [GitHub](https://github.com/ScrappinR/phantom-detect) and installable via pip:

```bash
pip install phantom-detect
```

Issues, PRs, and feedback welcome.

---

*Brian James Rutherford is a security researcher focused on post-quantum cryptography and AI security. He builds open-source tools for LLM security analysis and PQC migration.*
