---
title: "Aztec Plonk Verifier Bug: Forging Proofs via the Identity Point"
subtitle: "Aztec's Plonk Verifier and the Degenerate Identity Point"
date: 2026-02-19
description: "A degenerate edge case in elliptic curve pairing let an attacker forge valid-looking proofs from all-zero inputs in Aztec's Plonk verifier."
tags: ["zk", "proof-systems", "aztec"]
---

# The Proof That Was Just Zeros
### Aztec's Plonk Verifier and the Degenerate Identity Point

> **ZK Exploits Series**
>
> **Vulnerability class:** `PROOF SYSTEM` — Degenerate edge case in elliptic curve pairing check
>
> **Protocol:** Aztec Protocol (ZK privacy L2 on Ethereum)
>
> **Chain:** Ethereum
>
> **Year:** 2021

---

| | |
|---|---|
| **Date** | 2021 |
| **Protocol** | Aztec Protocol (ZK privacy L2) |
| **Chain** | Ethereum |
| **Funds at Risk** | All TVL in Aztec — any withdrawal could be faked with zero knowledge of any secret |
| **Funds Stolen** | None — found internally by the Aztec team |
| **References** | [0xPARC ZK Bug Tracker #7](https://github.com/0xPARC/zk-bug-tracker#7-aztec-plonk-verifier-0-bug) |

---

## Background

Aztec used a custom **Plonk** verifier. Plonk proofs consist of polynomial commitment openings — the prover commits to polynomials using elliptic curve points, and the verifier checks them via a **pairing equation** over BN254.

---

## The Vulnerability

Aztec's Plonk verifier had no check that rejected proof inputs containing the **elliptic curve identity element** (the point at infinity, typically represented as `(0, 0)` in affine coordinates).

This is a degenerate case with a specific algebraic property: the pairing of the identity element with *any* other elliptic curve point always evaluates to the **multiplicative identity** (1) in the target group. If all polynomial commitments in a Plonk proof are set to the identity element, every pairing in the verification equation evaluates to 1:

```
e(O, G₂) = 1
e(G₁, O) = 1
```

Where `O` is the identity point. The resulting verification equation degenerates to `1 = 1` — which is trivially true regardless of what the proof is claiming to prove.

An attacker could submit a "proof" consisting entirely of identity points as a legitimate withdrawal from Aztec. The verifier would accept it. No knowledge of any note, spending key, or circuit witness required — just send all zeros.

---

## The Fix

Reject any proof containing the identity point before running the pairing check:

```solidity
function verify(Proof calldata proof) public {
    require(!proof.commitment_a.isIdentity(), "invalid commitment");
    require(!proof.commitment_b.isIdentity(), "invalid commitment");
    // ... check all commitments, then proceed to pairing
}
```

---

## Why This Is Different From Circuit Bugs

This vulnerability lives in the **on-chain Solidity verifier**, not the circuit. The circuit and witness generation were correct. Most ZK security tooling focuses on the circuit layer; the verifier is often treated as a straightforward translation of the verification algorithm. Any pairing-based verifier that does not explicitly reject identity-point inputs should be considered suspect.

---

## How to Catch This

**Why it happens:** Verifier contracts are often generated mechanically (e.g., by snarkjs `exportVerifier`) or hand-written from the mathematical spec. The spec says "check `e(A, B) = e(C, D)`." It doesn't say "first check that none of `A, B, C, D` are the identity point." The identity point is a valid curve element — it doesn't trip any `require(isOnCurve(...))` check. The bug is a missing **precondition**, not a missing **constraint**.

**What to check:**

1. **Reject identity points at the top of the verifier.** Before any pairing computation, verify each G1 and G2 proof element is not the point at infinity. In Solidity, for affine BN254 G1 points: `require(proof.x != 0 || proof.y != 0)`.
2. **This applies to all pairing-based verifiers** — Groth16, PlonK, FFLONK, KZG. Any scheme where the verification equation uses elliptic curve pairings is vulnerable if it doesn't reject identity inputs.
3. **Audit the verifier contract independently from the circuit.** Circuit auditors often skip the Solidity layer. Verifier bugs like this are invisible to circuit analysis tools.
4. **If using a generated verifier** (snarkjs, gnark, etc.), check whether the generator adds identity-point checks. Many don't by default.

---

## Related Posts

- [Missing Bit-Length Checks in Dark Forest and Aztec 2.0](/posts/missing-bit-length-checks-dark-forest-aztec/) — a separate critical flaw in Aztec at the circuit layer, found around the same period
- [Broken Fiat-Shamir in Bulletproofs, PlonK, and Multiple ZK Libraries](/posts/broken-fiat-shamir-bulletproofs-plonk/) — another verifier-level PlonK vulnerability, this time in the Fiat-Shamir transcript
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Primary Disclosure

- [Nguyen Thoi Minh Quan, "The 0 Bug" — full technical paper (PDF)](https://github.com/cryptosubtlety/00/blob/main/00.pdf)
- [cryptosubtlety/00 — GitHub repo with paper and proof-of-concept](https://github.com/cryptosubtlety/00)
- [Aztec Network — Disclosure of Recent Vulnerabilities (HackMD)](https://hackmd.io/@aztec-network/disclosure-of-recent-vulnerabilities)

### Bug Trackers

- [0xPARC ZK Bug Tracker — #7: Aztec Plonk Verifier 0 Bug](https://github.com/0xPARC/zk-bug-tracker#7-aztec-plonk-verifier-0-bug)
- [0xPARC ZK Bug Tracker — #6: Aztec 2.0 Missing Bit Length Check](https://github.com/0xPARC/zk-bug-tracker#6-aztec-20-missing-bit-length-check--nondeterministic-nullifier)

### Aztec Protocol Repos

- [AztecProtocol/barretenberg — C++ Plonk prover/verifier](https://github.com/AztecProtocol/barretenberg)
- [AztecProtocol/aztec-2-bug-bounty](https://github.com/AztecProtocol/aztec-2-bug-bounty)
- [AztecProtocol/aztec-packages — current monorepo](https://github.com/AztecProtocol/aztec-packages)

### PlonK Protocol

- [Gabizon, Williamson, Ciobotaru — "PlonK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge" (IACR ePrint 2019/953)](https://eprint.iacr.org/2019/953)
