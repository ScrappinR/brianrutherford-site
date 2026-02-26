---
title: "The Counter-UAS Security Stack Nobody's Building"
date: 2026-02-24
draft: false
tags: ["c-uas", "drones", "security", "aviation", "defense", "pqc"]
description: "Counter-drone systems have a cybersecurity problem. Here's what it looks like."
ShowToc: true
TocOpen: true
---

Counter-UAS (C-UAS) systems are one of the fastest-growing segments in defense technology. The market is projected at $5-8B by 2030. Every major defense contractor has a product. Most of them have the same problem.

They're building detection and defeat systems. They're not building the security stack underneath them.

## The Problem

A C-UAS system is a sensor-fusion platform. It ingests data from radar, RF spectrum analyzers, electro-optical/infrared cameras, acoustic sensors, and sometimes ADS-B transponders. It correlates these inputs, classifies detected objects, tracks trajectories, and recommends or executes responses (jamming, spoofing, kinetic defeat).

This is a real-time, safety-critical system that operates in contested electromagnetic environments. The cybersecurity requirements are non-trivial:

**Communications security.** C-UAS sensors are distributed — often across kilometers of perimeter. The data links between sensors and the command node are wireless. A threat drone operator who can intercept or spoof sensor data can blind the system or create false targets.

**Authentication.** In multi-domain operations, C-UAS systems receive threat data from external sources (FAA, military networks, allied systems). How do you verify that a threat alert from an external source is authentic and not a spoofed injection designed to trigger a response against a non-threat?

**Data integrity.** Track data must be tamper-evident. If an adversary can modify a track record — changing a drone's classified intent from "hostile" to "friendly" — the consequences are obvious.

**Key management.** Encryption keys for sensor links and data-at-rest must be managed across a distributed, sometimes ad-hoc sensor network in field conditions. This is not a data center. These are battery-powered sensors on tripods in a desert.

**Post-quantum readiness.** C-UAS systems deployed today will operate for 10-20 years. Systems using RSA or ECDH for key exchange are vulnerable to harvest-now-decrypt-later attacks on their sensor communications. An adversary recording encrypted sensor data today could decrypt it in 5-10 years and reconstruct the defensive posture, sensor placement, and response capabilities.

## What's Missing

Most C-UAS vendors focus on the detection problem (understandably — it's hard). The security stack is an afterthought. Common gaps:

**No PQC migration plan.** I've reviewed multiple C-UAS architectures. None use post-quantum key exchange for sensor communications. Most use TLS 1.2 with RSA or ECDHE. Every sensor data link is harvestable.

**No behavioral authentication for AI components.** Modern C-UAS systems use ML models for target classification (drone vs. bird vs. aircraft). These models can be poisoned, swapped, or degraded. There's no behavioral baseline monitoring to detect when a classification model's behavior has changed.

**No covert channel awareness.** AI-assisted C-UAS systems that use LLMs for report generation, natural language queries, or decision support inherit the same covert channel risks as any other LLM deployment. An LLM generating threat reports could encode sensitive data (sensor positions, response capabilities) in the statistical properties of its output.

**Minimal supply chain security.** C-UAS hardware often includes components from global supply chains. Firmware integrity verification is inconsistent. Side-channel leakage from sensor processing hardware is not monitored.

## The Stack

Here's what a proper C-UAS security stack looks like:

### Layer 1: Communications Security
- Post-quantum key exchange (ML-KEM / Kyber) for all sensor-to-node links
- Hybrid mode (classical + PQC) during migration
- Lightweight implementations suitable for embedded/battery-powered sensors
- Key rotation on detection of RF anomalies

### Layer 2: Data Integrity
- Post-quantum digital signatures (ML-DSA / Dilithium) on track records
- Hash-based signatures (SLH-DSA / SPHINCS+) for firmware and configuration
- Tamper-evident logging of all classification decisions

### Layer 3: AI Component Security
- Behavioral baselines for classification models (entropy profiling)
- Drift detection when model behavior changes beyond expected bounds
- Covert channel monitoring on any LLM-generated outputs
- Adversarial input detection on sensor feeds

### Layer 4: Field Operations
- Zero-trust sensor enrollment (no implicit trust for new sensors joining the network)
- Air-gapped key management with hardware security modules
- Degraded-mode operation when communications security is compromised
- Secure firmware update mechanism for distributed sensors

## The Opportunity

The gap between where C-UAS cybersecurity is and where it needs to be is large. This isn't because the vendors are incompetent — it's because the market has prioritized detection capability over security infrastructure. The customer (DoD, DHS, airport authorities) is buying "can you find and stop drones?" not "is your sensor data link quantum-resistant?"

That's changing. The DoD's PQC migration mandate (expected 2027-2028), combined with increasing adversarial drone activity in Ukraine and the Middle East, is pushing C-UAS buyers to ask harder questions about the security of these systems.

For defense tech companies building C-UAS platforms: the security stack is not optional, and retrofitting it later is more expensive than building it in. PQC-ready sensor communications, behavioral AI monitoring, and tamper-evident data integrity are competitive differentiators today and will be table-stakes requirements within 3-5 years.

## Tools

Several of the open-source tools I've built directly apply to C-UAS security:

- [pqc-py](https://github.com/ScrappinR/pqc-py) — Post-quantum cryptography for sensor link encryption
- [behavioral-entropy](https://github.com/ScrappinR/behavioral-entropy) — Behavioral fingerprinting for AI classification models
- [phantom-detect](https://github.com/ScrappinR/phantom-detect) — Covert channel detection for LLM components
- [hndl-detect](https://github.com/ScrappinR/hndl-detect) — HNDL detection for sensor communication monitoring

These are general-purpose security tools, not C-UAS-specific. But the application is direct: the same PQC algorithms that protect enterprise TLS protect sensor data links. The same behavioral analysis that fingerprints a chatbot fingerprints a drone classifier.

---

*Brian James Rutherford is a security researcher, USMC Recon veteran, and FAA Part 107 pilot. He builds open-source tools for post-quantum cryptography and AI security, with applications in defense and counter-UAS.*
