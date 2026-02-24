---
title: "Post-Quantum Crypto in Python: Benchmarks and Gotchas"
date: 2026-02-24
draft: true
tags: ["pqc", "cryptography", "python", "rust", "fips", "quantum"]
description: "NIST's post-quantum standards are finalized. Here's what it actually takes to use them in Python."
ShowToc: true
TocOpen: true
---

NIST finalized FIPS 203, 204, and 205 in August 2024. ML-KEM (Kyber) for key encapsulation, ML-DSA (Dilithium) for digital signatures, and SLH-DSA (SPHINCS+) for stateless hash-based signatures. The standards are done. The migration is not.

I built [pqc-py](https://github.com/ScrappinR/pqc-py) to make PQC accessible from Python without sacrificing performance or security. Here's what I learned.

## Why Another PQC Library?

There are existing options:

- **liboqs-python** (Open Quantum Safe) — Wraps the C library liboqs. Broad algorithm support, but the Python bindings are thin and the build process requires CMake + C compiler + system-level dependencies.
- **pqcrypto** — Pure Python implementations. Easy to install, but slow and not suitable for production use.
- **PyCryptodome** — No PQC support as of early 2026.

pqc-py takes a different approach: **Rust core with PyO3 bindings**, compiled via maturin into a native Python wheel. The result:

- FIPS 203/204/205 algorithms backed by audited Rust crates (RustCrypto, pqcrypto-rs)
- Automatic key zeroization on drop (the `zeroize` crate)
- Constant-time comparisons where applicable (the `subtle` crate)
- No `unsafe` code in the Rust layer
- Single `pip install` (once wheels are published)
- Classical primitives included (AES-256-GCM, HKDF, HMAC-SHA256) for hybrid schemes

## The Algorithms

### ML-KEM (Kyber) — FIPS 203

Key Encapsulation Mechanism. Two parties establish a shared secret over an insecure channel. The quantum-safe replacement for Diffie-Hellman and RSA key exchange.

Three security levels:
- **ML-KEM-512** (Level 1, ~128-bit security): Smallest keys, fastest
- **ML-KEM-768** (Level 3, ~192-bit security): Recommended default
- **ML-KEM-1024** (Level 5, ~256-bit security): Maximum security

```python
import pqc_py

# Generate keypair
pk, sk = pqc_py.kyber_keygen("level3")

# Sender encapsulates
ciphertext, shared_secret = pqc_py.kyber_encapsulate(pk)

# Receiver decapsulates
shared_secret_2 = pqc_py.kyber_decapsulate(ciphertext, sk)

assert shared_secret == shared_secret_2  # 32-byte shared secret
```

**Key sizes:**

| Level | Public Key | Secret Key | Ciphertext | Shared Secret |
|-------|-----------|-----------|------------|---------------|
| 1 (512) | 800 B | 1,632 B | 768 B | 32 B |
| 3 (768) | 1,184 B | 2,400 B | 1,088 B | 32 B |
| 5 (1024) | 1,568 B | 3,168 B | 1,568 B | 32 B |

Compare to X25519: 32-byte public key, 32-byte secret key. The PQC tax on key sizes is real. Plan your protocols accordingly.

### ML-DSA (Dilithium) — FIPS 204

Digital Signature Algorithm. The quantum-safe replacement for ECDSA, Ed25519, and RSA signatures.

```python
pk, sk = pqc_py.dilithium_keygen("level3")
signature = pqc_py.dilithium_sign(sk, b"message to sign")
valid = pqc_py.dilithium_verify(pk, b"message to sign", signature)
```

**Key and signature sizes:**

| Level | Public Key | Secret Key | Signature |
|-------|-----------|-----------|-----------|
| 2 | 1,312 B | 2,528 B | 2,420 B |
| 3 | 1,952 B | 4,000 B | 3,293 B |
| 5 | 2,592 B | 4,864 B | 4,595 B |

Compare to Ed25519: 32-byte public key, 64-byte signature. Dilithium Level 3 signatures are 50x larger. This matters for bandwidth-constrained applications (IoT, embedded, certificate chains).

### SLH-DSA (SPHINCS+) — FIPS 205

Stateless hash-based signature scheme. The conservative alternative to Dilithium. SPHINCS+ relies only on hash function security — no lattice assumptions. If lattice cryptography is broken, SPHINCS+ still stands.

The trade-off: much larger signatures and slower signing.

```python
# 12 variants: {sha2, shake} x {128, 192, 256} x {fast, small}
pk, sk = pqc_py.sphincs_keygen("sha2-192f")
signature = pqc_py.sphincs_sign(sk, b"message", variant="sha2-192f")
valid = pqc_py.sphincs_verify(pk, b"message", signature, variant="sha2-192f")
```

**Selected variant sizes:**

| Variant | Public Key | Secret Key | Signature |
|---------|-----------|-----------|-----------|
| sha2-128f | 32 B | 64 B | 17,088 B |
| sha2-192f | 48 B | 96 B | 35,664 B |
| sha2-256f | 64 B | 128 B | 49,856 B |
| sha2-128s | 32 B | 64 B | 7,856 B |

The "f" (fast) variants sign faster but produce larger signatures. The "s" (small) variants produce smaller signatures but sign ~10x slower. Pick based on your bottleneck.

**When to use SPHINCS+ over Dilithium:**
- When you need a conservative fallback (no lattice assumptions)
- When signature size isn't a constraint
- When signing speed isn't a constraint
- For long-term signatures on artifacts that must remain valid for decades

## Practical Gotchas

### 1. Key Sizes Break Assumptions

Classical crypto libraries let you store keys in 32-64 byte fields. PQC keys are 800-5000 bytes. If your protocol serializes keys as fixed-length fields, you need to redesign. If your database has a `public_key CHAR(64)` column, it won't hold a Kyber key.

**Fix:** Use variable-length binary fields. Encode keys as base64 or hex for text protocols. Plan for 10x the storage.

### 2. Hybrid Schemes Are the Migration Path

Don't rip out your classical crypto. NIST and every major standards body recommends hybrid key exchange: combine a classical KEM (X25519) with a PQC KEM (ML-KEM) so that security depends on *either* being secure.

```python
import pqc_py

# Hybrid key exchange: X25519 + ML-KEM-768
# Classical part (using your existing library)
classical_shared = x25519_key_exchange(...)

# PQC part
pk, sk = pqc_py.kyber_keygen("level3")
ct, pqc_shared = pqc_py.kyber_encapsulate(pk)

# Combine with HKDF
combined = pqc_py.derive_key(
    classical_shared + pqc_shared,
    salt=b"hybrid-kem-v1",
    info=b"session-key",
    length=32
)
```

### 3. SPHINCS+ Signing Is Slow

SPHINCS+ "fast" variants are fast *for hash-based signatures*. They're still orders of magnitude slower than Dilithium for signing. Verification is fast for both.

If you're signing many documents per second, Dilithium is the right choice. SPHINCS+ is for high-value, low-frequency signing (code signing, certificate issuance, legal documents).

### 4. Zeroization Matters

When a secret key is no longer needed, the memory must be zeroed — not just freed. Standard Python garbage collection doesn't guarantee this. A secret key that's been `del`'d might persist in memory until the page is reused.

pqc-py handles this in Rust with the `zeroize` crate: secret key material is zeroed on drop, before the memory is freed. This is one of the main advantages of the Rust implementation over pure Python.

### 5. Side Channel Resistance

Constant-time operations matter for signing and decapsulation. A timing side channel in Kyber decapsulation would let an attacker recover the secret key by measuring response times. The Rust crate implementations use constant-time comparisons via the `subtle` crate.

Pure Python cannot guarantee constant-time execution. If you're using a pure Python PQC library in production, you have a timing side channel.

## Architecture: Why Rust + PyO3

The Rust + PyO3 + maturin stack solves three problems simultaneously:

1. **Performance.** Rust compiles to native code. No interpreter overhead on the hot path (key generation, signing, verification).
2. **Memory safety.** Rust's ownership model prevents use-after-free, double-free, and buffer overflow bugs in the crypto implementation. No `unsafe` blocks needed.
3. **Key zeroization.** The `zeroize` crate integrates with Rust's drop semantics to zero secret material when it goes out of scope. This happens deterministically, not when the GC gets around to it.

The trade-off: building from source requires a Rust toolchain. Once maturin wheels are published to PyPI for common platforms, this goes away for end users.

## What's Next

- Publishing wheels to PyPI for Linux, macOS, and Windows
- Benchmarking against liboqs-python on identical hardware
- Adding FrodoKEM (conservative lattice-based KEM, not standardized but interesting)
- Hybrid TLS integration examples

The code is on [GitHub](https://github.com/ScrappinR/pqc-py). Apache 2.0 licensed.

---

*Brian James Rutherford is a security researcher focused on post-quantum cryptography and AI security. He builds open-source tools for PQC migration and LLM security analysis.*
