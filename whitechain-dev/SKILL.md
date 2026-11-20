---
name: whitechain-dev
description: >
  Whitechain developer playbook. Use for any task involving Whitechain development on
  Whitechain Sepolia testnet (default) or Whitechain Mainnet (only when the user names it):
  connect to whitechain, add whitechain network to wallet, whitechain RPC url, whitechain chain ID,
  whitechain mainnet, chain ID 1875, deploy to whitechain mainnet,
  deploy contract to whitechain, compile solidity, forge deploy, deploy to whitechain testnet,
  create ERC-20 on whitechain, create NFT on whitechain, verify contract on whitechain,
  submit source code to blockscout, verify on blockscout, get testnet WBT, claim faucet,
  I need test tokens, run whitechain node, self-hosted RPC, run full node.
  Do not use this skill for transferring tokens, swapping, signing messages, or bridging —
  these are not yet supported (planned for a separate end-user MCP skill).
---

# Whitechain developer playbook

## Routing table

| Task | Trigger phrases | Reference |
|---|---|---|
| Connect wallet / network config | "connect to whitechain", "add network to wallet", "what's the RPC", "chain ID" | references/network.md |
| Deploy a contract | "deploy contract", "compile and deploy", "create ERC-20", "create NFT", "forge deploy" | references/deploy-contract.md |
| Verify a contract | "verify contract", "submit source code", "verify on blockscout" | references/verify-contract.md |
| Get testnet WBT | "get testnet WBT", "claim faucet", "I need test tokens" | references/claim-testnet-wbt.md |
| Run a node | "run whitechain node", "self-hosted RPC", "run full node" | references/run-node.md |

## Developer context

This skill assumes the developer controls their own signing key (Foundry keystore, environment
variable, or equivalent) and is running Claude Code or a terminal-capable agent. The agent
calls Foundry (`forge`/`cast`) and shell tools directly — no MCP approval flow is involved.

Transferring tokens, swapping, or bridging from a chat interface is out of scope for this
skill. A dedicated end-user MCP skill for those flows is planned but not yet built.

## Networks

| Property | Whitechain Sepolia (default) | Whitechain Mainnet |
| --- | --- | --- |
| Chain ID | `1874` | `1875` |
| RPC | `https://rpc.testnet.whitechain.io` | `https://rpc.whitechain.io` |
| Explorer | `https://explorer.testnet.whitechain.io` | `https://explorer.whitechain.io` |
| Blockscout API | `https://explorer.testnet.whitechain.io/api/v2` | `https://explorer.whitechain.io/api/v2` |
| Native token | WBT (test, from the faucet) | WBT (real value, no faucet) |

### Choosing the network

- **Whitechain Sepolia is the default.** Use it whenever the user does not name a network.
- **Use Whitechain Mainnet only when the user names it explicitly** in the request: "mainnet", "Whitechain Mainnet", or chain ID `1875`. Do not infer Mainnet from words like "production", "live", or "real", and do not switch to Mainnet on your own because a testnet step failed. If the target is unclear, ask, and use Sepolia until the user answers.
- Say which network you are using in every summary and result.

## Safety guardrails

- **Never commit private keys or accept them pasted in chat** — use Foundry's encrypted keystore (`cast wallet import --interactive`); testnet key hygiene should be no different from production key hygiene
- **Always show a summary before any write operation** (contract call, deployment) — silent writes are irreversible once mined
- **Wait for explicit confirmation before signing** — do not infer confirmation from an unrelated follow-up message
- **Mainnet writes need a separate, explicit Mainnet confirmation** — before any Mainnet deployment or contract call, show the Mainnet summary (network, chain ID `1875`, account, what will be sent, that it costs real WBT and cannot be undone) and continue only after the user replies that they want to proceed on Mainnet, for example "yes, deploy to mainnet". A plain "yes", "ok", or "go" is not enough for Mainnet. A confirmation for Sepolia never carries over to Mainnet
- **Validate chain ID before signing** — confirm the wallet/RPC is on the network the user chose: `1874` for Whitechain Sepolia, `1875` for Whitechain Mainnet. Signing against the wrong chain risks replay on another network
- **Validate all user-provided shell inputs** before constructing `forge`/`cast`/`solc` commands — no spaces, semicolons, pipes, or backticks (see each reference's Input validation section)

## Operating procedure

1. **Classify the task** using the routing table above
2. **Pick the network** per "Choosing the network": Sepolia unless the user named Mainnet
3. **Read the relevant reference** before implementing — don't rely on memory of the API shape
4. **Validate inputs** per the reference's Input validation section before building any shell command
5. **Implement**, showing a summary and waiting for confirmation before any write or signing step; on Mainnet, wait for the explicit Mainnet confirmation
6. **Deliver** the result — network, transaction hash, explorer link, or returned value — plus any manual steps the user still needs to do (e.g. funding the signer)

## For edge cases and API changes

Blockscout's API and Whitechain's RPC can change without notice. If a reference's endpoint or
response shape stops matching reality:

- Check the live docs: https://docs.whitechain.io
- Check the Blockscout OpenAPI spec on the explorer itself: `https://explorer.testnet.whitechain.io/api-docs` (Sepolia) or `https://explorer.whitechain.io/api-docs` (Mainnet)
- Report the mismatch to the user rather than guessing at a fixed shape

## Installation

```
npx skills add whitechain-labs/skills --skill whitechain-dev
```
