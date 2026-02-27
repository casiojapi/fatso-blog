---
title: "Under-Constrained Signals in Tornado Cash, Circom-Pairing, and ZK Email"
subtitle: "The Most Common ZK Bug -- Found Three Times Across Four Years"
date: 2026-02-17
description: "How under-constrained signals became the most recurring bug class in ZK circuits, hitting Tornado Cash, Circom-Pairing, and ZK Email across four years."
tags: ["zk", "circuits", "tornado-cash"]
---

# Assigned But Not Constrained
### The Most Common ZK Bug — Found Three Times Across Four Years

> **ZK Exploits Series**
>
> **Vulnerability class:** `CIRCUIT` — Under-constrained signals
>
> **Protocols:** Tornado Cash (2019) · Circom-Pairing / Succinct Labs (2022) · ZK Email (2023)

---

In Circom, two operators look nearly identical but behave completely differently:

```circom
signal <== expression;   // assigns AND adds an R1CS constraint  ✅
signal <-- expression;   // assigns only — NO constraint added   ❌
```

Assignment happens during witness generation — offline, by the prover. Constraints compile into the R1CS and get checked on-chain. A signal that is assigned but never constrained can hold any value and the proof will still verify.

In older Circom versions, `=` behaved like `<--`. This single character difference is the root cause of the most famous ZK circuit exploit. The same class of error was independently found in three protocols across four years.

---

## Exploit 1 — Tornado Cash: MiMC Hash Output Not Constrained

| | |
|---|---|
| **Date** | October 12, 2019 |
| **Chain** | Ethereum |
| **Funds at Risk** | All ETH in the mixer |
| **Outcome** | Team self-exploited (white hat), migrated all funds |
| **Discovered By** | Kobi Gurkan |

Tornado Cash is a privacy mixer. Users deposit ETH, receive a secret note, then later submit a ZK proof they know a note corresponding to some past deposit — without revealing which one. The proof operates over a Merkle tree built using the **MiMC hash function** in Circom.

In circomlib's MiMC implementation, the output signal was assigned using `=` (equivalent to `<--`) instead of `<==`:

```circom
// VULNERABLE
template MiMC7(nrounds) {
    signal input x_in;
    signal input k;
    signal output out;
    // ...
    out = someExpression;  // assignment only — NOT in the R1CS
}
```

Since `out` was never constrained, an attacker could construct a witness with an arbitrary fake Merkle root, generate a valid Groth16 proof, and withdraw ETH without ever depositing.

The team discovered this before any malicious actor did. The contract was immutable, so they self-exploited: drained all ETH and migrated to a patched version. The fix was two characters: `=` → `<==`.

**References:** [Tornado Cash disclosure](https://tornado-cash.medium.com/tornado-cash-got-hacked-by-us-b1e012a3c9a8) · [0xPARC Bug Tracker #14](https://github.com/0xPARC/zk-bug-tracker#14-mimc-hash-assigned-but-not-constrained)

---

## Exploit 2 — Circom-Pairing: BigLessThan Output Never Checked

| | |
|---|---|
| **Date** | 2022 (pre-deployment audit) |
| **Chain** | Ethereum bridge |
| **Funds at Risk** | All bridge TVL if deployed |
| **Outcome** | Caught by Veridise auditors before launch |
| **Discovered By** | Veridise Team |

Circom-Pairing implements BLS12-381 pairing verification in Circom. Succinct Labs used it for a trustless cross-chain light client bridge.

BLS12-381 field elements are 381 bits, larger than Circom's native field. All inputs must be less than the BLS12-381 prime `q`. The circuit instantiated ten `BigLessThan` components to enforce this — but their output signals were **never constrained to equal 1**:

```circom
// VULNERABLE
component lt[10];
for (var i = 0; i < 10; i++) {
    lt[i] = BigLessThan(n, k);
    lt[i].b[idx] <== q[idx];  // inputs wired
}
// lt[i].out never constrained — comparisons run but verifier ignores results
```

An attacker could supply public keys with values `≥ q`. The comparisons would output `0` (failed), but since that output was unconstrained, the proof still verified. Out-of-range inputs break BLS12-381 pairing assumptions, allowing a malicious bridge relay to forge consensus proofs for fake blocks and steal all locked funds.

**References:** [Veridise write-up](https://medium.com/veridise/circom-pairing-a-million-dollar-zk-bug-caught-early-c5624b278f25) · [0xPARC Bug Tracker #3](https://github.com/0xPARC/zk-bug-tracker#3-circom-pairing-missing-output-check-constraint)

---

## Exploit 3 — ZK Email: Under-Constrained Header Parsing Enables Address Spoofing

| | |
|---|---|
| **Date** | 2023 (audit, pre-deployment) |
| **Chain** | Ethereum (identity applications) |
| **Funds at Risk** | Any application using ZK Email for auth or identity |
| **Outcome** | Found in security audit before deployment |

ZK Email lets users prove ownership of an email address by generating a ZK proof over a DKIM-signed email — without revealing the email's contents.

The circuit proved that a specific address appeared in the `From:` header of a DKIM-validated email. A missing constraint in the header parsing logic left the `From:` extraction **under-determined** — multiple witnesses could satisfy the same constraint system.

An attacker with any valid DKIM-signed email from `attacker@attacker.com` could manipulate their witness to claim the `From:` field showed `victim@victim.com`. Any application relying on ZK Email for authentication could be fully bypassed.

**References:** [0xPARC Bug Tracker #27](https://github.com/0xPARC/zk-bug-tracker#27-zk-email-under-constrained-circuit-leads-to-email-address-spoofing)

---

## Three Bugs, One Root Cause

| | Tornado Cash (2019) | Circom-Pairing (2022) | ZK Email (2023) |
|---|---|---|---|
| **What was unconstrained** | MiMC hash output | `BigLessThan` output signals | `From:` header extraction |
| **What the attacker faked** | Merkle root (fake deposit) | BLS12-381 pubkey validity | Email address identity |
| **Impact** | Withdraw without depositing | Forge bridge consensus proofs | Impersonate any email address |
| **Exploited?** | Yes (by the team, white hat) | No — caught in audit | No — caught in audit |

Computing a value and *binding the verifier to that value* are separate acts in ZK circuits — forgetting the second is syntactically valid and completely silent. The circuit runs correctly in every honest test case. The failure only surfaces when a malicious prover deliberately exploits the gap. [Circomspect](https://github.com/trailofbits/circomspect) was built specifically to catch this class of bug.

---

## How to Catch This

**Why it happens:** In Circom, `<--` and `=` compile without warnings. The witness generates correctly. All honest tests pass. The bug is invisible unless you have adversarial test cases — and most projects don't.

**What to check:**

1. **Every signal that influences a public output or constraint must use `<==`**, not `<--` or `=`. Grep your circuits for `<--` — each one is a potential vulnerability unless a separate constraint binds it.
2. **Subcircuit outputs are not automatically constrained.** When you instantiate a template, its `.out` signal is just a wire. If your circuit doesn't constrain that wire (either via `<==` or an explicit `=== 1` check), the verifier ignores it entirely. This is the Circom-Pairing bug.
3. **Run [Circomspect](https://github.com/trailofbits/circomspect)** — Trail of Bits' static analyzer flags unconstrained signals, unused outputs, and `<--` assignments that lack corresponding constraints.
4. **Write adversarial witness tests.** Feed your circuit a witness where every unconstrained signal is set to a random value. If the proof still verifies, you have an under-constrained signal.

---

## Related Posts

- [Missing Bit-Length Checks in Dark Forest & Aztec](/posts/missing-bit-length-checks-dark-forest-aztec/) — another under-constrained circuit pattern
- [PSE/Scroll zkEVM: Under-Constrained Modular Arithmetic](/posts/pse-scroll-zkevm-under-constrained-modulo/) — same vulnerability class in zkEVM circuits
- [Tornado Cash Governance Takeover & Malicious Frontend](/posts/tornado-cash-governance-takeover-malicious-frontend/) — different vulnerability class, same protocol
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Tornado Cash — MiMC Hash (2019)

- [Tornado Cash disclosure: "Tornado.cash got hacked. By us."](https://tornado-cash.medium.com/tornado-cash-got-hacked-by-us-b1e012a3c9a8) — team blog post, October 12, 2019
- [circomlib PR #22: "mimcsponge: fixes assignment to outs[0]"](https://github.com/iden3/circomlib/pull/22) — fix by Kobi Gurkan, merged September 17, 2019
- [Fix commit `109cdf40`](https://github.com/iden3/circomlib/commit/109cdf40567fce284dca1d535819ce28922653e0) — one-line change: `=` to `<==` in `mimcsponge.circom`
- [0xPARC ZK Bug Tracker #14: MiMC Hash Assigned but not Constrained](https://github.com/0xPARC/zk-bug-tracker#14-mimc-hash-assigned-but-not-constrained)
- [Tornado Cash circuit audit — ABDK Consulting](https://tornado.cash/audits/TornadoCash_circuit_audit_ABDK.pdf)

### Circom-Pairing / Succinct Labs Bridge (2022)

- [Veridise: "Circom-pairing: a million-dollar ZK bug caught early"](https://medium.com/veridise/circom-pairing-a-million-dollar-zk-bug-caught-early-c5624b278f25)
- [circom-pairing PR #21: "CoreVerifyPubkeyG1 does not enforce lt checks on input"](https://github.com/yi-sun/circom-pairing/pull/21) — fix merged November 26, 2022
- [Fix commit `c686f001`](https://github.com/yi-sun/circom-pairing/pull/21/commits/c686f0011f8d18e0c11bd87e0a109e9478eb9e61)
- [0xPARC ZK Bug Tracker #3: Circom-Pairing Missing Output Check Constraint](https://github.com/0xPARC/zk-bug-tracker#3-circom-pairing-missing-output-check-constraint)
- [circom-pairing repository (Yi Sun)](https://github.com/yi-sun/circom-pairing)

### ZK Email — Under-Constrained Regex Circuit (2024)

- [Matter Labs Audits: "ZK Email: Unveiling Classic Attacks"](https://github.com/matter-labs-audits/reports/blob/main/research/zkemail/README.md)
- [ZK Email Security Review Report (PDF)](https://github.com/matter-labs-audits/reports/blob/main/reports/zkemail/ZKEmail%20Security%20Review%20Report.pdf)
- [zk-regex fix commit `7754156`](https://github.com/zkemail/zk-regex/commit/77541563c36075a0a5e817656d4613b5fb7ff548)
- [0xPARC ZK Bug Tracker #27: ZK Email Under-Constrained Circuit](https://github.com/0xPARC/zk-bug-tracker#27-zk-email-under-constrained-circuit-leads-to-email-address-spoofing)
- [ZK Regex technical explainer](https://prove.email/blog/zkregex)

### General

- [0xPARC ZK Bug Tracker](https://github.com/0xPARC/zk-bug-tracker)
- [Circomspect — Trail of Bits static analyzer for Circom](https://github.com/trailofbits/circomspect)
