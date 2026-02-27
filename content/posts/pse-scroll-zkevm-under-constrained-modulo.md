---
title: "PSE/Scroll zkEVM: Under-Constrained Modular Arithmetic in Halo2"
subtitle: "Under-Constrained Arithmetic in the PSE & Scroll Halo2 zkEVM"
date: 2026-02-23
description: "An under-constrained modular arithmetic gadget in the PSE/Scroll Halo2 zkEVM accepted false results as valid proofs."
tags: ["zk", "circuits", "scroll"]
---

# A False Modulo Operation That the zkEVM Accepted as True
### Under-Constrained Arithmetic in the PSE & Scroll Halo2 zkEVM

> **ZK Exploits Series**
>
> **Vulnerability class:** `CIRCUIT` — Under-constrained arithmetic gadget in Halo2
>
> **Protocols:** PSE zkEVM · Scroll zkEVM (shared codebase)
>
> **Chain:** Ethereum L2
>
> **Year:** 2022 (pre-deployment audit)

---

| | |
|---|---|
| **Date** | 2022 (found in audit, pre-deployment) |
| **Chain** | Ethereum L2 |
| **Protocols** | PSE (Privacy & Scaling Explorations) zkEVM · Scroll zkEVM |
| **Funds at Risk** | All funds on both rollups if deployed |
| **Outcome** | Found pre-deployment; patched before mainnet |
| **References** | [0xPARC Bug Tracker #23](https://github.com/0xPARC/zk-bug-tracker#23-pse--scroll-zkevms-under-constrained-modmul) |

---

## Background

PSE and Scroll both built ZK rollups using **Halo2** and shared large portions of their zkEVM circuit code, including arithmetic gadgets. The EVM uses 256-bit integer arithmetic emulated over the BN254 scalar field (~254 bits). The `MOD` opcode is frequent and non-trivial to implement correctly in a ZK circuit.

---

## The Vulnerability

The modulo gadget was implemented using a standard technique: to prove `a mod n = r`, it's sufficient to prove the existence of a quotient `q` such that `a = q * n + r` and `0 ≤ r < n`. This avoids explicitly performing division in the circuit (which is expensive) and instead proves the result by exhibiting the quotient as a witness.

The bug: the constraint `0 ≤ r < n` (that the remainder is actually smaller than the modulus) was **not fully enforced**. The circuit checked that `a = q * n + r` held over the BN254 field, but the bound check on `r` was under-constrained — it was either missing or only partially implemented.

This meant a prover could supply a fake remainder `r'` satisfying `a = q * n + r'` over the BN254 field, even if `r' ≥ n` (which would make it an invalid remainder in standard integer arithmetic). For example:

```
Honest:    10 mod 3 = 1   (correct: 10 = 3*3 + 1, and 1 < 3)
Malicious: 10 mod 3 = 99  (accepted: 10 = 3*(-29) + 99 mod field, and the r < n check is missing)
```

Since `MOD` output flows into subsequent EVM operations, a prover who can fabricate modular arithmetic results can make the zkEVM accept execution traces with arbitrary incorrect state transitions — fabricated withdrawal amounts, bypassed access control, corrupted state.

---

## The Fix

Strengthen the range constraint on `r` to enforce it at the R1CS / Plonkish constraint level, not just in witness generation. Specifically, decompose `r` into bits and verify `r < n` using a proper comparison gadget — the same issue raised in the Dark Forest and Aztec bugs, now manifesting in a Halo2 context rather than Circom.

---

## Note on Scope

This bug affects a shared gadget library — a single under-constrained gadget simultaneously compromised two independent rollups. Shared zkEVM infrastructure increases both efficiency and systemic risk.

---

## How to Catch This

**Why it happens:** Modular arithmetic in ZK circuits is always decomposed into a witness-based proof: instead of computing `a mod n` directly (which is expensive in R1CS/Plonkish), the prover supplies `q` and `r` as witnesses and the circuit checks `a = q * n + r`. This is correct only if `r < n` is also enforced. Developers often implement the multiplication check and forget the range check — or implement it only in witness generation (where it's not binding).

**What to check:**

1. **Every modular arithmetic gadget must enforce `r < n` as a constraint, not just in witness generation.** Decompose `r` into bits and use a comparison gadget, or use a lookup table for range checks.
2. **The same pattern applies to division, remainder, and any operation where the prover supplies auxiliary values.** If the prover provides `q` and `r` such that `a = q * n + r`, you need: (a) the multiplication constraint, (b) `0 <= r < n`, and (c) `q >= 0`.
3. **In Halo2, use `range_check` or `lookup` gates for bounds.** Unlike Circom where `Num2Bits` is the standard tool, Halo2 has native lookup gates that are more efficient for range constraints.
4. **When using shared gadget libraries (like the PSE/Scroll shared codebase), audit upstream.** A bug in a shared gadget propagates to every circuit that imports it. Pin versions and review diffs on upgrade.

---

## Related Posts

- [Polygon zkEVM: Circuit Bugs and STARK/SNARK Field Mismatch](/posts/polygon-zkevm-circuit-bugs-field-mismatch/) — the same `r < n` missing bound was independently found in Polygon zkEVM by Spearbit
- [Under-Constrained Signals in Tornado Cash, Circom-Pairing, and ZK Email](/posts/under-constrained-signals-tornado-circom-zkemail/) — the broader pattern of under-constrained circuits
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

- [0xPARC ZK Bug Tracker](https://github.com/0xPARC/zk-bug-tracker)
- [PSE zkevm-circuits](https://github.com/privacy-scaling-explorations/zkevm-circuits)
- [Scroll zkevm-circuits](https://github.com/scroll-tech/zkevm-circuits)
- [Halo2 documentation](https://zcash.github.io/halo2/)
