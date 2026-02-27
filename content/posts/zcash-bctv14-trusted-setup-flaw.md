---
title: "Zcash BCTV14 Trusted Setup Flaw: Counterfeiting Shielded ZEC"
subtitle: "Zcash, the BCTV14 Trusted Setup, and an Unforgeable Bug Kept Secret for Seven Months"
date: 2026-02-20
description: "A mathematical flaw in the BCTV14 paper allowed counterfeiting shielded ZEC. Zcash fixed it silently and disclosed seven months later."
tags: ["zk", "trusted-setup", "zcash"]
---

# The Flaw in the Paper Itself
### Zcash, the BCTV14 Trusted Setup, and an Unforgeable Bug Kept Secret for Seven Months

> **ZK Exploits Series**
>
> **Vulnerability class:** `TRUSTED SETUP` — Mathematical flaw in key generation; extra toxic waste
>
> **Protocol:** Zcash (shielded transactions, Sprout parameters)
>
> **Chain:** Zcash
>
> **Year:** 2018 (discovery) / 2019 (public disclosure) — CVE-2019-7167

---

| | |
|---|---|
| **CVE** | CVE-2019-7167 |
| **Date Discovered** | March 2018 |
| **Date Fixed** | October 2018 (Sapling upgrade) |
| **Date Disclosed** | February 2019 (7 months after fix) |
| **Chain** | Zcash |
| **Funds at Risk** | Undetectable infinite ZEC counterfeiting |
| **Funds Stolen** | Unknown — shielded pool is private; cannot be audited |
| **Discovered By** | Ariel Gabizon (then Zcash engineer, now Aztec) |
| **References** | [Zcash disclosure](https://z.cash/blog/zcash-counterfeiting-vulnerability-successfully-remediated/) · [CVE-2019-7167](https://nvd.nist.gov/vuln/detail/CVE-2019-7167) · [Ariel Gabizon's explanation](https://electriccoin.co/blog/new-research-on-zk-snarks/) |

---

## Background

Zcash's shielded transactions use the **BCTV14 zk-SNARK** (Ben-Sasson, Chiesa, Tromer, Virza 2014). BCTV14 requires a **trusted setup ceremony** that generates proving/verification keys and produces secret randomness (*"toxic waste"*) that must be destroyed. Zcash conducted its ceremony in 2016 with six participants — only one needed to destroy their randomness for security.

This vulnerability is not about toxic waste leakage. It's a **mathematical flaw in the BCTV14 construction itself**.

---

## The Vulnerability

The BCTV14 key generation algorithm, due to an error in the original paper, produced **extra α-shifted evaluation elements** in the proving key that should not have been there. These extra elements gave a malicious prover additional degrees of freedom — enough to construct valid proofs for **false statements**. In Zcash's case: proofs claiming to spend shielded notes that didn't exist. Unlimited ZEC from thin air.

What makes this catastrophic: **the shielded pool is private by design**. There is no public way to audit total shielded ZEC supply. An attacker could inflate it without leaving any observable trace.

---

## The Response

Gabizon discovered the flaw in March 2018. The team needed to replace the entire proving system without disclosing the vulnerability.

The **Sapling upgrade** (October 2018) replaced BCTV14 with **Groth16** and conducted a new trusted setup ceremony. It was framed publicly as a performance improvement (~100x faster proof generation) without disclosing the security motivation. Public disclosure came in February 2019, after the fix had been live for months.

---

## What Remains Unknown

The Zcash team believes the vulnerability was never exploited. But by the nature of the shielded pool, **this cannot be proven**. If the supply was inflated, there is no way to detect it.

---

## Why This Stands Out

This is not a circuit bug or an implementation error. It is an error in the **academic paper that defined the protocol**, discovered after a production deployment with a multi-party ceremony and years of real usage. The ceremony can be executed perfectly, every participant can destroy their toxic waste, and the system is still broken if the math was wrong.

---

## How to Catch This

**Why it happens:** The BCTV14 key generation algorithm included extra polynomial evaluation elements that shouldn't have been in the proving key. This wasn't a typo — it was a mathematical oversight in the security proof. The construction was published, peer-reviewed, and implemented in production. Nobody noticed until Gabizon re-derived the security reduction from scratch.

**What to check:**

This is the hardest vulnerability class to prevent — there's no linter for mathematical proofs.

1. **Don't roll your own proof system.** Use Groth16, PlonK, or Halo2 with established, audited implementations (snarkjs, gnark, arkworks, bellman). These have been scrutinized by multiple teams over years.
2. **If you must use a custom or lesser-known construction,** get independent formal verification of the mathematical security reduction — not just a code audit.
3. **Understand what your trusted setup produces.** The proving key should contain exactly what the security proof requires — no extra elements. If an element in the proving key is not referenced in the completeness/soundness proof, it's a red flag.
4. **When possible, avoid trusted-setup schemes entirely.** Transparent proof systems (STARKs, Halo2 without trusted setup) eliminate this entire attack surface.

---

## Related Posts

- [Groth16 Soundness Collapse: Veil Cash and FOOM Cash](/posts/groth16-soundness-collapse-veil-foom-cash/) — a related trusted setup failure where gamma == delta in a deployed Groth16 verifier collapses soundness entirely
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Primary Disclosure

- [Zcash: Counterfeiting Vulnerability Successfully Remediated (Feb 5, 2019)](https://z.cash/blog/zcash-counterfeiting-vulnerability-successfully-remediated/)
- [Electric Coin Company: New Research on zk-SNARKs](https://electriccoin.co/blog/new-research-on-zk-snarks/) — Ariel Gabizon's technical explanation

### CVE

- [CVE-2019-7167 (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2019-7167)
- [CVE-2019-7167 (MITRE)](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-7167)

### Academic Paper

- [BCTV14: "Succinct Non-Interactive Zero Knowledge for a von Neumann Architecture" — Ben-Sasson, Chiesa, Tromer, Virza (2014)](https://eprint.iacr.org/2013/879)
- [IEEE S&P 2014 published version](https://doi.org/10.1109/SP.2014.36)

### Sapling Upgrade (The Fix)

- [ZIP-205: Sapling Network Upgrade](https://zips.z.cash/zip-0205)
- [Zcash Sapling upgrade page](https://z.cash/upgrade/sapling/)
- [Electric Coin Company: Sapling Activation Complete](https://electriccoin.co/blog/sapling-activation-complete/)
- [Zcash v2.0.0 release (GitHub)](https://github.com/zcash/zcash/releases/tag/v2.0.0)

### Community Trackers

- [0xPARC ZK Bug Tracker #13 — Zcash Trusted Setup Leak](https://github.com/0xPARC/zk-bug-tracker#13-zcash-trusted-setup-leak)
- [ZKSecurity zkbugs database](https://github.com/zksecurity/zkbugs)

### Related Reading

- [Gabizon, Williamson, Ciobotaru: "Simulation Extractable Versions of Groth's zk-SNARK" (2018)](https://eprint.iacr.org/2018/187)
- [a16z crypto: On-Chain Trusted Setup Ceremony](https://a16zcrypto.com/posts/article/on-chain-trusted-setup-ceremony/)
