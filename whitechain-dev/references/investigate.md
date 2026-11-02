# Investigate addresses, transactions and the network

Read-only lookups on Whitechain Sepolia: balances, transaction status, revert reasons, fee
breakdowns, finality, contract details and network health. Nothing here signs or sends a
transaction, so no keystore and no confirmation step are needed.

Every command was run against `https://rpc.testnet.whitechain.io` and the Blockscout API. For
large-scale reads (indexing, metrics, pagination) use the Blockscout API directly; this file covers
one-off questions.

## Input validation

| Input | Rule |
| --- | --- |
| Address | Must match `^0x[a-fA-F0-9]{40}$` |
| Transaction hash | Must match `^0x[a-fA-F0-9]{64}$` |
| Block number | Decimal integer, or one of `latest`, `safe`, `finalized` |

If any input fails validation, stop and ask the user to correct it.

All examples use:

```bash
R=https://rpc.testnet.whitechain.io
B=https://explorer.testnet.whitechain.io/api/v2
```

## Balances

```bash
cast balance <address> --ether --rpc-url $R                                       # native WBT (not ETH)
cast call <token> "balanceOf(address)(uint256)" <address> --rpc-url $R            # one ERC-20, in base units
cast call <token> "decimals()(uint8)" --rpc-url $R                                # never assume 18
curl -s "$B/addresses/<address>/token-balances" | jq -c '.[] | {symbol: .token.symbol, value, decimals: .token.decimals}'   # all tokens
```

## Transaction status

Always pass `--async`. Without it, `cast receipt` waits indefinitely for a hash that does not
exist instead of returning an error.

```bash
cast receipt --async <tx-hash> status --rpc-url $R        # "1 (success)" or "0 (failed)"
cast receipt --async <tx-hash> --rpc-url $R               # full receipt
cast tx <tx-hash> --rpc-url $R                            # from, to, value, nonce, input
```

`Error: tx not found` means the hash is wrong, on another network, or not yet seen by the node.
For a contract deployment, the receipt's `contractAddress` holds the new address.

## Why a transaction failed

A failed receipt does not carry the revert reason. Replay the call against the state of the block
before it:

```bash
TX=<tx-hash>
BN=$(cast receipt --async $TX blockNumber --rpc-url $R)
T=$(cast tx $TX --json --rpc-url $R)
cast call --from $(echo $T | jq -r .from) $(echo $T | jq -r .to) $(echo $T | jq -r .input) \
  --value $(cast to-dec $(echo $T | jq -r .value)) --block $((BN - 1)) --rpc-url $R
```

The error returns revert `data`. Decode a custom error with:

```bash
cast decode-error <revert-data>
```

The replay runs at the end of the previous block, so a transaction that depended on an earlier
transaction in the same block may replay differently. Say so when reporting the reason.

## Fee breakdown

Every Whitechain receipt carries the L1 data fee in a separate `l1Fee` field, and the explorer may
show only the L2 part. `cast receipt <hash> l1Fee` does not work, so read the JSON:

```bash
J=$(cast receipt --async <tx-hash> --json --rpc-url $R)
G=$(cast to-dec $(echo $J | jq -r .gasUsed))
P=$(cast to-dec $(echo $J | jq -r .effectiveGasPrice))
L1=$(cast to-dec $(echo $J | jq -r .l1Fee))
echo "L2: $(cast from-wei $((G * P))) WBT  L1: $(cast from-wei $L1) WBT  total: $(cast from-wei $((G * P + L1))) WBT"
```

Report all three numbers. A reverted transaction pays both parts too. See
[l2-specifics.md](l2-specifics.md#fees) for how each part is set.

## Finality

Being in a block is not final. Compare the transaction's block with the `safe` and `finalized`
heads:

```bash
BN=$(cast receipt --async <tx-hash> blockNumber --rpc-url $R)
cast block safe -f number --rpc-url $R
cast block finalized -f number --rpc-url $R
```

| Condition | Report as |
| --- | --- |
| `BN` > safe | Soft-confirmed only — the sequencer's word; batch not yet on Ethereum (up to ~30 min) |
| safe ≥ `BN` > finalized | Safe — batch confirmed on Ethereum, not yet final (~13 min more) |
| `BN` ≤ finalized | Finalized — irreversible |

Details in [l2-specifics.md](l2-specifics.md#transaction-finality).

## Contract details

```bash
curl -s "$B/smart-contracts/<address>" | jq '{name, is_verified, compiler_version, evm_version}'
curl -s "$B/tokens/<address>" | jq '{name, symbol, decimals, total_supply, holders_count}'
cast call <address> "owner()(address)" --rpc-url $R      # if the contract is Ownable
cast codesize <address> --rpc-url $R                     # 0 means no contract at this address
```

An unverified contract returns `null` for `name` and `is_verified` — offer
[verify-contract.md](verify-contract.md) if the user owns it. An address with no contract returns
`{"message":"Not found"}` (HTTP 404). `owner()` reverts with `execution reverted` when the contract
is not `Ownable`.

## Network health

```bash
cast chain-id --rpc-url $R                                               # must be 1874
cast block-number --rpc-url $R
echo "$(( $(date +%s) - $(cast block latest -f timestamp --rpc-url $R) ))s since last block"
```

Blocks arrive every second, so expect a gap near `0s` (a small negative value is local clock skew).
A gap of more than a few seconds means the sequencer may be stalled —
check [status.whitechain.io](https://status.whitechain.io/).

## Error handling

| Error | Cause | Action |
| --- | --- | --- |
| `cast receipt` hangs | `--async` missing and the hash is unknown | Re-run with `--async`. |
| `Error: tx not found` | Wrong hash, wrong network, or not yet propagated | Check the hash and chain ID; retry after a few seconds. |
| `invalid receipt field: l1Fee` | `cast receipt` field shortcut does not know OP Stack fields | Use `--json` and `jq` as shown above. |
| Replay succeeds although the receipt shows failure | State changed by an earlier transaction in the same block | Report the status from the receipt and say the reason could not be reproduced. |
| `429 Too Many Requests` | Public RPC limit (50 rps per IP) | Back off for the `Retry-After` interval. |
