---
title: "Tornado Cash Governance Takeover and Malicious IPFS Frontend"
subtitle: "A Governance Takeover and a Malicious Frontend -- Neither a ZK Bug"
date: 2026-02-24
description: "Two non-cryptographic attacks on Tornado Cash: a governance takeover via malicious proposal in 2023, and a compromised IPFS frontend leaking deposits in 2024."
tags: ["zk", "governance", "tornado-cash"]
---

# Tornado Cash: When the Protocol Gets Attacked
### A Governance Takeover and a Malicious Frontend — Neither a ZK Bug

> **ZK Exploits Series**
>
> **Vulnerability classes:** `GOVERNANCE` · `SUPPLY CHAIN`
>
> **Protocol:** Tornado Cash (Ethereum privacy mixer)
>
> **Years:** May 2023 (governance) · February 2024 (frontend)

---

Tornado Cash was attacked twice in consecutive years — neither attack had anything to do with its ZK circuits. Both targeted the governance and infrastructure layers.

---

## Attack 1 — Governance Takeover via CREATE2 Sleight of Hand

| | |
|---|---|
| **Date** | May 20, 2023 |
| **Chain** | Ethereum |
| **Type** | Governance exploit — CREATE2 + SELFDESTRUCT address reuse |
| **Funds Stolen** | ~$1M (1.2M fraudulent TORN governance tokens) |
| **References** | [Post-mortem thread](https://twitter.com/samczsun/status/1660012956632104960) · [Etherscan: malicious proposal](https://etherscan.io/tx/0x3274df5c6e3c2) |

Tornado Cash is governed by TORN token holders through on-chain voting. The attacker submitted a proposal that *looked* identical to a previous legitimate one. Community members reviewed it, found no differences, and voted it through.

The deception: a hidden `emergencyStop()` function and the ability to self-destruct and redeploy.

1. Deploy a contract at address `A` with mostly innocent code, plus a hidden `emergencyStop()` function and a `SELFDESTRUCT` opcode
2. Submit it as a governance proposal; community votes it in (contract at `A` looks safe)
3. Before execution: call `emergencyStop()` to trigger `SELFDESTRUCT`, destroying the contract at `A`
4. Use `CREATE2` to **redeploy a completely different, malicious contract at the same address `A`** — CREATE2 allows deterministic address computation, so the same address can be reused after self-destruction
5. Execute the now-passed governance proposal, which runs the *new* malicious contract at `A`

The malicious redeployment minted 1.2M TORN to the attacker and drained the governance contract (~$1M). The community eventually passed a counter-proposal to undo the changes.

---

## Attack 2 — Malicious IPFS Frontend Leaks Deposit Notes

| | |
|---|---|
| **Date** | February 2024 (discovered; active since ~November 2023) |
| **Chain** | Ethereum |
| **Type** | Supply chain / frontend attack — malicious JavaScript injected via governance |
| **Outcome** | At least 1 deposit stolen; privacy of all users who interacted over ~2 months permanently compromised |
| **References** | [Community disclosure thread](https://twitter.com/samczsun/status/1760012934631408) |

Since OFAC sanctions (August 2022), Tornado Cash has had no official frontend — the canonical UI is hosted on IPFS, with the hash governed by the DAO. In November 2023, an attacker submitted a governance proposal to update the IPFS hash as a "routine UI improvement." It passed.

The new frontend's JavaScript included a **data exfiltration payload**. When a user deposited ETH and received their withdrawal note, the JS silently sent it to an attacker-controlled server. The note is the entire privacy guarantee — anyone who holds it can withdraw the funds.

The malicious frontend ran ~2 months before detection. Every user who deposited through the UI had their note leaked. At least one deposit was stolen. For the rest, **privacy is permanently broken** — the attacker knows the link between deposit and withdrawal addresses.

---

## Side by Side

| | Governance Takeover (2023) | IPFS Frontend (2024) |
|---|---|---|
| **Mechanism** | CREATE2 + SELFDESTRUCT address reuse | Malicious JS in governance-controlled frontend |
| **What was stolen** | TORN tokens (~$1M) | Deposit notes → at least 1 deposit |
| **Privacy impact** | None — ZK circuits unaffected | Permanent — all affected users deanonymized |
| **Attack sophistication** | High (subtle EVM mechanic) | Low (standard supply chain attack) |
| **ZK relevance** | None | None |

---

Neither attack reveals anything about ZK cryptography. They reveal that **the trust surface of a ZK protocol is much larger than its circuits**. A mixer with unbreakable circuits can still be drained through governance or a malicious frontend.

---

## How This Gets Introduced

These aren't ZK bugs — they're governance and supply chain bugs that happen to target a ZK protocol. But ZK protocol engineers should understand them because **your protocol's attack surface includes everything outside the circuit.**

1. **Governance proposals that deploy or upgrade contracts must be reviewed at the bytecode level**, not just the source. The CREATE2 + SELFDESTRUCT trick means the code at an address can change after community review. Timelocks between proposal approval and execution give defenders time to detect substitution.
2. **IPFS hashes in governance are code deployments.** An IPFS frontend update is equivalent to deploying new code that every user will execute. Review it accordingly.
3. **ZK protocols that handle real value need multisig/timelock on every privileged function** — not just the contract upgrade path, but also frontend hash updates, parameter changes, and emergency functions.

---

## Related Posts

- [Under-Constrained Signals in Tornado Cash, Circom-Pairing, and ZK Email](/posts/under-constrained-signals-tornado-circom-zkemail/) — the actual ZK circuit vulnerability in Tornado Cash (2019), a fundamentally different vulnerability class
- [Full series index](/posts/zk-exploits/)

---

## Sources & References

### Governance Takeover (May 2023)

- [Rekt News — Tornado Cash Governance](https://rekt.news/tornado-governance/)
- [Etherscan — Tornado Governance contract](https://etherscan.io/address/0x5efda50f22d34F262c29268506C5Fa42cB56A1Ce)

### Malicious IPFS Frontend (Late 2023 / Disclosed Feb 2024)

- Community disclosure via Gas404 on X

### General

- [0xPARC ZK Bug Tracker](https://github.com/0xPARC/zk-bug-tracker)
- [Tornado Cash core repository](https://github.com/tornadocash/tornado-core)
