# Testing contracts for Whitechain

Unit, fuzz and fork testing with Foundry, plus the places where Whitechain behaves differently
from Ethereum mainnet and from a local Anvil chain. Run every layer before
[deploy-contract.md](deploy-contract.md) — a mined deployment cannot be undone.

## Testing layers

| Layer | Command | Catches |
| --- | --- | --- |
| Unit | `forge test` | Logic errors, access control, revert paths |
| Fuzz | `forge test` (any `testFuzz_*` with parameters) | Edge values the author did not think of |
| Fork | `forge test --match-contract Fork --fork-url whitechain_sepolia` | Assumptions about live chain state, predeploys and deployed protocols |
| Coverage | `forge coverage` | Untested branches |
| Gas | `forge test --gas-report` | Functions that will be expensive for users |

`whitechain_sepolia` resolves through the `[rpc_endpoints]` table in `foundry.toml` (see
[contract-templates.md](contract-templates.md#1-project-setup)).

## Whitechain-specific differences to test for

| Difference | Why it matters in tests |
| --- | --- |
| Native token is **WBT**, not ETH (OP Stack Custom Gas Token mode) | `msg.value`, `address.balance` and `payable` amounts are WBT. Code or tests that assume ETH pricing, or that call `L1StandardBridge.depositETH`, are wrong. |
| 1-second L2 blocks | `block.number` advances ~12× faster than on Ethereum. Never derive time from block counts — use `block.timestamp`, and test with `vm.warp`, not `vm.roll` alone. |
| L1 data fee on every transaction | Gas reports show only L2 execution gas. The user also pays an L1 data fee, charged even when the transaction reverts. Estimate it with the `GasPriceOracle` predeploy (below). |
| Sepolia has **no per-transaction gas cap**; Mainnet caps at `16,777,216` (EIP-7825) | A transaction that succeeds on Sepolia can fail on Mainnet. Check the max column of `forge test --gas-report` for batch functions and keep every call under the cap. |
| Mainnet has the Osaka `CLZ` opcode (OP Stack Karst); Sepolia does not | Compile with `evm_version = "prague"` so the same bytecode runs on both. |
| Soft confirmations can be reordered before the batch reaches Ethereum | Off-chain services (indexers, payment listeners) should wait for `safe` or `finalized` — test with `cast block safe` / `cast block finalized`. See [l2-specifics.md](l2-specifics.md). |

## Unit and fuzz tests

Matches the ERC-20 template in [contract-templates.md](contract-templates.md).

`test/WhitechainToken.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {Test} from "forge-std/Test.sol";
import {IERC20Errors} from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";
import {WhitechainToken} from "../src/WhitechainToken.sol";

contract WhitechainTokenTest is Test {
    WhitechainToken token;
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    uint256 constant SUPPLY = 1_000_000 ether;

    function setUp() public {
        token = new WhitechainToken("Whitechain Token", "WCT", SUPPLY, alice);
    }

    function test_InitialSupplyGoesToRecipient() public view {
        assertEq(token.totalSupply(), SUPPLY);
        assertEq(token.balanceOf(alice), SUPPLY);
        assertEq(token.decimals(), 18);
    }

    function test_RevertWhen_TransferExceedsBalance() public {
        vm.prank(bob);
        vm.expectRevert(abi.encodeWithSelector(IERC20Errors.ERC20InsufficientBalance.selector, bob, 0, 1));
        token.transfer(alice, 1);
    }

    function testFuzz_TransferConservesSupply(uint256 amount) public {
        amount = bound(amount, 0, SUPPLY);
        vm.prank(alice);
        token.transfer(bob, amount);
        assertEq(token.balanceOf(alice) + token.balanceOf(bob), SUPPLY);
    }

    function testFuzz_TransferFromSpendsAllowance(uint256 allowance, uint256 amount) public {
        allowance = bound(allowance, 0, SUPPLY);
        amount = bound(amount, 0, allowance);
        vm.prank(alice);
        token.approve(bob, allowance);
        vm.prank(bob);
        token.transferFrom(alice, bob, amount);
        assertEq(token.allowance(alice, bob), allowance - amount);
        assertEq(token.balanceOf(bob), amount);
    }
}
```

`test/WhitechainNFT.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {Test} from "forge-std/Test.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {WhitechainNFT} from "../src/WhitechainNFT.sol";

contract WhitechainNFTTest is Test {
    WhitechainNFT nft;
    address owner = makeAddr("owner");
    address alice = makeAddr("alice");

    function setUp() public {
        nft = new WhitechainNFT("Whitechain NFT", "WCNFT", "ipfs://example/", owner);
    }

    function test_OwnerMintsSequentialIds() public {
        vm.startPrank(owner);
        assertEq(nft.safeMint(alice), 0);
        assertEq(nft.safeMint(alice), 1);
        vm.stopPrank();
        assertEq(nft.ownerOf(1), alice);
        assertEq(nft.tokenURI(1), "ipfs://example/1");
    }

    function test_RevertWhen_NonOwnerMints() public {
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, alice));
        nft.safeMint(alice);
    }
}
```

Conventions:

- Name tests `test_*`, `testFuzz_*`, `test_RevertWhen_*` so intent is visible in the report
- Use `bound()` instead of `vm.assume()` for numeric ranges — `assume` discards runs and can exhaust the fuzzer
- Assert the exact custom error with `vm.expectRevert(abi.encodeWithSelector(...))`, not a bare `vm.expectRevert()`

## Fork tests against live Whitechain Sepolia

Fork tests run your contracts against real chain state, including the OP Stack predeploys. The
suite skips itself unless it is running on chain ID `1874`, so a plain `forge test` stays offline.

`test/WhitechainFork.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {Test} from "forge-std/Test.sol";
import {WhitechainToken} from "../src/WhitechainToken.sol";

interface IL1Block {
    function isCustomGasToken() external view returns (bool);
    function number() external view returns (uint64);
}

interface IGasPriceOracle {
    function getL1FeeUpperBound(uint256 unsignedTxSize) external view returns (uint256);
}

/// @notice Runs against live Whitechain Sepolia state: forge test --match-contract Fork --fork-url whitechain_sepolia
contract WhitechainForkTest is Test {
    IL1Block constant L1_BLOCK = IL1Block(0x4200000000000000000000000000000000000015);
    IGasPriceOracle constant GAS_PRICE_ORACLE = IGasPriceOracle(0x420000000000000000000000000000000000000F);

    function setUp() public {
        vm.skip(block.chainid != 1874);
    }

    function test_PredeploysReflectWhitechain() public view {
        assertTrue(L1_BLOCK.isCustomGasToken(), "native token is WBT, not ETH");
        assertGt(L1_BLOCK.number(), 0, "L1 origin block");
        assertGt(GAS_PRICE_ORACLE.getL1FeeUpperBound(200), 0, "L1 data fee is charged");
    }

    function test_DeployOnForkedState() public {
        WhitechainToken token = new WhitechainToken("Fork Token", "FORK", 1 ether, address(this));
        assertEq(token.balanceOf(address(this)), 1 ether);
    }
}
```

```bash
forge test --match-contract Fork --fork-url whitechain_sepolia
```

Pin a block for reproducible results: add `--fork-block-number <n>`. The public RPC is
rate-limited (50 rps per IP), so keep fork suites small or run your own node — see
[run-node.md](run-node.md).

## Pre-flight against the live node

Before handing the deploy command to the user, confirm the real node accepts the bytecode. This
sends nothing and needs no key:

```bash
cast estimate --from <deployer-address> --rpc-url https://rpc.testnet.whitechain.io \
  --create $(forge inspect <ContractName> bytecode)<abi-encoded-constructor-args-without-0x>
```

Encode constructor arguments with `cast abi-encode "f(string,string,uint256,address)" ...`.
A gas number means the node executed the init code. An `invalid opcode` error usually means the
EVM target is newer than `prague`.

Estimate the L1 data fee the user will pay on top of L2 gas:

```bash
cast call 0x420000000000000000000000000000000000000F \
  "getL1FeeUpperBound(uint256)(uint256)" <unsigned-tx-size-in-bytes> \
  --rpc-url https://rpc.testnet.whitechain.io
```

Do not reimplement the L1 fee formula — it changes with OP Stack upgrades and the predeploy is kept in step.

## Error handling

| Error | Cause | Action |
| --- | --- | --- |
| Fork suite reports `skipped` | Ran without `--fork-url` | Expected — rerun with `--fork-url whitechain_sepolia`. |
| `429 Too Many Requests` during fork tests | Public RPC rate limit | Add `--fork-block-number` (enables caching), reduce fuzz runs in fork suites, or use a self-hosted node. |
| Fuzz test fails with ``The `vm.assume` cheatcode rejected too many inputs`` | Overuse of `vm.assume` | Replace with `bound()`. |
| Passes on Sepolia, fails on Mainnet with gas limit error | Transaction above the Mainnet 16,777,216 per-tx cap | Split the work into smaller transactions. |
