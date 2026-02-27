---
title: "Groth16 delta==gamma On-Chain Scan: Who Skipped Phase 2?"
subtitle: "A Systematic Scan of Every Groth16 Verifier We Could Find"
date: 2026-02-27
description: "We scanned Groth16 verifiers across Ethereum, Base, BSC, Avalanche, and GitHub for the delta==gamma misconfiguration that collapsed Veil Cash and FOOM Cash."
tags: ["zk", "groth16", "trusted-setup", "security-research"]
---

# Groth16 delta==gamma On-Chain Scan: Who Skipped Phase 2?
### A Systematic Scan of Every Groth16 Verifier We Could Find

> **ZK Exploits Series** *(research addendum — February 2026)*
>
> **Vulnerability class:** `TRUSTED SETUP` / `DEPLOYMENT` — Groth16 verification key misconfiguration; delta == gamma collapses soundness
>
> **Related:** [Groth16 Soundness Collapse: Veil Cash and FOOM Cash](/posts/groth16-soundness-collapse-veil-foom-cash/)

---

## The Check

The BN254 G2 generator point has these coordinates:

```
x1 = 11559732032986387107991004021392285783925812861821192530917403151452391805634
x2 = 10857046999023057135944570762232829481370756359578518086990519993285655852781
y1 = 4082367875863433681332203403145435568316851327593401208105741076214120093531
y2 = 8495653923123431417604973247489272438418190587263600148770280649306958101930
```

In snarkjs, `gamma` being the G2 generator is **normal** (some variants fix gamma = G2 gen). The red flag is when **delta is also the G2 generator** — that means no phase 2 ceremony was performed. Delta stayed at its initial value. If delta == gamma, the Groth16 pairing check degenerates to `1 = 1` and anyone can forge proofs.

Reference: [EIP-197 — BN254 Pairing Precompile](https://eips.ethereum.org/EIPS/eip-197)

---

## Confirmed Vulnerable (Exploited)

### Veil Cash

| | |
|---|---|
| **Chain** | Base |
| **Verifier** | [`0x1E65C075989189E607ddaFA30fa1a0001c376cfd`](https://basescan.org/address/0x1E65C075989189E607ddaFA30fa1a0001c376cfd) |
| **Pool (0.1 ETH)** | [`0xD3560eF60Dd06E27b699372c3da1b741c80B7D90`](https://basescan.org/address/0xD3560eF60Dd06E27b699372c3da1b741c80B7D90) |
| **delta == gamma?** | **YES** — both G2 generator |
| **Loss** | ~2.9 ETH (29 forged withdrawals with nullifiers `0xdead0000`–`0xdead001c`) |
| **Attacker contract** | [`0x5F68aD46F500949FA7E94971441F279A85cB3354`](https://basescan.org/address/0x5F68aD46F500949FA7E94971441F279A85cB3354) |
| **Phase 2 ceremony** | None |

- [Veil Cash GitHub](https://github.com/veildotcash/veildotcash_contracts)
- [Veil Cash docs — deployments](https://docs.veil.cash/veil-cash-on-base/deployments)
- [PoC exploit repo](https://github.com/DK27ss/VeilCash-5K-PoC)
- [Evgenii — Forging zkSNARK Proofs via Misconfigured Verification Keys (CoinsBench)](https://coinsbench.com/forging-zksnark-proofs-via-misconfigured-verification-keys-the-veil-01-eth-exploit-2a6bb7d0078b)

### FOOM Cash

| | |
|---|---|
| **Chain** | Ethereum + Base |
| **Verifier (ETH)** | [`0xc043865fb4D542E2bc5ed5Ed9A2F0939965671A6`](https://etherscan.io/address/0xc043865fb4D542E2bc5ed5Ed9A2F0939965671A6#code) |
| **Verifier (Base)** | [`0x02c30D32A92a3C338bc43b78933D293dED4f68C6`](https://basescan.org/address/0x02c30D32A92a3C338bc43b78933D293dED4f68C6) |
| **delta == gamma?** | **YES** — both G2 generator |
| **Loss** | ~$2.26M total ($1.83M ETH, $427K Base) |
| **Phase 2 ceremony** | None |

- [FOOM Cash](https://foom.cash/)
- [CryptoTimes — FOOMCASH Loses $2.26M](https://www.cryptotimes.io/2026/02/26/foomcash-loses-2-26m-in-copycat-zksnark-exploit/)
- [Cryptopolitan — Foom.Cash $2.3M exploit](https://www.cryptopolitan.com/foom-cash-faces-2-3m-loss-in-exploit/)
- [Phemex — FOOMCASH Copycat Attack](https://phemex.com/news/article/foomcash-loses-226-million-in-copycat-attack-exploiting-zksnark-vulnerability-62810)
- [CertiK alert](https://x.com/CertiKAlert/status/2026935120943526178)
- [BlockSec Phalcon alert](https://x.com/Phalcon_xyz/status/2026941738141778394)

### FOOM Lottery (Terrestrials/foomlottery)

| | |
|---|---|
| **Chain** | Base + Ethereum |
| **Contract (Base)** | [`0xdb203504ba1fea79164AF3CeFFBA88C59Ee8aAfD`](https://basescan.org/address/0xdb203504ba1fea79164AF3CeFFBA88C59Ee8aAfD) |
| **Contract (ETH)** | [`0x239AF915abcD0a5DCB8566e863088423831951f8`](https://etherscan.io/address/0x239AF915abcD0a5DCB8566e863088423831951f8) |
| **delta == gamma?** | **YES** — all 10 verifier contracts vulnerable |
| **Phase 2 ceremony** | None (uses PSE phase 1 only) |
| **FOOM token (Base)** | [`0x02300aC24838570012027E0A90D3FEcCEF3c51d2`](https://basescan.org/address/0x02300aC24838570012027E0A90D3FEcCEF3c51d2) |

- [foomlottery GitHub](https://github.com/Terrestrials/foomlottery)
- Part of the FOOM ecosystem, active as of Feb 26, 2026

---

## Confirmed Safe (delta != gamma, proper ceremony)

### Tornado Cash (original)

| | |
|---|---|
| **Chain** | Ethereum |
| **Withdraw Verifier** | [`0x09193888b3f38c82dedfda55259a82c0e7de875e`](https://etherscan.io/address/0x09193888b3f38c82dedfda55259a82c0e7de875e) |
| **Reward Verifier** | [`0x88fd245fedec4a936e700f9173454d1931b4c307`](https://etherscan.io/address/0x88fd245fedec4a936e700f9173454d1931b4c307) |
| **Tree Update Verifier** | [`0x653477c392c16b0765603074f157314cc4f40c32`](https://etherscan.io/address/0x653477c392c16b0765603074f157314cc4f40c32) |
| **delta == gamma?** | **NO** — delta starts with `21280594949518...` |
| **Phase 2 ceremony** | Yes, 1,114 participants |

- [tornado-core Verifier.sol](https://github.com/tornadocash/tornado-core/blob/master/contracts/Verifier.sol)
- [Tornado Cash trusted setup ceremony](https://tornado-cash.medium.com/the-biggest-trusted-setup-ceremony-in-the-world-3c6ab9c8fffa)

### Railgun

| | |
|---|---|
| **Chain** | Ethereum, Polygon, BSC, Arbitrum |
| **Verifier (ETH)** | [`0x4025ee6512DBbda97049Bcf5AA5D38C54aF6bE8a`](https://etherscan.io/address/0x4025ee6512DBbda97049Bcf5AA5D38C54aF6bE8a) |
| **Verifier (Polygon)** | [`0xc7FfA542736321A3dd69246d73987566a5486968`](https://polygonscan.com/address/0xc7FfA542736321A3dd69246d73987566a5486968) |
| **Verifier (BSC)** | [`0x741936fb83DDf324636D3048b3E6bC800B8D9e12`](https://bscscan.com/address/0x741936fb83DDf324636D3048b3E6bC800B8D9e12) |
| **Verifier (Arbitrum)** | [`0x5aD95C537b002770a39dea342c4bb2b68B1497aA`](https://arbiscan.io/address/0x5aD95C537b002770a39dea342c4bb2b68B1497aA) |
| **delta == gamma?** | **NO** |
| **Phase 2 ceremony** | Yes, Perpetual Powers of Tau + circuit-specific phase 2 for 54+ circuits |
| **TVL** | ~$80–105M |

- [Railgun docs — trusted setup](https://docs.railgun.org/wiki/learn/privacy-system/trusted-setup-ceremony)
- [Railgun docs — Groth16 prover](https://docs.railgun.org/developer-guide/wallet/getting-started/6.-load-a-groth16-prover-for-each-platform)

### RISC Zero

| | |
|---|---|
| **Chain** | Ethereum |
| **Groth16Verifier v1** | [`0x7CCA385bdC790c25924333F5ADb7F4967F5d1599`](https://etherscan.io/address/0x7CCA385bdC790c25924333F5ADb7F4967F5d1599) |
| **Groth16Verifier v3** | [`0x2a098988600d87650Fb061FfAff08B97149Fa84D`](https://etherscan.io/address/0x2a098988600d87650Fb061FfAff08B97149Fa84D) |
| **delta == gamma?** | **NO** — delta starts with `16683235016...` |

- [RISC Zero](https://www.risczero.com/)

### SP1 / Succinct

| | |
|---|---|
| **Chain** | Ethereum |
| **SP1Verifier v3** | [`0x6A87EFd4e6B2Db1ed73129A8b9c51aaA583d49e3`](https://etherscan.io/address/0x6A87EFd4e6B2Db1ed73129A8b9c51aaA583d49e3) |
| **delta == gamma?** | **NO** — delta starts with `20409334339...` |

- [Succinct Labs](https://succinct.xyz/)

### Privacy Pools (0xBow)

| | |
|---|---|
| **Chain** | Ethereum |
| **Entrypoint** | [`0x6818809EefCe719E480a7526D76bD3e561526b46`](https://etherscan.io/address/0x6818809EefCe719E480a7526D76bD3e561526b46) |
| **Privacy Pools** | [`0xf241d57c6debae225c0f2e6ea1529373c9a9c9fb`](https://etherscan.io/address/0xf241d57c6debae225c0f2e6ea1529373c9a9c9fb) |
| **delta == gamma?** | **NO** — delta starts with `355730187...` |
| **TVL** | ~$2.3M |

- [0xbow GitHub](https://github.com/0xbow-io/privacy-pools-core)
- [Privacy Pools on ethereum.org](https://ethereum.org/apps/privacy-pools/)

### Cyclone Protocol

| | |
|---|---|
| **Chain** | IoTeX, BSC, Ethereum, Polygon |
| **delta == gamma?** | **NO** — reuses Tornado Cash's delta (`21280594949518...`) |
| **Phase 2 ceremony** | Inherits from Tornado Cash |

- [Cyclone Protocol GitHub](https://github.com/cycloneprotocol/cyclone-contracts)
- [Cyclone docs](https://docs.cyclone.xyz)
- [Elliptic analysis](https://www.elliptic.co/blog/analysis/the-next-tornado-cash-six-protocols-vying-to-replace-the-famous-crypto-mixing-service)

### Tonic Cash

| | |
|---|---|
| **Chain** | Klaytn, WEMIX, Base |
| **delta == gamma?** | **NO** — uses Tornado Cash's delta values |
| **Phase 2 ceremony** | Inherits from Tornado Cash (1,100+ participants) |

- [Tonic Cash GitHub](https://github.com/tonic-cash/tonic-core)

### Whirlwind Cash

| | |
|---|---|
| **Chain** | BSC |
| **delta == gamma?** | **NO** — neither delta nor gamma is the G2 generator |
| **Phase 2 ceremony** | Ran their own independent ceremony |

- [Whirlwind Cash GitHub](https://github.com/WhirlwindCash/contracts)

### Typhoon Cash

| | |
|---|---|
| **Chain** | BSC |
| **Verifier** | [`0x7036a0555637B88C5a68C2a39799C7A9AB858C60`](https://bscscan.com/address/0x7036a0555637B88C5a68C2a39799C7A9AB858C60) |
| **delta == gamma?** | **NO** — delta starts with `35089135162969...` |
| **TVL** | ~$33–67K |

- [Typhoon Cash on DeFiLlama](https://defillama.com/protocol/typhoon-cash)

### Sherpa Cash

| | |
|---|---|
| **Chain** | Avalanche |
| **Verifier** | [`0xFC0829E792b0D10DF95B895d56FaD4712DA30B25`](https://snowtrace.io/address/0xFC0829E792b0D10DF95B895d56FaD4712DA30B25) |
| **delta == gamma?** | **NO** (inferred — ran dedicated ceremony) |
| **Phase 2 ceremony** | Yes, at ceremony.sherpa.cash |

- [Sherpa Cash ceremony instructions](https://ceremony.sherpa.cash/instructions)
- [Sherpa Cash Medium](https://medium.com/sherpa-cash/tagged/avalanche)
- [Torn community — Sherpa Cash intro](https://torn.community/t/introducing-a-tornado-fork-on-avalanche-sherpa-cash/837)

### Hinkal Protocol

| | |
|---|---|
| **Chain** | Ethereum, Arbitrum, Base, Optimism, Polygon, BNB, Avalanche |
| **delta == gamma?** | **NO** — delta starts with `42799555226547...` |
| **TVL** | ~$130K |

- [Hinkal GitHub](https://github.com/Hinkal-Protocol/Hinkal-Smart-Contracts)
- [Hinkal docs](https://hinkal-team.gitbook.io/hinkal/technical-description/smart-contracts)
- [ZKSecurity audit](https://zksecurity.xyz/reports/silent/)

### Webb Protocol / Tangle

| | |
|---|---|
| **Chain** | Multiple (Substrate + EVM) |
| **delta == gamma?** | **NO** — all verifiers have unique deltas |

- [Webb/Tangle GitHub — protocol-solidity](https://github.com/tangle-network/protocol-solidity)
- [Webb/Tangle GitHub — zero-knowledge-gadgets](https://github.com/tangle-network/zero-knowledge-gadgets)

### Worm Privacy

| | |
|---|---|
| **delta == gamma?** | **NO** |
| **Phase 2 ceremony** | Yes, [published ceremony artifacts](https://github.com/worm-privacy/trusted-setup/releases/tag/final) |

- [Worm Privacy GitHub](https://github.com/worm-privacy/worm)

### iden3

| | |
|---|---|
| **delta == gamma?** | **NO** |

- [iden3 contracts GitHub](https://github.com/iden3/contracts)

### ZK Email

| | |
|---|---|
| **delta == gamma?** | **NO** |

- [ZK Email JWT verifier GitHub](https://github.com/zkemail/jwt-tx-builder)

### Panther Protocol

| | |
|---|---|
| **Chain** | Polygon |
| **delta == gamma?** | **NO** |
| **Phase 2 ceremony** | Yes, [preZKPceremony repo](https://github.com/pantherprotocol/preZKPceremony) |
| **ZKP token** | [`0x9A06Db14D639796B25A6ceC6A1bf614fd98815EC`](https://polygonscan.com/address/0x9A06Db14D639796B25A6ceC6A1bf614fd98815EC) |

- [Panther Protocol docs — Groth16](https://docs.pantherprotocol.io/docs/learn/cryptographic-primitives/zk-snarks/groth16)

### zkBob

| | |
|---|---|
| **Chain** | Polygon, Optimism |
| **delta == gamma?** | Likely NO (proper ceremony documented) |

- [zkBob docs](https://docs.zkbob.com)
- [zkBob — Railgun comparison](https://blog.zkbob.com/zkbob-railgun-privacy-app-comparison/)

### Dark Forest

| | |
|---|---|
| **Chain** | Ethereum |
| **delta == gamma?** | **NO** — multiple verifiers, all with distinct deltas |

- [Dark Forest GitHub](https://github.com/darkforest-eth/eth)
- [Dark Forest circuit walkthrough](https://blog.zkga.me/df-init-circuit)

---

## Not Groth16 (Not Applicable)

| Protocol | Chain | Proving System | Links |
|---|---|---|---|
| Polygon zkEVM | Ethereum L2 | FFLONK | [`0x1C3A...BD8`](https://etherscan.io/address/0x1C3A3da552b8662CD69538356b1E7c2E9CC1EBD8) |
| zkSync | Ethereum L2 | PLONK | [zkSync Era Explorer](https://explorer.zksync.io/) |
| Loopring | Ethereum | Custom batch Groth16 | [`0x6150...1ef`](https://etherscan.io/address/0x6150343E0F43A17519c0327c41eDd9eBE88D01ef) |
| Aztec Connect | Ethereum | PLONK | [Aztec GitHub](https://github.com/AztecProtocol) |
| Starknet | Ethereum L2 | STARKs | Not applicable |

---

## Unable to Verify (No Public Source)

| Protocol | Chain | Risk | Notes | Links |
|---|---|---|---|---|
| ShadeCash | Fantom | Unknown | TC fork, no public repo, docs expired | — |
| Messier 87 / Black Hole | ETH, BSC, Polygon | Unknown | TC fork, offline, no source | [Elliptic analysis](https://www.elliptic.co/blog/analysis/the-next-tornado-cash-six-protocols-vying-to-replace-the-famous-crypto-mixing-service) |
| Buccaneer V3 | Ethereum | Unknown | Uses Gas Station Network, different arch | [Elliptic analysis](https://www.elliptic.co/blog/analysis/the-next-tornado-cash-six-protocols-vying-to-replace-the-famous-crypto-mixing-service) |
| Silent Protocol | Ethereum | Unknown | Source not public, ZKSecurity audit exists | [ZKSecurity audit](https://zksecurity.xyz/reports/silent/) |
| Nocturne | Ethereum (dead) | N/A | Shut down June 2024 | [CoinTelegraph](https://cointelegraph.com/news/vitalik-buterin-nocturne-labs-shuts-down) |

---

## Needs Storage Read (Dynamic VK)

| Protocol | Chain | Notes | Links |
|---|---|---|---|
| Railgun (legacy) | Ethereum | VK set via `initializeRailgunLogic()`, not hardcoded | [`0xc636...c09`](https://etherscan.io/address/0xc6368d9998ea333b37eb869f4e1749b9296e6d09) |
| Hermez v1 | Ethereum | Behind proxy, proper audits | [`0xa68d...633`](https://etherscan.io/address/0xa68d85df56e733a06443306a095646317b5fa633) |
| WorldCoin / World ID | Ethereum | Uses Semaphore, had proper ceremony | [Semaphore Phase 2 Setup](https://github.com/privacy-ethereum/semaphore-phase2-setup) |

---

## GitHub: Vulnerable Verifiers (delta == gamma)

Repos where `deltax1 == gammax1 == G2 generator`. Most are educational or experimental, but some have deployment configs.

### Potentially Deployed

| Repo | Stars | Notes | Link |
|---|---|---|---|
| **Terrestrials/foomlottery** | 4 | Deployed on Base + ETH, 10 vulnerable verifiers, active Feb 2026 | [GitHub](https://github.com/Terrestrials/foomlottery) |
| **masa-finance/masa-zkSBT** | 11 | Real org, has mainnet hardhat config, "template" repo | [GitHub](https://github.com/masa-finance/masa-zkSBT) |

### Educational / Experimental (vulnerable but not deployed with real funds)

| Repo | Stars | Notes | Link |
|---|---|---|---|
| nkrishang/tornado-cash-rebuilt | 248 | Popular TC clone for learning, explicitly educational | [GitHub](https://github.com/nkrishang/tornado-cash-rebuilt) |
| TheBojda/mini-zk-rollup | 8 | Minimalistic ZK rollup demo | [GitHub](https://github.com/TheBojda/mini-zk-rollup) |
| Manuel-yang/mixer | 0 | TC clone | [GitHub](https://github.com/Manuel-yang/mixer) |
| cqlyj/CzechOut | 3 | Face scanning payment project | [GitHub](https://github.com/cqlyj/CzechOut) |
| jose-blockchain/zk-vrf | 0 | ZK VRF | [GitHub](https://github.com/jose-blockchain/zk-vrf) |
| CleanPegasus/EthMail | 0 | Decentralized messaging | [GitHub](https://github.com/CleanPegasus/EthMail) |
| zk-bankai/zkure | 1 | — | [GitHub](https://github.com/zk-bankai/zkure) |
| AbiramiR-27/ZKBridge | 0 | — | [GitHub](https://github.com/AbiramiR-27/ZKBridge) |
| RezaNourmohammadi/ZK_Watermarking | 0 | — | [GitHub](https://github.com/RezaNourmohammadi/ZK_Watermarking) |
| roccomarotta123/HealthApp_ETH | 0 | — | [GitHub](https://github.com/roccomarotta123/HealthApp_ETH) |
| Ham3798/zk_did_scheme | 0 | — | [GitHub](https://github.com/Ham3798/zk_did_scheme) |
| thisisazaro/zk-health-iot | 0 | — | [GitHub](https://github.com/thisisazaro/zk-health-iot) |
| Consensus-Score/Consensus-Score | 0 | — | [GitHub](https://github.com/Consensus-Score/Consensus-Score) |
| zero-savvy/zk-remote-attestation | 0 | — | [GitHub](https://github.com/zero-savvy/zk-remote-attestation) |
| wabalabudabdab/iot | 0 | — | [GitHub](https://github.com/wabalabudabdab/iot) |
| hswopeams/composite-number-game | 0 | — | [GitHub](https://github.com/hswopeams/composite-number-game) |
| doutv/groth16VerifierContract | 0 | — | [GitHub](https://github.com/doutv/groth16VerifierContract) |
| kennyrimba/skripsi2 | 0 | Indonesian thesis project | [GitHub](https://github.com/kennyrimba/skripsi2) |
| topliceanu/IC3-dec-id | 0 | Voting verifier | [GitHub](https://github.com/topliceanu/IC3-dec-id) |
| hamiha70/zkFusion | 0 | — | [GitHub](https://github.com/hamiha70/zkFusion) |
| jinali98/zk-circom-projects | 0 | Learning project | [GitHub](https://github.com/jinali98/zk-circom-projects) |

### GitHub: Safe Verifiers (delta != gamma)

| Repo | Stars | Link |
|---|---|---|
| worm-privacy/worm | 12 | [GitHub](https://github.com/worm-privacy/worm) |
| cycloneprotocol/cyclone-contracts | 63 | [GitHub](https://github.com/cycloneprotocol/cyclone-contracts) |
| factorgroup/nightmarket | 46 | [GitHub](https://github.com/factorgroup/nightmarket) |
| ProofOfInnocence/privacy-pools-v1 | — | [GitHub](https://github.com/ProofOfInnocence/privacy-pools-v1) |
| stanbar/tornado-haze | — | [GitHub](https://github.com/stanbar/tornado-haze) |
| srv-smn/zk-cryptomixer-Tornado-Cash | — | [GitHub](https://github.com/srv-smn/zk-cryptomixer-Tornado-Cash) |
| smashcash/heco | — | [GitHub](https://github.com/smashcash/heco) |
| holyaustin/ZeroSecret | — | [GitHub](https://github.com/holyaustin/ZeroSecret) |
| emmaguo13/zk-blind | — | [GitHub](https://github.com/emmaguo13/zk-blind) |
| dulumao/ZKP_ERC20 | — | [GitHub](https://github.com/dulumao/ZKP_ERC20) |
| ashablovskiy/zkBOX | — | [GitHub](https://github.com/ashablovskiy/zkBOX) |
| phamnam1805/private-dao-circuits | — | [GitHub](https://github.com/phamnam1805/private-dao-circuits) |
| keeper402/Hongbao | — | [GitHub](https://github.com/keeper402/Hongbao) |
| spalladino/zkp-tests | — | [GitHub](https://github.com/spalladino/zkp-tests) |

---

## DeFiLlama Privacy Protocols (TVL Reference)

| Protocol | TVL | Link |
|---|---|---|
| Tornado Cash | ~$729M | [DeFiLlama](https://defillama.com/protocol/tornado-cash) |
| Railgun | ~$80–105M | [DeFiLlama](https://defillama.com/protocol/railgun) |
| Aztec | ~$9.7M | [DeFiLlama](https://defillama.com/protocol/aztec) |
| Privacy Pools (0xBow) | ~$2.3M | [DeFiLlama](https://defillama.com/protocol/privacy-pools) |
| 0x0.ai | ~$414K | [DeFiLlama](https://defillama.com/protocol/0x0-ai) |
| Hinkal | ~$130K | [DeFiLlama](https://defillama.com/protocol/hinkal) |
| Typhoon Cash | ~$33–67K | [DeFiLlama](https://defillama.com/protocol/typhoon-cash) |
| Silent Protocol | ~$35K | [DeFiLlama](https://defillama.com/protocol/silent-protocol) |

- [DeFiLlama — Privacy category](https://defillama.com/protocols/privacy)
- [DeFiLlama — Tornado Cash forks](https://defillama.com/forks/Tornado%20Cash)

---

## Methodology & Limitations

**What we checked:**
- Verified contract source on Etherscan, Basescan, BSCScan, Snowtrace
- GitHub repos for verifier source code
- GitHub code search for the G2 generator constant in delta variable names
- DeFiLlama fork/protocol listings

**What we couldn't check:**
- Unverified contracts (no source code on block explorer)
- Contracts behind proxies where VK is in storage
- Solana programs (different tooling, not BN254)
- Starknet (uses STARKs, not Groth16)
- Private repos

**The detection is trivially scriptable:** read `delta2` and `gamma2` from any Groth16 verifier and compare. If all 4 coordinates match, it's vulnerable. No dedicated scanner tool exists yet.

---

## Sources

### Exploit Coverage
- [Evgenii — Forging zkSNARK Proofs via Misconfigured Verification Keys (CoinsBench)](https://coinsbench.com/forging-zksnark-proofs-via-misconfigured-verification-keys-the-veil-01-eth-exploit-2a6bb7d0078b)
- [CryptoTimes — FOOMCASH $2.26M exploit](https://www.cryptotimes.io/2026/02/26/foomcash-loses-2-26m-in-copycat-zksnark-exploit/)
- [BitcoinEthereumNews — Foom.Cash exploit](https://bitcoinethereumnews.com/finance/upgraded-tornado-cash-foom-cash-faces-almost-2-3m-loss-in-exploit/)

### Technical References
- [EIP-197 — BN254 Pairing Precompile](https://eips.ethereum.org/EIPS/eip-197)
- [snarkjs Groth16 verifier template](https://github.com/iden3/snarkjs/blob/master/templates/verifier_groth16.sol.ejs)
- [snarkjs Security PR #36](https://github.com/iden3/snarkjs/pull/36)
- [Groth16 paper — ePrint 2016/260](https://eprint.iacr.org/2016/260)

### Protocol Aggregators
- [Elliptic — Six protocols replacing Tornado Cash](https://www.elliptic.co/blog/analysis/the-next-tornado-cash-six-protocols-vying-to-replace-the-famous-crypto-mixing-service)
- [DeFiLlama Privacy Protocols](https://defillama.com/protocols/privacy)
- [0xPARC ZK Bug Tracker](https://github.com/0xPARC/zk-bug-tracker)
