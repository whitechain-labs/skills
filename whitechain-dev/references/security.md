# Contract security on Whitechain

Whitechain is fully EVM-compatible, so every Ethereum vulnerability class applies unchanged. On
top of those, an OP Stack L2 with a custom gas token adds its own assumptions. This file covers
both, plus a checklist to run before any contract holds real value.

Key hygiene (keystores, never pasting private keys) is covered in
[deploy-contract.md](deploy-contract.md#security) and is not repeated here.

## Whitechain and OP Stack specifics

| Risk | What goes wrong | Mitigation |
| --- | --- | --- |
| Native token is WBT, not ETH | Contracts that price in "ETH", hardcode ETH oracles, or call `L1StandardBridge.depositETH` (reverts with `OptimismPortal_NotAllowedOnCGTMode()`) | Treat `msg.value` as WBT. Bridged ETH is an ERC-20 on L2 — on Sepolia `0x1c7E0793c69838D1da9D5b98B5893866B977bA67`; check its `remoteToken()` and `bridge()` before trusting it. |
| Trusting soft confirmations | A single sequencer orders transactions; a soft-confirmed transaction can be reordered or dropped before its batch reaches Ethereum | Off-chain services that release value (payments, withdrawals, mints from another chain) wait for the `safe` or `finalized` block tag. See [l2-specifics.md](l2-specifics.md#transaction-finality). |
| Block-number timing | 1-second blocks make `block.number`-based delays ~12× shorter than the author expects from Ethereum | Use `block.timestamp` for time windows. |
| L1 → L2 message sender | Messages sent through the messenger arrive with `msg.sender` = `L2CrossDomainMessenger`, not the L1 caller. Deposit transactions sent by an L1 *contract* directly to `OptimismPortal` arrive with an aliased sender (`L1 address + 0x1111000000000000000000000000000000001111`, standard OP Stack [address aliasing](https://docs.optimism.io/stack/differences#address-aliasing)) | Authenticate L1 callers through `L2CrossDomainMessenger` (`0x4200000000000000000000000000000000000007`): check `msg.sender == messenger` **and** `messenger.xDomainMessageSender() == expectedL1Sender`. |
| Calling `OptimismPortal.depositTransaction` directly | No replay if the L2 call runs out of gas — funds or messages can be lost | Use `L1CrossDomainMessenger.sendMessage`, which supports replay. |
| Bridging non-standard tokens | Fee-on-transfer, rebasing, blocklist or pausable tokens can make bridge finalization revert permanently | Do not pair them through the standard bridge; use a custom bridge. |
| Sepolia vs Mainnet gas cap | Sepolia has no per-transaction cap; Mainnet caps at `16,777,216` gas (EIP-7825). A loop that works on testnet can never execute on Mainnet once its gas exceeds the cap | Bound every loop; paginate batch operations. |
| Admin keys on an EOA | One leaked key controls ownership, upgrades and mints | Hold ownership in a [Safe](https://docs.whitechain.io/build/wallet/wallets.md#safe) multisig (`app.safe.global`, network prefix `wch-sepolia:`). Safe is live on Whitechain Sepolia; check the docs for Mainnet availability before planning a Mainnet handover. |
| Sequencer downtime | No new L2 blocks while the operator's sequencer is down | Time-sensitive logic (auctions, liquidations, oracle staleness) needs grace periods; forced inclusion via `OptimismPortal` takes up to ~12 hours. |

## General EVM vulnerability classes

| Class | Pattern to look for | Fix |
| --- | --- | --- |
| Reentrancy | External call (`call`, token transfer, `safeMint` → `onERC721Received`) before state update | Checks-effects-interactions; OpenZeppelin `ReentrancyGuard` |
| Access control | Missing `onlyOwner` / role check on mint, pause, upgrade, withdraw | OpenZeppelin `Ownable` or `AccessControl`; test every privileged function with a non-privileged caller |
| Unchecked external calls | Ignoring the return value of `call` or of non-reverting ERC-20s | `SafeERC20`; check `(bool ok, ) = ...` |
| Oracle manipulation | Spot price from a single AMM pool used for pricing | TWAP or an external oracle with staleness checks |
| Signature replay | Signed messages without nonce, chain ID or contract address | EIP-712 with `block.chainid` (`1874` Sepolia, `1875` Mainnet) and per-signer nonces |
| Unbounded loops | Iterating over user-growable arrays | Pagination or pull patterns |
| Transaction-ordering dependence | Commit-less auctions, swaps without slippage limits — the sequencer decides ordering | Commit-reveal, user-supplied min-out and deadline |
| Upgradeable storage collisions | Reordering or inserting storage variables between versions | OpenZeppelin upgradeable contracts with storage gaps or ERC-7201 namespaces; verify with `@openzeppelin/upgrades-core`. Verify proxies per [verify-contract.md](verify-contract.md#proxy-contracts) |
| Uninitialized proxies | Implementation contract left initializable | `_disableInitializers()` in the implementation constructor |

## Static analysis tools

Run at least one before deploying anything beyond a throwaway test:

| Tool | Command |
| --- | --- |
| Slither | `slither .` (`pip install slither-analyzer`) |
| Aderyn | `aderyn .` (`cargo install aderyn` or `npm i -g @cyfrin/aderyn`) |
| Foundry lint | `forge lint` |
| Coverage | `forge coverage` — aim for every privileged and revert branch covered |

Report tool findings to the user with severity; do not silently "fix" findings that change
contract behavior.

## Pre-mainnet checklist

The skill targets Whitechain Sepolia. Before the user moves a contract to Whitechain Mainnet,
walk them through:

- [ ] All tests in [testing.md](testing.md) pass, including fork tests
- [ ] Static analysis run; every high/medium finding resolved or explicitly accepted
- [ ] No transaction path exceeds the Mainnet 16,777,216 gas cap
- [ ] `solc` and `evm_version = "prague"` pinned; source verified on the explorer
- [ ] Ownership / admin roles held by a Safe multisig, not an EOA (confirm Safe is deployed on Mainnet first)
- [ ] No logic assumes the native token is ETH
- [ ] Off-chain services wait for `safe`/`finalized` before releasing value
- [ ] Withdrawal flows account for the 7-day Mainnet dispute period (≈1 hour on Sepolia)
- [ ] External audit for any contract that will hold meaningful value

## Error handling

| Error | Cause | Action |
| --- | --- | --- |
| `OptimismPortal_NotAllowedOnCGTMode()` | ETH deposit attempted on a Custom Gas Token chain | On Sepolia, wrap to WETH on L1 and deposit as ERC-20 (not yet available on Mainnet), or use the Portal bridge. |
| L2 call from L1 fails `onlyOwner` | `msg.sender` is the messenger (or the aliased L1 contract address), not the L1 owner | Authenticate via `xDomainMessageSender()` as described above. |
| Slither fails to compile | Remappings not picked up | Run `forge build` first; Slither reads Foundry's output. |
