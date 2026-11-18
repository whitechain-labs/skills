# Connecting to Whitechain

Whitechain Sepolia is the default network. Use Whitechain Mainnet only when the user names it
explicitly (see "Choosing the network" in `SKILL.md`).

## Network config

| Property | Whitechain Sepolia (default) | Whitechain Mainnet |
| --- | --- | --- |
| Network name | Whitechain Sepolia | Whitechain |
| Chain ID | `1874` (hex `0x752`) | `1875` (hex `0x753`) |
| RPC endpoint | `https://rpc.testnet.whitechain.io` | `https://rpc.whitechain.io` |
| Alternative RPC (Ankr) | `https://rpc.ankr.com/whitechain_testnet` | `https://rpc.ankr.com/whitechain` |
| WebSocket | `wss://rpc.testnet.whitechain.io/ws` | `wss://rpc.whitechain.io/ws` |
| Currency | WBT | WBT |
| Explorer | `https://explorer.testnet.whitechain.io` | `https://explorer.whitechain.io` |
| Blockscout API | `https://explorer.testnet.whitechain.io/api/v2` | `https://explorer.whitechain.io/api/v2` |
| Bridge | `https://bridge.testnet.whitechain.io` | `https://bridge.whitechain.io` |
| Faucet | `https://faucet.testnet.whitechain.io` | None. WBT has real value |
| L1 settlement | Ethereum Sepolia | Ethereum mainnet |

Differences that matter when moving from Sepolia to Mainnet:

- Mainnet caps a single transaction at 16,777,216 gas. Sepolia has no such cap, so a transaction that passes on Sepolia can fail on Mainnet
- Mainnet WebSocket serves `newHeads` and `logs` subscriptions only; `newPendingTransactions` is Sepolia only
- Set `evmVersion` to `prague` (or earlier): it is the highest target that runs on both networks

## Security

- **Never embed the RPC endpoint behind an API key in client-side code** — proxy through a backend if you use a keyed RPC plan, such as an Ankr Freemium or Premium key
- **Validate the chain ID before signing** — `1874` for Sepolia, `1875` for Mainnet. A transaction signed against the wrong chain ID can be replayed on another chain
- **Use HTTPS RPC endpoints only** — reject any `http://` endpoint to prevent credential/data interception
- **The public RPC is rate-limited** — on both networks, 50 rps sustained / burst 500 per IP; returns `429 Too Many Requests` with a `Retry-After: 1` header when exceeded; not a guarantee of uptime for anything time-sensitive
- **The Ankr endpoints are separate public RPCs** – `https://rpc.ankr.com/whitechain_testnet` (answers `1874`) and `https://rpc.ankr.com/whitechain` (answers `1875`) serve HTTPS with no API key. Ankr sets their limits, not Whitechain: about 1,800 requests per minute and an `eth_getLogs` range of up to 1,000 blocks on the public plan. WebSocket, `debug_*` and `trace_*` on Ankr need a paid Ankr plan (see https://www.ankr.com/docs/rpc-service/service-plans/). Use them as a fallback when the Whitechain RPC returns `429` or is down

## Wallet setup

Three ways to add the network, fastest first:

**Option 1 — Chainlist (one click)**
1. Open Chainlist: [Whitechain Sepolia](https://chainlist.org/?search=Whitechain+Sepolia&testnets=true) or [Whitechain Mainnet](https://chainlist.org/chain/1875)
2. Click **Connect Wallet**, then **Add to MetaMask**

**Option 2 — from the explorer**
1. Open the explorer: [Sepolia](https://explorer.testnet.whitechain.io/) or [Mainnet](https://explorer.whitechain.io/)
2. Click **Connect Wallet** in the top-right corner and approve the prompt

**Option 3 — manual**
1. In the wallet, choose **Add network** → **Add a network manually**
2. Enter the Chain ID, RPC endpoint, currency symbol, and explorer URL from the table above

After any option:
1. Confirm the wallet shows the network the user chose and its chain ID (`1874` or `1875`) before signing anything
2. On Sepolia, get test WBT from the faucet — see [claim-testnet-wbt.md](claim-testnet-wbt.md). On Mainnet there is no faucet: WBT comes from WhiteBIT or another holder, see https://docs.whitechain.io/learn/general/get-wbt

Full connect-wallet walkthrough: https://docs.whitechain.io/learn/get-started/connect-wallet

## Using viem

```ts
import { defineChain } from 'viem'

export const whitechainSepolia = defineChain({
  id: 1874,
  name: 'Whitechain Sepolia',
  nativeCurrency: { name: 'WBT', symbol: 'WBT', decimals: 18 },
  rpcUrls: { default: { http: ['https://rpc.testnet.whitechain.io'] } },
  blockExplorers: { default: { name: 'Blockscout', url: 'https://explorer.testnet.whitechain.io' } },
})

export const whitechain = defineChain({
  id: 1875,
  name: 'Whitechain',
  nativeCurrency: { name: 'WBT', symbol: 'WBT', decimals: 18 },
  rpcUrls: { default: { http: ['https://rpc.whitechain.io'] } },
  blockExplorers: { default: { name: 'Blockscout', url: 'https://explorer.whitechain.io' } },
})
```

To fall back to the Ankr endpoint when the Whitechain RPC fails or returns `429`, use a `fallback` transport:

```ts
import { createPublicClient, fallback, http } from 'viem'

export const client = createPublicClient({
  chain: whitechainSepolia,
  transport: fallback([
    http('https://rpc.testnet.whitechain.io'),
    http('https://rpc.ankr.com/whitechain_testnet'),
  ]),
})
```

For Mainnet, use `chain: whitechain` with `https://rpc.whitechain.io` and `https://rpc.ankr.com/whitechain`.

## Error handling

| Error | Cause | Action |
| --- | --- | --- |
| Wallet shows wrong chain ID after adding network | Typo in Chain ID field | Re-check against the table above — `1874` for Sepolia, `1875` for Mainnet, not `1337` or another network's ID. |
| RPC times out or refuses connection | Public RPC rate limit or outage | Retry after a short delay, check https://status.whitechain.io/ for current uptime, or switch to the Ankr endpoint for the same network. |
| `429 Too Many Requests` response | Rate limit exceeded (50 rps sustained / burst 500 per IP) | Respect the `Retry-After: 1` header before retrying — each HTTP request counts against the limit, not each JSON-RPC call inside a batch. For more throughput, spread reads across the Whitechain and Ankr endpoints. |
| Transaction signed but never appears on the chosen network | Wallet was still on a different chain when signing | Re-confirm the active network in the wallet before retrying. |
| Transaction fails on Mainnet but passed on Sepolia | It needs more than 16,777,216 gas, the Mainnet per-transaction cap | Split the work into smaller transactions. |
