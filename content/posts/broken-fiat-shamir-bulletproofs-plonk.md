---
title: "Broken Fiat-Shamir in Bulletproofs, PlonK, and Multiple ZK Libraries"
subtitle: "How Incomplete Fiat-Shamir Transcripts Broke Multiple ZK Systems at Once"
date: 2026-02-21
description: "Trail of Bits found the same Fiat-Shamir implementation flaw across Bulletproofs, PlonK, Girault, and multiple ZK libraries simultaneously."
tags: ["zk", "proof-systems", "security"]
---

# Frozen Heart
### How Incomplete Fiat-Shamir Transcripts Broke Multiple ZK Systems at Once

> **ZK Exploits Series**
>
> **Vulnerability class:** `PROOF SYSTEM` — Insecure Fiat-Shamir transformation; missing values in hash transcript
>
> **Protocols affected:** ING Bank zkrp · SECBIT Labs · Adjoint · ZenGo · Dusk Network · Iden3 SnarkJS · ConsenSys gnark (and more)
>
> **Year:** April 2022 (coordinated disclosure by Trail of Bits)

---

| | |
|---|---|
| **Date** | April 2022 (public disclosure) |
| **Discovered By** | Jim Miller, Trail of Bits |
| **Affected Systems** | 7+ ZK implementations across Bulletproofs, PlonK, Σ-protocols |
| **Funds at Risk** | Varies per implementation; forged proofs possible in all affected systems |
| **Funds Stolen** | None — coordinated disclosure, all patched |
| **References** | [Trail of Bits: "Frozen Heart" (Apr 2022)](https://blog.trailofbits.com/2022/04/15/the-frozen-heart-vulnerability-in-plonk/) · [Follow-up post on Bulletproofs](https://blog.trailofbits.com/2022/04/15/the-frozen-heart-vulnerability-in-bulletproofs/) · [CVE-2022-29566](https://nvd.nist.gov/vuln/detail/CVE-2022-29566) |

---

## Background: The Fiat-Shamir Transformation

Interactive ZK proofs work because the verifier sends random challenges the prover can't predict. Blockchain protocols need non-interactive proofs, so the **Fiat-Shamir transformation** replaces verifier challenges with the output of a hash over the transcript so far.

The critical requirement: the hash must include **all prover messages before the challenge point** and all **public inputs**. If any value is omitted, the challenge becomes predictable. An attacker can find inputs producing a favorable challenge — breaking soundness.

---

## The Vulnerability

Jim Miller (Trail of Bits) discovered that multiple independent ZK implementations omitted values from their Fiat-Shamir transcripts. The missing value varied by system:

**Bulletproofs** (ING Bank zkrp, SECBIT Labs, Adjoint, ZenGo): The Pedersen commitment `V` was not in the transcript. A malicious prover can manipulate `V` after fixing the challenge, "proving" that any value satisfies the range constraint.

```
Correct transcript:  H(G, H, V, A, S)  →  challenge y, z
Vulnerable:          H(G, H, A, S)     →  challenge y, z
                          ↑
                     V omitted — prover can substitute any V after challenge is fixed
```

**PlonK** (Iden3 SnarkJS, Dusk Network, ConsenSys gnark): **Public inputs** were not in the first challenge hash. An attacker could construct a valid proof then substitute different public inputs — proving a 1-token transfer then claiming it was 1,000,000.

**Σ-protocols** (Schnorr-based): The public key or commitment was missing from the transcript, enabling classic forgery.

---

## Why Seven Systems at Once

The requirement is documented in every security proof. Implementations drifted for mundane reasons:

- Reference implementations in papers omitted the value; downstream copied it
- Developers added public inputs without updating transcript hashing
- Honest tests never reveal the bug

Miller found the first instance during a routine audit. A targeted sweep revealed it was widespread. All seven teams were notified and disclosed simultaneously in April 2022.

---

## Affected Systems

| Implementation | Missing Value | Proof Type |
|---|---|---|
| ING Bank zkrp | `V` (committed value) | Bulletproofs range proof |
| SECBIT Labs | `V` | Bulletproofs range proof |
| Adjoint Inc. | `V` | Bulletproofs range proof |
| ZenGo (threshold sigs) | `V` | Bulletproofs |
| Dusk Network | Public inputs | PlonK |
| Iden3 SnarkJS | Public inputs | PlonK |
| ConsenSys gnark | Public inputs | PlonK |

---

## Impact

In affected Bulletproofs libraries: prove a secret balance is non-negative when it's massively negative. In affected PlonK libraries: generate a proof for one statement, substitute different public inputs.

The defense is procedural: every public input and every prover message must be hashed. Trail of Bits open-sourced test vectors specifically designed to catch Fiat-Shamir transcript errors.

---

## How to Catch This

**Why it happens:** Developers implement the Fiat-Shamir transcript by reading the security proof and listing what goes into the hash. They miss a value — sometimes because the paper itself omits it (the original Bulletproofs paper had this bug), sometimes because a new public input was added after the transcript code was written. The resulting system passes every honest test because honest provers don't try to manipulate challenges.

**What to check:**

1. **The transcript must include: all public inputs, all verifier parameters (generators, group elements), and every prover message sent before the challenge point.** If any of these are missing, the challenge is manipulable. Write down the full list and cross-check against your hash calls.
2. **When you add a new public input to a protocol, update the transcript.** This is the most common way the bug gets re-introduced — the protocol evolves but the Fiat-Shamir hash doesn't.
3. **Use a structured transcript API** (like Merlin in Rust) instead of manually calling `keccak256(abi.encodePacked(...))`. Structured transcript libraries make omissions harder because they force you to explicitly commit each value.
4. **Test with Trail of Bits' adversarial vectors.** They released test inputs specifically designed to detect Frozen Heart variants. If your implementation accepts them, you have a transcript bug.
5. **Review the paper your implementation follows.** If the paper's Fiat-Shamir specification is wrong, your correct implementation of that spec is still vulnerable.

---

## Related Posts

- [Aztec Plonk Verifier Bug: Forging Proofs via the Identity Point](/posts/aztec-plonk-verifier-identity-point/) — another critical PlonK vulnerability, this time in the verifier's elliptic curve pairing check
- [Under-Constrained Signals in Tornado Cash, Circom-Pairing, and ZK Email](/posts/under-constrained-signals-tornado-circom-zkemail/) — SnarkJS (Iden3), also affected by Frozen Heart, had separate under-constrained circuit vulnerabilities
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Trail of Bits — Frozen Heart Disclosure (April 2022)

- [Part 1: Coordinated Disclosure of Vulnerabilities Affecting Girault, Bulletproofs, and PlonK](https://blog.trailofbits.com/2022/04/13/part-1-coordinated-disclosure-of-vulnerabilities-affecting-girault-bulletproofs-and-plonk/)
- [The Frozen Heart Vulnerability in Bulletproofs](https://blog.trailofbits.com/2022/04/15/the-frozen-heart-vulnerability-in-bulletproofs/)
- [The Frozen Heart Vulnerability in PlonK](https://blog.trailofbits.com/2022/04/15/the-frozen-heart-vulnerability-in-plonk/)

### CVE

- [CVE-2022-29566 — Dusk Network `dusk-plonk`: Frozen Heart vulnerability](https://nvd.nist.gov/vuln/detail/CVE-2022-29566)

### Affected Library Repositories

- [Adjoint Inc. bulletproofs](https://github.com/adjoint-io/bulletproofs)
- [ZenGo-X bulletproofs](https://github.com/ZenGo-X/bulletproofs)
- [Dusk Network plonk](https://github.com/dusk-network/plonk)
- [Iden3 snarkjs](https://github.com/iden3/snarkjs)
- [ConsenSys gnark](https://github.com/ConsenSys/gnark)

### Academic Papers

- Fiat, A. & Shamir, A. (1986). [How to Prove Yourself: Practical Solutions to Identification and Signature Problems](https://link.springer.com/chapter/10.1007/3-540-47721-7_12)
- Bunz, B. et al. (2018). [Bulletproofs: Short Proofs for Confidential Transactions and More](https://eprint.iacr.org/2017/1066)
- Gabizon, A., Williamson, Z., & Ciobotaru, O. (2019). [PlonK](https://eprint.iacr.org/2019/953)
- Dao, Q., Miller, J., Wright, O., & Grubbs, P. (2023). [Weak Fiat-Shamir Attacks on Modern Proof Systems](https://eprint.iacr.org/2023/691)

### Community

- [0xPARC ZK Bug Tracker](https://github.com/0xPARC/zk-bug-tracker)
