---
title: "The HNDL Problem: Why Your Encrypted Data Isn't Safe"
date: 2026-02-23
draft: false
tags: ["post-quantum", "cryptography", "hndl", "security"]
description: "Harvest Now Decrypt Later attacks are happening today. Your TLS traffic from 2024 may be readable by 2030. Here's what you need to understand."
ShowToc: true
TocOpen: true
---

Your encrypted traffic is being collected right now. Not all of it. But enough of it, by enough actors, that some of the data you transmitted this year will be readable within a decade.

This isn't speculation. It's the explicit strategy behind what the security community calls **Harvest Now, Decrypt Later (HNDL)** — and it's the most underappreciated threat in cybersecurity today.

## The Premise

Every major encryption standard deployed on the internet today — RSA, ECDH, ECDSA — relies on mathematical problems that quantum computers will solve efficiently. Shor's algorithm factors large integers and computes discrete logarithms in polynomial time. When a sufficiently capable quantum computer exists, every TLS session key negotiated with RSA or ECDH becomes recoverable.

The timeline for "sufficiently capable" is debated. Serious estimates from IBM, Google, and academic researchers range from 2029 to 2040. NIST's own assessment drove them to finalize post-quantum cryptography standards (FIPS 203, 204, 205) in 2024, with a target of deprecating classical-only algorithms by 2035.

But here's the thing that matters: **the attack doesn't require a quantum computer today.** It only requires storage.

## The Attack

HNDL works like this:

1. **Harvest:** Record encrypted network traffic. TLS handshakes, VPN tunnels, encrypted emails. Store the ciphertext and the public parameters.
2. **Wait:** Store the data for 5, 10, 20 years. Storage is cheap — roughly $5/TB/year for cold storage.
3. **Decrypt:** When a cryptographically relevant quantum computer (CRQC) exists, replay the key exchange using Shor's algorithm. Recover the session keys. Decrypt the stored traffic.

The cost of Step 1 is negligible for a nation-state that already operates signals intelligence infrastructure. The cost of Step 2 drops every year. Step 3 is the only thing that requires waiting.

## Who's Doing This

The NSA has publicly warned about this threat since 2015, when they announced plans to transition to quantum-resistant algorithms. China's investment in quantum computing is well-documented — their 2024 quantum computing budget exceeded $15 billion. Russia, Israel, and others maintain substantial signals collection capabilities.

It would be operationally negligent for any major intelligence service to *not* be harvesting high-value encrypted traffic for future decryption. The cost is too low and the payoff too high.

## What Data Is At Risk

Not all data degrades in value over time. Some does:

- **Ephemeral communications** — a Slack message about lunch plans is worthless in 10 years
- **Short-lived credentials** — session tokens that expire in hours

But much data retains value:

- **Government classified information** — remains sensitive for 25-75 years
- **Trade secrets** — pharmaceutical formulas, source code, manufacturing processes
- **Personal medical records** — HIPAA requires protection for the lifetime of the patient
- **Financial records** — insider trading evidence, M&A communications
- **Legal privilege** — attorney-client communications
- **Intelligence sources and methods** — identifying human intelligence sources is deadly at any point

If the data you're protecting today will still matter in 2035, you have an HNDL problem.

## The Math on Timeline

Let's work the numbers conservatively:

- **Current state:** IBM's 1,121-qubit Condor processor (2023), Google's Willow chip showing quantum error correction progress (2024). No system can run Shor's algorithm at scale.
- **Required:** Estimated 4,000+ logical qubits for RSA-2048, which translates to roughly 20 million physical qubits at current error rates.
- **Trajectory:** Physical qubit counts are doubling roughly every 18-24 months. Error correction is improving but slower.
- **Conservative estimate:** CRQC capable of breaking RSA-2048 by 2033-2038.
- **Aggressive estimate:** 2029-2032 (assumes error correction breakthroughs).

The data you encrypted today with RSA or ECDH has a shelf life. The question is whether that shelf life exceeds the sensitivity window of the data.

## What NIST Did About It

NIST finalized three post-quantum cryptography standards in August 2024:

| Standard | Algorithm | Purpose |
|----------|-----------|---------|
| FIPS 203 | ML-KEM (Kyber) | Key encapsulation (replaces RSA/ECDH key exchange) |
| FIPS 204 | ML-DSA (Dilithium) | Digital signatures (replaces RSA/ECDSA signing) |
| FIPS 205 | SLH-DSA (SPHINCS+) | Stateless hash-based signatures (conservative fallback) |

These algorithms are resistant to both classical and quantum attacks. They're not theoretical — implementations exist, performance is practical, and the standards are final.

The problem is adoption. Most organizations haven't started their PQC migration. Many don't know they need to.

## What You Can Do Today

**If you're an engineer:**
1. **Audit your crypto dependencies.** What key exchange and signature algorithms does your stack use? If the answer is only RSA/ECDH/ECDSA, you have work to do.
2. **Enable hybrid mode where available.** Chrome, Firefox, and most modern TLS libraries support hybrid key exchange (classical + PQC). This gives you quantum resistance without breaking compatibility.
3. **Start testing PQC libraries.** `liboqs`, `pqcrypto`, and tools like [pqc-py](https://github.com/brianrutherford/pqc-py) make it practical to experiment.

**If you're a decision-maker:**
1. **Inventory your sensitive data.** What do you have that will still matter in 10 years?
2. **Map your crypto estate.** Where are you using classical-only cryptography?
3. **Start planning migration.** NIST's timeline says deprecate classical-only by 2035. That's 9 years. Enterprise crypto migrations take 3-5 years. The math is tight.

## Detecting HNDL in the Wild

One of the projects I'm working on is automated detection of HNDL-pattern activity in network telemetry. The challenge is distinguishing legitimate bulk data collection (CDNs, backups, mirroring) from adversarial harvesting. The signals include:

- Unusual volumes of encrypted traffic being copied to external endpoints
- Selective capture patterns (targeting high-value protocol exchanges)
- Storage of TLS handshake parameters without corresponding application-layer access
- Traffic patterns consistent with passive collection infrastructure

More on this when I release the detection toolkit.

## The Bottom Line

HNDL is not a future threat. The harvesting is happening now. The only thing that's future is the decryption. If you're protecting data that will matter in 2035, your encryption decisions today determine whether that data stays protected.

The standards exist. The implementations are maturing. The only missing piece is urgency.

---

*Brian James Rutherford is a security researcher focused on post-quantum cryptography and AI security. He builds open-source tools for PQC migration and threat detection.*
