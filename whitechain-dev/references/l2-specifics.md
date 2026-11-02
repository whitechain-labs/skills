# Whitechain as an OP Stack L2

Whitechain is an OP Stack Layer 2 settling on Ethereum, running in Custom Gas Token mode with WBT
as the native token. This file covers what a contract or dapp developer needs beyond "it's EVM":
fees, finality, predeploys and the canonical bridge. Values are for Whitechain Sepolia unless a
row says otherwise.

Source of truth: [docs.whitechain.io](https://docs.whitechain.io/llms.txt). If a value here
disagrees with the docs or with the chain, trust the chain and report the mismatch.

## Network facts

| Property | Whitechain Sepolia | Whitechain Mainnet |
| --- | --- | --- |
| Chain ID | `1874` | `1875` |
| L1 | Ethereum Sepolia (`11155111`) | Ethereum |
| Block time | 1 s | 1 s |
| EVM level | Prague (OP Stack through Jovian) | Prague + Osaka subset (OP Stack through Karst: `CLZ`, EIP-7825 gas cap) |
| Per-transaction gas cap | none | `16,777,216` |
| Highest safe `evm_version` | `prague` | `prague` |
| Native token | WBT (18 decimals), Custom Gas Token mode | WBT |
| Base fee floor | 5 gwei | 5 gwei |
| Withdrawal to L1 | ~1 h typical, up to ~3 h | ≥ 7 days |

Check what the chain itself reports:

```bash
cast call 0x4200000000000000000000000000000000000015 "isCustomGasToken()(bool)" --rpc-url https://rpc.testnet.whitechain.io   # true
cast call 0x420000000000000000000000000000000000000F "isIsthmus()(bool)" --rpc-url https://rpc.testnet.whitechain.io         # true
```

## Fees

A transaction pays two parts, both in WBT:

| Part | Depends on | Notes |
| --- | --- | --- |
| L2 execution fee | gas used × (base fee + priority fee) | EIP-1559, elasticity 6, denominator 100, never below 5 gwei |
| L1 data fee | compressed tx size, current L1 base fee and blob base fee | Charged on every transaction, including reverted ones; not limited by the gas limit |

The explorer may show only the L2 part, and `forge` gas reports always do, so a balance can drop
slightly more than the displayed fee. Estimate the L1 part from the `GasPriceOracle` predeploy:

```bash
cast call 0x420000000000000000000000000000000000000F "getL1FeeUpperBound(uint256)(uint256)" <unsigned-tx-bytes> \
  --rpc-url https://rpc.testnet.whitechain.io
```

In apps, use viem's OP Stack actions (`viem/op-stack`), which return both parts and track formula
changes across upgrades. Do not reimplement the formula.

## Transaction finality

| Stage | Meaning | Typical time | Block tag |
| --- | --- | --- | --- |
| Soft confirmation | In an L2 block, on the sequencer's word only | ~1 s | `latest` |
| Safe | Batch posted to and confirmed on Ethereum | up to ~30 min | `safe` |
| Finalized | Ethereum block carrying the batch is final; irreversible | ~13 min after safe | `finalized` |

A soft confirmation can be reordered or dropped before its batch lands on L1. Use it for UI
feedback only. Anything that releases value off-chain must wait for `safe` (low value) or
`finalized` (high value):

```bash
cast block finalized -f number --rpc-url https://rpc.testnet.whitechain.io
```

Flashblocks (sub-second preconfirmations) are not enabled on either network.

## Predeploys (L2, identical on every OP Stack chain)

| Contract | Address | Use |
| --- | --- | --- |
| `L2CrossDomainMessenger` | `0x4200000000000000000000000000000000000007` | Send/receive L1 ↔ L2 messages; `xDomainMessageSender()` |
| `L2StandardBridge` | `0x4200000000000000000000000000000000000010` | Withdraw ERC-20s to L1 (`bridgeERC20`) |
| `OptimismMintableERC20Factory` | `0x4200000000000000000000000000000000000012` | Create the L2 side of an L1 token |
| `L2ERC721Bridge` | `0x4200000000000000000000000000000000000014` | Withdraw L1-origin NFTs |
| `L1Block` | `0x4200000000000000000000000000000000000015` | Latest known L1 block number, timestamp, base fee; `isCustomGasToken()` |
| `OptimismMintableERC721Factory` | `0x4200000000000000000000000000000000000017` | Create the L2 side of an L1 NFT |
| `GasPriceOracle` | `0x420000000000000000000000000000000000000F` | L1 data fee estimation |

`block.number` inside a contract is the **L2** block number. Read the L1 block from `L1Block`
when logic depends on Ethereum time.

## Canonical bridge (Sepolia)

L1 contracts on Ethereum Sepolia — verify against the
[bridge docs](https://docs.whitechain.io/build/bridge/bridge-assets.md#contract-addresses) before
sending value, since they change if the testnet is redeployed:

| Contract | L1 address |
| --- | --- |
| `L1StandardBridge` | `0x0c50be539ab5d72d226038928f2eb25100899ded` |
| `OptimismPortal` | `0xff9b597b0781457ae6aa7256ca5ed5839bf7d0c3` |
| `L1CrossDomainMessengerProxy` | `0x547967f45d19dc4ca0a056ba525afe99bda18a88` |
| `L1Erc721BridgeProxy` | `0x1e419fc5575bbd6d352582e9d8073918d79f0c5b` |
| `DisputeGameFactoryProxy` | `0xfaa2faa8912c069c01abc169c33713c79027c833` |
| `OptimismMintableERC20FactoryProxy` | `0x5898f6c319842a379799d634c6ed2da960a42b9b` |

What can move:

| Asset | L1 → L2 | L2 → L1 |
| --- | --- | --- |
| ERC-20 | `approve`, then `L1StandardBridge.depositERC20` (~2 min). The L2 token must exist first | `L2StandardBridge.bridgeERC20`, then prove + finalize on L1 |
| ERC-721 (L1-origin only) | `L1Erc721BridgeProxy.bridgeERC721` | `L2ERC721Bridge.bridgeERC721`, then prove + finalize |
| Arbitrary message | `L1CrossDomainMessenger.sendMessage` (replayable) | `L2CrossDomainMessenger.sendMessage`, then prove + finalize |
| Native ETH | Not supported (`depositETH` reverts in CGT mode). On Sepolia, wrap to WETH and `depositERC20`; it arrives as bridged ETH `0x1c7E0793c69838D1da9D5b98B5893866B977bA67` (`remoteToken()` = Sepolia WETH, `bridge()` = `L2StandardBridge`). Not yet available on Mainnet | Withdraw bridged ETH like any ERC-20 |
| Native WBT | Not supported on the canonical bridge — use the [Portal bridge](https://docs.whitechain.io/build/bridge/portal-bridge.md) | Portal bridge |

To make an L1 ERC-20 bridgeable, call
`OptimismMintableERC20Factory.createOptimismMintableERC20WithDecimals(remoteToken, name, symbol, decimals)`
on L2, then confirm `remoteToken()` and `bridge()` on the new token before depositing.

Withdrawals are never automatic: after initiating on L2, the user must wait for a dispute game
covering their block (one every 30 min), prove on L1, wait the dispute delays, and finalize on L1.
Use viem OP Stack actions (`buildProveWithdrawal`, `proveWithdrawal`, `finalizeWithdrawal`); the
`@eth-optimism/sdk` package is deprecated. Proof generation needs an archive-capable L2 RPC.

This section is reference material for developers integrating the bridge into contracts or
apps. Executing bridge transfers on the user's behalf is out of scope for this skill — see
[SKILL.md](../SKILL.md).

## Error handling

| Error | Cause | Action |
| --- | --- | --- |
| `OptimismPortal_NotAllowedOnCGTMode()` | `depositETH` on a Custom Gas Token chain | On Sepolia, wrap to WETH and use `depositERC20`, or use the Portal bridge. |
| Deposit sent but L2 balance unchanged after 5+ min | No L2 token paired with the L1 token, or wrong `_l2Token` | Check the L2 token's `remoteToken()`; confirm `L1StandardBridge.deposits(l1Token, l2Token)` increased. |
| `proveWithdrawalTransaction` reverts | No dispute game yet covers the withdrawal's L2 block | Wait for the next game (every 30 min) and use its `gameIndex`. |
| `eth_getProof` fails during prove | Public RPC pruned the needed state | Use an archive node — see [run-node.md](run-node.md). |
| Fee higher than the explorer shows | L1 data fee not displayed | Expected; estimate via `GasPriceOracle`. |
