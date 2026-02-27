---
title: "ZKSync Airdrop: $5M Drained via Compromised Admin Key"
subtitle: "ZKSync's Compromised Admin Key -- Not a ZK Bug"
date: 2026-02-25
description: "An attacker compromised a privileged admin key to drain $5M in unclaimed ZKSync airdrop tokens. No cryptographic flaw, just opsec failure."
tags: ["zk", "opsec", "zksync"]
---

# The $5M Airdrop Heist
### ZKSync's Compromised Admin Key — Not a ZK Bug

> **ZK Exploits Series**
>
> **Vulnerability class:** `OPSEC` — Compromised private key; privileged admin function abuse
>
> **Protocol:** ZKSync (ZK rollup by Matter Labs)
>
> **Chain:** Ethereum L2
>
> **Year:** April 2025

---

| | |
|---|---|
| **Date** | April 2025 |
| **Chain** | Ethereum / ZKSync Era L2 |
| **Protocol** | ZKSync (Matter Labs) |
| **Attack Type** | Compromised admin private key |
| **Funds Stolen** | ~111 million ZK tokens (~$5M at time of incident) |
| **Outcome** | Tokens minted from unclaimed airdrop allocation; Matter Labs disclosed publicly; investigation ongoing |
| **References** | [Matter Labs official statement](https://zksync.mirror.xyz/airdrop-security-incident) |

---

## What Happened

ZKSync airdropped governance tokens in 2024. Unclaimed tokens sat in a distribution contract with a privileged `sweepUnclaimed()` function, callable only by an admin address.

In April 2025, an attacker compromised that admin key and called `sweepUnclaimed()`, transferring all 111M unclaimed ZK tokens to their wallet. No cryptography involved — just a key compromise and a function call.

---

## Why It's in This Series

Not a ZK vulnerability. Included because a single private key protecting a `sweepUnclaimed()` function on a $5M reserve is an **operational key management** failure. Functions that can move nine-figure token amounts need HSMs, multisigs, or timelocks.

---

## The Pattern Across This Series

**ZK cryptography itself has an excellent track record.** The underlying math — elliptic curves, polynomial commitments, pairings — has not been broken in any of these incidents. What breaks is the implementation, the composition, and the operational layer: circuit code that assigns without constraining, recursive pipelines with incompatible fields, governance mechanisms, and single private keys.

---

## How This Gets Introduced

A privileged function (`sweepUnclaimed()`) protected by a single EOA key. The ZK protocol itself was never at risk — but the surrounding infrastructure was.

**What to check:** Any function that can move tokens or change protocol state needs more than a single private key. Use multisig (Safe), timelocks, or role-based access control. This is basic smart contract security, not ZK-specific — but ZK protocol teams sometimes focus all their security effort on the circuit layer and leave the contract layer under-protected.

---

## Related Posts

- [Tornado Cash Governance Takeover and Malicious IPFS Frontend](/posts/tornado-cash-governance-takeover-malicious-frontend/) — similar opsec/operational failure class, different vector
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

- [Halborn: Explained — The ZKSync Hack (April 2025)](https://www.halborn.com/blog/post/explained-the-zksync-hack-april-2025)
- [ZKSync Era Explorer](https://explorer.zksync.io/)
- [0xPARC ZK Bug Tracker](https://github.com/0xPARC/zk-bug-tracker)
