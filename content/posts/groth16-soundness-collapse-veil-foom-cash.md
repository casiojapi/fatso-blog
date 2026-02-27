---
title: "Groth16 Soundness Collapse: Veil Cash and FOOM Cash gamma==delta Exploit"
subtitle: "The Veil Cash and FOOM Cash Exploits -- A New Class of ZK Deployment Failure"
date: 2026-02-26
description: "Deploying a Groth16 verifier with gamma equal to delta collapses soundness entirely. Two protocols on Base and Ethereum got drained within days."
tags: ["zk", "trusted-setup", "groth16", "exploits"]
---

# When γ Equals δ, Groth16 Has No Soundness
### The Veil Cash and FOOM Cash Exploits — A New Class of ZK Deployment Failure

> **ZK Exploits Series** *(addendum — February 2026)*
>
> **Vulnerability class:** `TRUSTED SETUP` / `DEPLOYMENT` — Groth16 verification key misconfiguration; gamma == delta collapses soundness entirely
>
> **Protocols:** Veil Cash (Feb 21, 2026) · FOOM Cash (Feb 26, 2026)
>
> **Chains:** Base · Ethereum + Base

---

| | Veil Cash | FOOM Cash |
|---|---|---|
| **Date** | ~February 21, 2026 | February 26, 2026 |
| **Chain** | Base | Ethereum + Base |
| **Root cause** | `gamma2 == delta2` in deployed Groth16 verifier | Same (copycat) |
| **Funds stolen** | ~4.5 ETH (~2.9 ETH after whitehat recovery) | ~$2.26M ($1.3M ETH chain, $316K Base) |
| **Funds recovered** | ~2 ETH by whitehat (Decurity) | Partial (~2 ETH by @DefimonAlerts on ETH) |
| **Discovered By** | Identified post-exploit | BlockSec Phalcon (real-time monitoring), CertiK |
| **References** | Protos reporting · BlockSec post-mortem | [CertiK alert (Feb 26)](https://twitter.com/CertiKAlert) · [BlockSec alert](https://twitter.com/Phalcon_xyz) · [QuillAudits thread](https://twitter.com/QuillAudits_AI) · ETH verifier: [0xc043...71A6](https://etherscan.io/address/0xc043865fb4D542E2bc5ed5Ed9A2F0939965671A6#code) · Base verifier: [0x02c3...68C2](https://basescan.org/address/0x02c30D32A92a3C338bc43b78933D293dED4f68C6) |

---

## Was This Known Before?

**No prior exploit.** The Groth16 paper (2016) defines γ and δ as separate toxic waste values — unambiguous in the spec. But no CVE, no advisory, and no write-up warned about this deployment error before Veil Cash was exploited on February 21, 2026.

The closest prior research is Groth16 *proof malleability* (SlowMist 2023, Shchybovyk 2023) — a different attack class that derives new valid proofs from existing ones. The gamma==delta attack doesn't need a valid starting proof. It's a **complete soundness break** from scratch using only public information.

---

## Background: What γ and δ Do in Groth16

The Groth16 verification equation is:

```
e(A, B) = e(α, β) · e(vk_x, γ) · e(C, δ)
```

Where:
- `(A, B, C)` — the three elliptic curve points making up the proof, supplied by the prover
- `α, β` — fixed constants in the verifier key, outputs of the trusted setup
- `vk_x` — the **linear combination of public inputs**: `Σ aᵢ · ICᵢ`, computed entirely from publicly known values
- `γ` (gamma) — a fixed G2 point in the verifier key; its role is to bind the `vk_x` term
- `δ` (delta) — a different fixed G2 point; its role is to bind the `C` term (which contains the private witness)

γ and δ **must be different** — they separate the public-input side (`vk_x`, scaled by γ) from the private-witness side (`C`, scaled by δ). Since both are toxic waste destroyed after the ceremony, an attacker cannot compute their discrete logs and therefore cannot freely construct a valid `C`.

---

## The Attack: γ = δ Collapses the Equation

When the verifier is deployed with `gamma == delta` (i.e., the same G2 point used for both), the verification equation collapses:

```
e(A, B) = e(α, β) · e(vk_x, γ) · e(C, γ)      ← both now use γ
         = e(α, β) · e(vk_x + C, γ)             ← bilinearity: e(P,Q)·e(R,Q) = e(P+R,Q)
```

The equation now only checks that `vk_x + C` equals a specific value — it no longer separately checks that `C` was derived from a legitimate private witness. And since `vk_x` is computed entirely from **public inputs** that the attacker can freely read, the attacker can compute the required `C` directly:

```
C = target - vk_x
```

Where `target` is derivable from `α`, `β`, `γ` — all public constants in the verifier. No private witness needed. Just elliptic curve arithmetic on publicly visible values.

---

## The Attack Execution

CertiK's on-chain analysis of the FOOM Cash exploit confirms the attack sequence:

1. **Read the verifier constants** — `alpha`, `beta`, `gamma`/`delta` (same point), and the IC array — all hardcoded in the Solidity verifier
2. **Choose target public inputs** — specifically a fresh `nullifierHash` value not yet posted on-chain
3. **Compute `vk_x`** — the linear combination `Σ aᵢ · ICᵢ` for the chosen public inputs; this is straightforward elliptic curve arithmetic
4. **Compute `C = -vk_x`** — elliptic curve negation (one operation)
5. **Set `A = [alpha]G1` and `B = [beta]G2`** — constants from the verifier; this makes `e(A,B) = e(alpha, beta)`
6. **Submit the proof** — the verifier equation evaluates to `e(alpha,beta) = e(alpha,beta) · e(0, γ)` = `e(alpha,beta) · 1` = `e(alpha,beta)` ✓
7. **Claim the reward** — FOOM Cash's `collect()` function accepts the proof, posts the nullifier, and transfers tokens
8. **Repeat with a new nullifierHash** — increment the nullifier, recompute `vk_x` and `C`, repeat

The attack was executed via a helper contract submitting crafted proofs in a loop. BlockSec's Phalcon detected the pattern in real time but couldn't prevent the drain.

---

## The Copycat Dimension

Five days between Veil Cash and FOOM Cash. BlockSec described FOOM as an **imitation attack** — someone saw the Veil exploit, found the same misconfiguration in FOOM's verifier, and repeated it. FOOM's losses ($2.26M) dwarfed Veil's (~4.5 ETH) because of higher TVL.

---

## How This Gets Introduced

**The snarkjs workflow:**

1. `snarkjs groth16 setup circuit.r1cs powersoftau.ptau circuit_0000.zkey` — creates the initial zkey with **no phase-2 contributions**. At this stage, `delta` and `gamma` are both set to the BN254 G2 generator point.
2. `snarkjs zkey contribute circuit_0000.zkey circuit_0001.zkey` — adds a phase-2 contribution, randomizing delta (gamma stays as-is). **This is the step that separates gamma from delta.**
3. `snarkjs zkey export verificationkey circuit_0001.zkey verification_key.json` — exports the key for the Solidity verifier.

If you skip step 2 and export from `circuit_0000.zkey`, you get a verification key where `gamma == delta == G2 generator`. The resulting verifier compiles, deploys, and accepts honest proofs — but has zero soundness.

**The specific values to check for** (BN254 G2 generator, which is what delta/gamma default to before phase 2):

```
x₀ = 10857046999023057135944570762232829481370756359578518086990519993285655852781
x₁ = 11559732032986387107991004021392285783925812861821192530917403151452391805634
y₀ = 8495653923123431417604973247489272438418190587263600148770280649306958101930
y₁ = 4082367875863433681332203403145435568316851327593401208105741076214120093531
```

If your verifier's `delta2` constants match these values, you have no phase 2 ceremony. Soundness is zero.

**Minimal defense:** Before deploying any Groth16 verifier, `assert(gamma2 != delta2)`. One comparison. If they're equal, don't deploy.

---

## Why This Hadn't Been Exploited Before

1. Most Groth16 deployments (Tornado Cash, Zcash, Semaphore) follow the snarkjs ceremony correctly
2. No one had documented this as a specific deployment pitfall
3. Both Veil Cash and FOOM Cash launched without external security review
4. The attack is invisible to circuit auditing — the vulnerability lives in the verification key constants, not the constraint system

---

## What This Adds

**The code was probably fine. The key generation process was broken.** The [Zcash BCTV14 bug](/posts/zcash-bctv14-trusted-setup-flaw/) showed a flaw in the *math spec* of key generation is catastrophic. This shows a flaw in the *execution* of key generation is equally catastrophic.

Minimal sanity check before deploying any Groth16 verifier: **assert `gamma != delta`**. One comparison. If they're equal, soundness is zero.

---

## Related Posts

- [Zcash BCTV14 Trusted Setup Flaw: Counterfeiting Shielded ZEC](/posts/zcash-bctv14-trusted-setup-flaw/) — another `TRUSTED SETUP` class vulnerability where a mathematical flaw in key generation enabled undetectable counterfeiting
- [Under-Constrained Signals in Tornado Cash, Circom-Pairing, and ZK Email](/posts/under-constrained-signals-tornado-circom-zkemail/) — FOOM Cash markets itself as "upgraded Tornado Cash"
- [Tornado Cash Governance Takeover and Malicious IPFS Frontend](/posts/tornado-cash-governance-takeover-malicious-frontend/) — Tornado Cash forks keep inheriting different vulnerability classes
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Exploit Reporting

- [CryptoTimes: "FOOMCASH Loses $2.26M in Copycat zkSNARK Exploit" (Feb 26, 2026)](https://www.cryptotimes.io/2026/02/26/foomcash-loses-2-26m-in-copycat-zksnark-exploit/)
- [Cryptopolitan: "'Upgraded Tornado Cash' Foom.Cash faces almost $2.3M loss in exploit"](https://www.cryptopolitan.com/foom-cash-faces-2-3m-loss-in-exploit/)
- [Phemex News: "FOOMCASH Loses $2.26M in Copycat Attack"](https://phemex.com/news/article/foomcash-loses-226-million-in-copycat-attack-exploiting-zksnark-vulnerability-62810)

### Security Firm Alerts

- [CertiK Alert: "~$1.8M exploit/whitehat rescue on @FOOMCASH — delta2==gamma2 root cause"](https://x.com/CertiKAlert/status/2026935120943526178)
- [BlockSec Phalcon: "ALERT! Imitation Attack Targets @FOOMCASH on Base & Ethereum"](https://x.com/Phalcon_xyz/status/2026941738141778394)
- [GoPlus Security: FOOM Cash exploit alert](https://x.com/GoPlusZH/status/2027002497911623960)
- [Apex777.eth: Veil Cash whitehat recovery by @DefimonAlerts](https://x.com/apex_ether/status/2024928539410141516)

### On-Chain

- [FOOM Cash Groth16 verifier — Ethereum (0xc043...71A6)](https://etherscan.io/address/0xc043865fb4D542E2bc5ed5Ed9A2F0939965671A6#code)
- [FOOM Cash Groth16 verifier — Base (0x02c3...68C6)](https://basescan.org/address/0x02c30D32A92a3C338bc43b78933D293dED4f68C6)
- [Phalcon Compliance: Exploiter risk report](https://blocksec.com/phalcon/compliance/report/ethereum/0x776baadc935bd1a2339b45953063ee497a24b246)

### Protocol Pages

- [FOOM Cash](https://foom.cash/)
- [Veil Cash docs](https://docs.veil.cash/)
- [Veil Cash deployments](https://docs.veil.cash/veil-cash-on-base/deployments)
- [Veil Cash GitHub](https://github.com/veildotcash)

### Groth16 — Original Paper

- Jens Groth, "On the Size of Pairing-based Non-interactive Arguments" (EUROCRYPT 2016) — [ePrint 2016/260](https://eprint.iacr.org/2016/260)

### Groth16 — Academic & Technical References

- [Simulation Extractable Versions of Groth's zk-SNARK Revisited (2020)](https://eprint.iacr.org/2020/1306)
- [HackMD: "On the weak simulation extractability of Groth16 arguments"](https://hackmd.io/@Oana/BJrS2IOJ6)
- [LambdaClass: "An overview of the Groth16 proof system"](https://blog.lambdaclass.com/groth16/)

### Groth16 Proof Malleability (Prior Art — Different Attack Class)

- [ethresear.ch: "Transaction Malleability Attack of Groth16 Proof"](https://ethresear.ch/t/transaction-malleability-attack-of-groth16-proof/15881)
- [Sui Foundation: "On the Malleability of Groth16 Proofs"](https://blog.sui.io/malleability-groth16-zkproof/)
- [snarkjs Issue #383: "Malleability in groth16 leads to double spending"](https://github.com/iden3/snarkjs/issues/383)
