---
title: "Polygon zkEVM: Circuit Bugs and STARK/SNARK Field Mismatch"
subtitle: "From Pre-Mainnet Circuit Bugs to a Field Mismatch That Broke the Recursive Prover"
date: 2026-02-22
description: "Multiple critical vulnerabilities in Polygon zkEVM: PIL/zkASM circuit bugs found by Hexens, and a STARK/SNARK field incompatibility that broke recursive proving."
tags: ["zk", "circuits", "polygon"]
---

# Polygon zkEVM Under Scrutiny
### From Pre-Mainnet Circuit Bugs to a Field Mismatch That Broke the Recursive Prover

> **ZK Exploits Series**
>
> **Vulnerability classes:** `CIRCUIT` · `PROOF SYSTEM` — PIL/zkASM bugs; STARK/SNARK field incompatibility
>
> **Protocol:** Polygon zkEVM (ZK rollup, EVM-equivalent)
>
> **Chain:** Ethereum L2
>
> **Years:** February 2023 (audits) · December 2023 (Verichains)

---

Polygon zkEVM launched mainnet beta in March 2023 as the first production ZK rollup claiming full EVM equivalence. Pre-launch audits found 14+ critical bugs. Nine months later, Verichains demonstrated a live exploit on mainnet.

---

## Part 1 — The Pre-Mainnet Audits: Critical Bugs Before Launch

| | |
|---|---|
| **Date** | February 2023 |
| **Chain** | Ethereum L2 (Polygon zkEVM, pre-mainnet) |
| **Auditors** | Hexens (PIL/zkASM) · Spearbit (broader review) |
| **Combined Critical Findings** | 14+ critical severity bugs |
| **Funds Stolen** | None — all fixed before mainnet launch |
| **References** | [Hexens audit report](https://github.com/0xPolygon/zkevm-contracts/blob/main/audits/polygon-zkevm-hexens-audit.pdf) · [Spearbit audit summary](https://polygon.technology/blog/polygon-zkevm-spearbit-audit) |

Polygon zkEVM uses **PIL** (Polynomial Identity Language) and **zkASM** to describe EVM execution circuits. Hexens' audit found four critical vulnerabilities:

**1. Fake SMT Inclusion Proofs** — Under-constrained SMT verification let a prover claim any key-value pair existed in the state tree, including fake balances.

**2. ETH Balance Inflation** — A constraint error let a prover assert that an ETH transfer created more ETH than existed in the source account.

**3. Execution Flow Hijack** — Missing constraints on the program counter let a prover skip or reorder opcodes — e.g., skipping balance checks in ERC-20 transfers.

**4. `ecrecover` Bypass** — Under-constrained ECDSA recovery let an attacker prove any public key signed any message. All transaction authentication broken.

Spearbit's concurrent review found 10 additional critical findings in verifier contracts, bridge logic, and state transition functions.

---

## Part 2 — The Verichains Exploit: Field Mismatch in the Recursive Prover

| | |
|---|---|
| **Date** | December 2023 (disclosed) |
| **Chain** | Polygon zkEVM mainnet (Fork ID 4) |
| **Discovered By** | Verichains Security |
| **Exploit** | Live proof-of-concept demonstrated on mainnet |
| **Funds at Risk** | All funds in Polygon zkEVM bridge |
| **Funds Stolen** | None — responsibly disclosed; patched in Fork ID 5 |
| **References** | [Verichains disclosure](https://www.verichains.io/news/polygon-zkevm-critical-vulnerability-2023) |

### The Architecture

Polygon zkEVM uses a **recursive proving pipeline**:

1. **eSTARK** proves the EVM execution trace in a field called Fp³ — an extension of the 64-bit Goldilocks field, chosen for its fast FFT performance on modern CPUs
2. The eSTARK proof is then **recursively verified** inside a Groth16 SNARK circuit, which operates over the BN254 curve's base field Fq (~254 bits)
3. The Groth16 proof is what actually gets posted on-chain and verified by the Ethereum smart contract

The L2 **state root** from step 1 gets passed into step 2 and committed in the Groth16 proof. The L1 bridge uses this root to validate withdrawals.

### The Vulnerability

eSTARK operates in Fp³ (Goldilocks, ≈ 2^64). Groth16 operates in Fq (BN254, ≈ 2^254). When the state root crosses this boundary without re-reduction, an attacker can find values that produce the same Goldilocks residue but are distinct in BN254 — representing a different state root.

This allowed a malicious operator to:
1. Construct a legitimate-looking EVM execution trace with honest state transitions
2. In the Groth16 verification step, substitute a **different state root** — one corresponding to fake account balances — that produced the same Fp³ residue as the honest root
3. The Groth16 proof would verify (the field mismatch made the check meaningless), and the fraudulent state root would be committed on-chain
4. The attacker could then withdraw funds corresponding to the fake balances, draining the bridge

Verichains demonstrated this exploit on the live Polygon zkEVM mainnet with Fork ID 4, confirming the attack was practical. The vulnerability was patched in Fork ID 5.

---

## Takeaways

The Verichains finding shows that the **interfaces between proving systems** — where outputs from one layer become inputs to another — create attack surfaces invisible to any audit reviewing systems in isolation.

The Spearbit finding (missing remainder bound `r < n`) is structurally identical to the [PSE/Scroll modulo bug](/posts/pse-scroll-zkevm-under-constrained-modulo/) — same bug class, two independent zkEVM implementations, different proving stacks.

---

## How to Catch This

### For Circuit Bugs (Hexens findings)

**Why it happens:** zkEVM circuits are massive — tens of thousands of constraints implementing the full EVM instruction set. PIL/zkASM is a low-level language where constraints are written by hand. Each opcode implementation needs its own constraints, and a single missing constraint in any opcode breaks the entire system. There's no compiler checking that you constrained everything.

**What to check:**

1. **Every zkASM opcode implementation needs independent review.** Treat each opcode as a separate circuit with its own completeness/soundness requirements.
2. **Fuzz the prover.** Generate random EVM execution traces, create witnesses, then try mutating witness values. If a mutated witness still produces a valid proof, you have an under-constrained opcode.
3. **Cross-reference against the EVM spec.** For each opcode, list every property the EVM guarantees (e.g., `ecrecover` returns address(0) on invalid sig) and verify each property is constrained — not just computed.

### For Field Mismatches (Verichains finding)

**Why it happens:** Recursive proving pipelines compose proof systems with different native fields. The Goldilocks field (64-bit) is chosen for STARK prover performance, but the final Groth16 proof operates over BN254 (254-bit). Values crossing this boundary must be explicitly reduced and re-validated in the target field. If the conversion assumes values fit without checking, an attacker can find collisions.

**What to check:**

1. **At every proof system boundary, explicitly re-constrain all transferred values in the target field.** Don't assume that a valid Goldilocks element maps to a unique BN254 element.
2. **Test with values near field boundaries.** `p - 1`, `p`, `p + 1` in the source field should all be tested to verify correct handling in the target field.
3. **Audit the recursive verifier circuit as carefully as the base circuit.** The recursive step is where soundness is most likely to break.

---

## Related Posts

- [PSE/Scroll zkEVM: Under-Constrained Modular Arithmetic in Halo2](/posts/pse-scroll-zkevm-under-constrained-modulo/) — same `r < n` missing bound, different proving stack
- [Under-Constrained Signals in Tornado Cash, Circom-Pairing, and ZK Email](/posts/under-constrained-signals-tornado-circom-zkemail/) — the same under-constrained pattern at the circuit level
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Hexens Audit — PIL/zkASM Circuit Bugs (Feb 2023)

- [0xPARC zk-bug-tracker #21 — Fake SMT Inclusion Proof](https://github.com/0xPARC/zk-bug-tracker#hexens-polygonzkevm-1)
- [0xPARC zk-bug-tracker #22 — ETH Balance Inflation via CTX](https://github.com/0xPARC/zk-bug-tracker#hexens-polygonzkevm-2)
- [0xPARC zk-bug-tracker #23 — Execution Flow Hijack](https://github.com/0xPARC/zk-bug-tracker#hexens-polygonzkevm-3)
- [Fix: SMT constraint (zkevm-proverjs PR #145)](https://github.com/0xPolygonHermez/zkevm-proverjs/pull/145/commits/9d6a8948636c05d508694a90d192a0713562ce29)
- [Fix: CTX assignation (zkevm-rom commit)](https://github.com/0xPolygonHermez/zkevm-rom/commit/2ddeffbed7c022e04032e6d56ed6c6fb14cc38dc)

### Spearbit Audit — Missing Remainder Constraint (Feb 2023)

- [0xPARC zk-bug-tracker #20 — Missing Remainder Constraint](https://github.com/0xPARC/zk-bug-tracker#polygon-zkevm-1)
- [Spearbit Twitter Thread](https://twitter.com/SpearbitDAO/status/1679189382907953180)

### Polygon zkEVM Repos

- [zkevm-proverjs](https://github.com/0xPolygonHermez/zkevm-proverjs)
- [zkevm-rom](https://github.com/0xPolygonHermez/zkevm-rom)
- [zkevm-prover](https://github.com/0xPolygonHermez/zkevm-prover)
- [pilcom](https://github.com/0xPolygonHermez/pilcom)
- [pil-stark](https://github.com/0xPolygonHermez/pil-stark)
- [Polygon zkEVM documentation](https://wiki.polygon.technology/docs/zkEVM/)

### Bug Bounty

- [Polygon zkEVM on Immunefi](https://immunefi.com/bug-bounty/polygonzkevm/)

### General

- [Polygon zkEVM mainnet beta announcement](https://polygon.technology/blog/polygon-zkevm-mainnet-beta-is-live)
- [0xPARC zk-bug-tracker](https://github.com/0xPARC/zk-bug-tracker)
