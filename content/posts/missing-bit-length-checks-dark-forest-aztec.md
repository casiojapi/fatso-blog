---
title: "Missing Bit-Length Checks in Dark Forest and Aztec 2.0"
subtitle: "Missing Bit-Length Checks in Dark Forest and Aztec 2.0"
date: 2026-02-18
description: "Circom's assert() doesn't compile into R1CS constraints. How Dark Forest and Aztec 2.0 both fell for unconstrained input ranges."
tags: ["zk", "circuits", "aztec"]
---

# When assert() Is Not a Constraint
### Missing Bit-Length Checks in Dark Forest and Aztec 2.0

> **ZK Exploits Series**
>
> **Vulnerability class:** `CIRCUIT` — Unconstrained input range; nondeterministic witnesses
>
> **Protocols:** Dark Forest v0.3 (2020–21) · Aztec 2.0 (2021)

---

Circom's standard library (`circomlib`) uses `assert()` statements to document size expectations — but `assert()` is evaluated at build time, not compiled into the R1CS. At proof time, nothing stops a malicious prover from feeding inputs that violate those assumptions.

Both bugs here follow the same pattern: a component assumes its inputs fit within `n` bits, that assumption is never constrained at the call site, and a malicious prover supplies oversized inputs to break the protocol.

---

## Exploit 1 — Dark Forest v0.3: Fake Coordinates via Oversized RangeProof Inputs

| | |
|---|---|
| **Date** | 2020–2021 |
| **Chain** | Ethereum / xDai (Gnosis Chain) |
| **Protocol** | Dark Forest (fully on-chain ZK strategy game) |
| **Impact** | Any player could teleport to any coordinate in the universe |
| **Funds at Risk** | N/A — game state, not financial protocol |
| **Discovered By** | Daira Hopwood (Electric Coin Company) |

Dark Forest is an on-chain strategy game where ZK proofs hide player positions. Moving requires a proof that the destination is within the ship's range. The circuit used a `RangeProof` template:

```circom
template RangeProof(bits, max_abs_value) {
    signal input in;

    component lowerBound = LessThan(bits);
    component upperBound = LessThan(bits);

    lowerBound.in[0] <== max_abs_value + in;
    lowerBound.in[1] <== 0;
    lowerBound.out === 0;

    upperBound.in[0] <== 2 * max_abs_value;
    upperBound.in[1] <== max_abs_value + in;
    upperBound.out === 0;
}
```

This looks correct. The logic enforces that `in` is within `[-max_abs_value, +max_abs_value]`. But look at how `LessThan(bits)` works internally:

```circom
template LessThan(n) {
    assert(n <= 252);      // ← JS assertion, NOT an R1CS constraint
    signal input in[2];
    signal output out;

    component n2b = Num2Bits(n+1);
    n2b.in <== in[0] + (1<<n) - in[1];
    out <== 1 - n2b.out[n];
}
```

`LessThan(bits)` works by computing `in[0] + (1 << bits) - in[1]` and decomposing the result into `bits+1` bits. This only gives the correct comparison result if **both inputs fit within `bits` bits**. That assumption is documented only as an `assert()` — invisible at proof time.

In `RangeProof`, neither `in` nor `max_abs_value` is constrained to fit within `bits` bits. A malicious player supplies an oversized coordinate, `Num2Bits(n+1)` wraps incorrectly, `LessThan` outputs the wrong result, and the player teleports anywhere.

Fix: add `Num2Bits` constraints on both inputs *before* passing them to `LessThan`.

**References:** [0xPARC Bug Tracker #1](https://github.com/0xPARC/zk-bug-tracker#1-dark-forest-v03-missing-bit-length-check) · [Dark Forest circuit explanation](https://blog.zkga.me/df-init-circuit)

---

## Exploit 2 — Aztec 2.0: Note Index Without Bit Constraint Enables Infinite Double-Spend

| | |
|---|---|
| **Date** | 2021 |
| **Chain** | Ethereum |
| **Protocol** | Aztec 2.0 (ZK privacy L2) |
| **Funds at Risk** | All TVL in Aztec 2.0 — any note could be spent indefinitely |
| **Outcome** | Found internally by Aztec team; no funds lost |

Aztec represents funds as **note commitments** in an on-chain Merkle tree. Spending a note requires posting a **nullifier** — a unique value derived from the note to prevent double-spending. The same note must always produce the same nullifier.

The nullifier was computed as a hash of several values, including the **note's index in the Merkle tree** — assumed to be 32-bit. But **no R1CS constraint enforced this**.

An attacker who spent a note with index `i` could compute new witnesses using `i + 2^32`, `i + 2 * 2^32`, etc. Each has the same lower 32 bits (passes the Merkle inclusion proof) but produces a **distinct nullifier**. The same note can be spent unlimited times.

```
Note index:  i              →  nullifier_1  (already posted, rejected)
             i + 2^32       →  nullifier_2  (fresh, accepted) ← double spend
             i + 2 * 2^32   →  nullifier_3  (fresh, accepted) ← triple spend
             ...
```

Fix: a single `Num2Bits(32)` constraint on the note index.

**References:** [0xPARC Bug Tracker #6](https://github.com/0xPARC/zk-bug-tracker#6-aztec-20-missing-bit-length-check--nondeterministic-nullifier)

---

## The Pattern

| | Dark Forest (2020) | Aztec 2.0 (2021) |
|---|---|---|
| **Unconstrained value** | Coordinate range (input to `LessThan`) | Note index width (input to hash) |
| **Missing check** | `Num2Bits(bits)` on both inputs to `LessThan` | `Num2Bits(32)` on note index |
| **Security broken** | Movement range enforcement | Nullifier uniqueness (no double-spend) |
| **Impact** | Teleport anywhere in the game | Spend same note unlimited times |

Both circuits ran correctly in honest test cases. The vulnerability only appears when a malicious prover deliberately supplies out-of-range inputs.

---

## How to Catch This

**Why it happens:** Circomlib's `LessThan`, `GreaterThan`, and similar templates use `assert(n <= 252)` to document input size assumptions. This `assert()` runs only during compilation/witness generation — it is **not** an R1CS constraint. There is no on-chain enforcement. Developers see the assert and assume the check is happening at proof time. It isn't.

**What to check:**

1. **Every input to a comparison template must be explicitly bit-constrained at the call site.** Before passing a signal to `LessThan(n)`, constrain it with `component check = Num2Bits(n); check.in <== yourSignal;`. This forces the input into `n` bits inside the R1CS.
2. **Treat circomlib templates as unchecked building blocks.** Their `assert()` statements are documentation, not security. The caller is responsible for enforcing all input assumptions.
3. **Hash inputs that feed into nullifier computation must be deterministic.** If a nullifier depends on a value that can take multiple valid forms (e.g., an unbounded integer that wraps modulo the field), the same note can produce multiple distinct nullifiers — enabling double-spends. Bit-constrain all hash inputs.
4. **Circomspect flags some of these**, but not all. Manual review of every `LessThan`/`GreaterThan`/`IsZero` call site is necessary.

---

## Related Posts

- [Under-Constrained Signals in Tornado Cash, Circom-Pairing, and ZK Email](/posts/under-constrained-signals-tornado-circom-zkemail/) — the same under-constrained pattern across three protocols
- [Aztec Plonk Verifier Bug: Forging Proofs via the Identity Point](/posts/aztec-plonk-verifier-identity-point/) — a separate critical flaw in Aztec at the verifier layer
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Dark Forest v0.3 — Missing Bit-Length Check

- [0xPARC ZK Bug Tracker — Bug #1: Dark Forest v0.3](https://github.com/0xPARC/zk-bug-tracker#1-dark-forest-v03-missing-bit-length-check)
- [Dark Forest: Initializing the Circuit](https://blog.zkga.me/df-init-circuit) — official circuit walkthrough
- [Dark Forest v0.3 — Circuit Source Code](https://github.com/darkforest-eth/circuits)
- [circomlib `LessThan` template](https://github.com/iden3/circomlib/blob/master/circuits/comparators.circom)
- [Dark Forest Project](https://zkga.me/)

### Aztec 2.0 — Nondeterministic Nullifier

- [0xPARC ZK Bug Tracker — Bug #6: Aztec 2.0 Missing Bit-Length Check](https://github.com/0xPARC/zk-bug-tracker#6-aztec-20-missing-bit-length-check--nondeterministic-nullifier)
- [Aztec Protocol — GitHub](https://github.com/AztecProtocol)
- [Aztec 2.0 Bug Bounty Repository](https://github.com/AztecProtocol/aztec-2-bug-bounty)

### General

- [0xPARC ZK Bug Tracker](https://github.com/0xPARC/zk-bug-tracker)
- [Circom Documentation — `assert()` Semantics](https://docs.circom.io/circom-language/code-quality/assert/)
- [circomlib — iden3](https://github.com/iden3/circomlib)
- [0xPARC Applied ZK Learning Resources](https://learn.0xparc.org/)
