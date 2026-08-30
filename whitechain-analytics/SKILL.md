---
name: whitechain-analytics
description: >
  Whitechain analytics playbook. Use when querying Whitechain Blockscout, JSON-RPC, GraphQL,
  or Etherscan-compat APIs; computing ecosystem health metrics; handling wei/BigInt; chain ID
  1874 testnet or 1875 legacy L1; Soul/SoulDrop on-chain data; rate limits 50 req/s;
  WhiteData judgement layer (not a Blockscout clone, not Dune).
  Do not use for wallet transfers, swapping, bridging, KYC, or linking Soul to a person.
---

# Whitechain analytics playbook

Public APIs have **no keys**. This skill is for reading chain data and building a judgement layer (health, flows, scoring). It is not an explorer clone and not a SQL warehouse.

## Default network

| Property | Value |
| --- | --- |
| Name | Whitechain Sepolia |
| Chain ID | `1874` (`0x752`) |
| RPC | `https://rpc.testnet.whitechain.io` |
| WebSocket | `wss://rpc.testnet.whitechain.io/ws` |
| Explorer | `https://explorer.testnet.whitechain.io` |
| REST v2 | `https://explorer.testnet.whitechain.io/api/v2` |
| Etherscan-compat | `https://explorer.testnet.whitechain.io/api` |
| GraphQL | `https://explorer.testnet.whitechain.io/api/v1/graphql` |
| Native token | WBT |
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` |

Legacy L1 (archive only): chain `1875` (`0x753`), RPC `https://rpc.whitechain.io`. Hostnames change meaning at Mainnet — always read `eth_chainId` before writes.

## Errors (HTTP 200 is not success)

Check the payload **before** reading data:

- JSON-RPC: `error`
- GraphQL: `errors`
- Etherscan-compat: `status` `"0"`
- REST v2: HTTP `422` with an `errors` array
- Unknown address: HTTP `200` with `null` fields — empty state, **not** an error
- `429`: exponential backoff from `Retry-After`

## Limits

50 req/s per IP. Public RPC is not sized for a block-by-block indexer.

- Live heads: `eth_subscribe(newHeads)` over WebSocket
- Aggregates: Blockscout REST pagination
- Batch reads: Multicall3
- GraphQL `first` ≤ 8
- L1 archive: throttle (example 5 rps)

## Numbers

Wei arrives as a decimal **string** and overflows `Number`. Use BigInt. Never `parseInt` / `parseFloat` / divide by `1e18`. ERC-20 decimals are per token — never assume 18.

Zero is zero. An unavailable source is **“metric unavailable”**, not zero.

## Soul / wording

- “Address with a verified Soul”, not “verified person”
- “Automation signals”, not “bot”
- On-chain data only. Never link Soul to identity. No KYC work.
- Do not copy SoulDrop compounding rates from L1 docs. Use on-chain values only after Soul contracts exist on this chain.

Exclude genesis sentinel (`2^256-1`) and OP Stack predeploys from metrics.

## Safety

- Never commit private keys or accept them in chat
- Confirm `eth_chainId` matches the intended network before any write
- Sample size under 100: do not draw a chart; report caveats

## Installation

```
npx skills add whitechain-labs/skills --skill whitechain-analytics
```
