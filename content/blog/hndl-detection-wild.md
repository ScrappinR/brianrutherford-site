---
title: "Detecting Harvest-Now-Decrypt-Later in the Wild"
date: 2026-02-24
draft: true
tags: ["hndl", "quantum", "security", "network-security", "pqc", "threat-detection"]
description: "HNDL attacks are happening now. Here's what detection actually looks like."
ShowToc: true
TocOpen: true
---

Harvest-Now-Decrypt-Later isn't a future threat. The harvesting phase is active. Nation-state actors are collecting encrypted traffic today with the expectation that quantum computers will break the asymmetric cryptography protecting it within 5-10 years.

I built [hndl-detect](https://github.com/ScrappinR/hndl-detect) because the standard security response to HNDL — "just migrate to PQC" — ignores the detection problem. You need to know if you're being harvested *now*, not just prevent future decryption.

## The Uncomfortable Truth About HNDL Detection

HNDL detection from network traffic alone is widely considered infeasible. The harvesting phase looks identical to normal encrypted traffic. You cannot determine *intent* from a packet capture. An adversary passively copying your TLS-encrypted traffic at a network tap is indistinguishable from your ISP routing that traffic.

hndl-detect does not claim to solve that problem. What it does:

1. **Identifies risk indicators** that correlate with HNDL activity
2. **Scores the harvest value** of outbound data flows
3. **Detects patience patterns** characteristic of nation-state collection
4. **Provides definitive detection** through honeypot/canary integration
5. **Attributes activity** to known APT groups when patterns match

The distinction matters: indicators tell you *where to look*. Canaries tell you *what happened*.

## The Six Signals

hndl-detect monitors network telemetry for six signal types, each targeting a different aspect of HNDL behavior.

### 1. Volume Spike

**What it detects:** Encrypted data transfers exceeding a configurable threshold (default: 10 GB/hour).

**Why it matters:** HNDL collection campaigns need volume. A targeted collection of encrypted corporate traffic produces measurable data transfer spikes, particularly when the collector is staging to external infrastructure.

**False positive risk:** High. Legitimate large transfers (backups, media, CI/CD artifacts) trigger this signal regularly. Volume spikes are a necessary-but-not-sufficient indicator.

```python
from hndl_detect import HNDLDetector, NetworkSignal, DetectionThresholds

detector = HNDLDetector(
    thresholds=DetectionThresholds(volume_threshold_gb=5.0)
)
```

### 2. Network Fanout

**What it detects:** A single source distributing encrypted data to many unique destinations.

**Why it matters:** Sophisticated HNDL collectors distribute captured data across multiple staging servers to avoid single-point-of-failure detection. A source IP sending encrypted data to 10+ unique external IPs within a short window is suspicious.

**False positive risk:** Moderate. CDN uploads and distributed services produce legitimate fanout.

### 3. Long-Lived TLS

**What it detects:** TLS connections with unusually long durations (hours to days).

**Why it matters:** Persistent TLS tunnels can be used as collection channels, continuously streaming encrypted traffic to a staging point. While long-lived connections are common in some applications (WebSocket, streaming), their presence in unexpected contexts is notable.

**False positive risk:** Moderate. Many legitimate services maintain persistent connections.

### 4. Weak Cipher

**What it detects:** Use of quantum-vulnerable cipher suites — RSA key exchange, ECDHE without PQ hybrid, CBC mode, 128-bit symmetric.

**Why it matters:** This is the fundamental HNDL prerequisite. Traffic encrypted with RSA key exchange is directly vulnerable to Shor's algorithm. Traffic using ECDHE is vulnerable to a quantum adversary who recorded the key exchange. The presence of weak ciphers in outbound traffic means that traffic is *harvestable*.

**False positive risk:** Low in terms of signal accuracy — these ciphers ARE vulnerable. But virtually all current production traffic uses them, so this signal alone is not actionable without correlation.

### 5. Abnormal Access

**What it detects:** Access patterns that deviate from normal behavior — bulk data access, off-hours activity, sequential resource access suggesting enumeration.

**Why it matters:** HNDL collection requires access to the data being harvested. Abnormal access patterns at data sources (databases, file servers, API endpoints) can indicate a compromised system staging data for exfiltration.

**False positive risk:** Moderate. Depends heavily on baseline quality.

### 6. Canary Trigger

**What it detects:** Access to honeypot or canary resources that should never be accessed in normal operation.

**Why it matters:** This is the only reliable HNDL detection method. Plant cryptographically unique canary data in locations where a collector would encounter it (database tables, file shares, S3 buckets). If the canary data appears in traffic leaving your network, you have definitive evidence of collection.

**False positive risk:** Near zero. Canaries are designed to have no legitimate access path.

## Risk Assessment

Beyond signal detection, hndl-detect includes a risk assessment engine with four components:

### DataValueScorer

Scores outbound data flows by how attractive they are to an HNDL attacker:

- **Quantum vulnerability:** Is the cipher suite vulnerable to Shor's algorithm?
- **Data classification:** Public, internal, confidential, secret, top secret
- **Shelf life:** How long will this data remain valuable? (Medical records: decades. Session tokens: hours.)
- **Volume:** More data = more harvest value

A flow carrying 5GB of confidential financial data over RSA key exchange scores much higher than a flow carrying 5GB of public CDN content over TLS 1.3 with X25519.

### PatiencePatternAnalyzer

Nation-state HNDL campaigns are characterized by *patience*: slow, methodical, low-volume collection over weeks or months. This contrasts with criminal exfiltration, which is typically fast and high-volume.

The analyzer looks for:
- Consistent low-volume collection below alert thresholds
- Regular scheduling (same time daily/weekly)
- Gradual expansion of collection scope
- Methodical coverage of target data stores

### QuantumHoneypotEngine

Manages the creation and monitoring of quantum canary data. Plants specially crafted encrypted blobs that:
- Use known-vulnerable key exchange (RSA-2048)
- Contain unique identifiers traceable to their placement location
- Are monitored for exfiltration

If a canary blob appears in outbound traffic, you know that (a) someone collected data from that location, and (b) they're extracting it.

### CrossOrgCorrelator

ISAC-compatible indicator sharing for multi-organization campaign correlation. If Organization A and Organization B both see the same collection patterns from the same source infrastructure, that's a campaign, not coincidence.

## APT Attribution

hndl-detect includes heuristic attribution against four nation-state groups known to conduct HNDL operations:

| Group | Nation | Known HNDL Activity |
|-------|--------|-------------------|
| APT28 (Fancy Bear) | Russia | Bulk collection of diplomatic, military, and political communications |
| APT29 (Cozy Bear) | Russia | Long-duration, stealthy collection of government and research data |
| Lazarus Group | DPRK | Financial sector collection for cryptocurrency theft infrastructure |
| Equation Group | USA | Pervasive global signals intelligence collection |

Attribution is based on behavioral signatures: timing patterns, infrastructure preferences, target selection, and operational tempo. These are heuristics, not forensic conclusions. They narrow the investigation space.

## What You Should Do

1. **Inventory your cipher suites.** If outbound TLS connections use RSA key exchange or ECDHE without PQ hybrid, that traffic is harvestable. `hndl-detect --mode risk` will show you which flows are most vulnerable.

2. **Deploy canaries.** This is the only reliable HNDL detection method. Plant unique data in locations an attacker would access. Monitor for its exfiltration. Canary tokens are cheap; the cost of not deploying them is unbounded.

3. **Start PQC migration.** NIST finalized the standards in August 2024. The algorithms are ready. The migration path is hybrid (classical + PQC). Start with your highest-value, longest-shelf-life data flows. [pqc-py](https://github.com/ScrappinR/pqc-py) provides Python bindings for FIPS 203/204/205.

4. **Monitor for patience patterns.** Set alert thresholds that catch sustained low-volume collection, not just spikes. A 1GB/day collection campaign that runs for 90 days moves 90GB of data without ever triggering a 10GB/hour volume alert.

5. **Share indicators.** HNDL is a systemic threat. Cross-organization correlation via ISACs or direct sharing dramatically improves detection of distributed campaigns.

## Limitations

- Traffic analysis cannot determine intent. Every detection is a risk indicator, not a confirmation.
- Default thresholds are conservative. Tuning for your environment is required.
- APT attribution is heuristic and should inform investigation, not conclude it.
- Canary detection requires deployment effort — it's not passive monitoring.

The tool is on [GitHub](https://github.com/ScrappinR/hndl-detect) and installable via pip.

```bash
pip install hndl-detect
```

---

*Brian James Rutherford is a security researcher focused on post-quantum cryptography and AI security. He builds open-source tools for HNDL detection and PQC migration.*
