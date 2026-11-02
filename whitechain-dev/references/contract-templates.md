# Contract templates for Whitechain

Reference ERC-20 and ERC-721 contracts built on OpenZeppelin Contracts v5.4.0, plus the
`foundry.toml` that matches Whitechain's EVM level. Prefer these audited building blocks over
hand-written token logic.

Both contracts were compiled with the config below, unit-, fuzz- and fork-tested, then deployed
and verified on Whitechain Sepolia with the exact `forge create --verify` commands in step 5.
Measured deploy cost: ERC-20 979,308 gas (≈0.0049 WBT), ERC-721 1,081,270 gas (≈0.0054 WBT) at the
5 gwei base fee floor; the L1 data fee on Sepolia was negligible.

The official [whitechain-labs/templates](https://github.com/whitechain-labs/templates) repo ships
Foundry, Hardhat, Next.js and Vite starters around a `Storage` contract (and a `BoxV1`/`BoxV2`
upgradeable proxy example in Hardhat). Start from that repo when the user wants a full project
scaffold; use the contracts on this page when they want a token.

## Input validation

| Input | Rule |
| --- | --- |
| Token name | Must match `^[a-zA-Z0-9 ]{1,64}$` — quote it as a single shell argument |
| Token symbol | Must match `^[A-Z0-9]{1,11}$` |
| Initial supply | Integer in base units (`1000000` tokens at 18 decimals = `1000000000000000000000000`) — reject decimals and scientific notation |
| Recipient / owner | Must match `^0x[a-fA-F0-9]{40}$` and must not be the zero address |
| Base URI | Must start with `ipfs://`, `ar://` or `https://` and end with `/`; reject shell metacharacters |

If any input fails validation, stop and ask the user to correct it.

## 1. Project setup

```bash
forge init my-token
cd my-token
forge install OpenZeppelin/openzeppelin-contracts@v5.4.0
```

Pin the compiler and EVM version. `prague` is the highest EVM target that runs on both Whitechain
Mainnet and Whitechain Sepolia; the solc default may be newer, so never rely on it. Verification
fails if the compiler version at verify time differs from the one used at deploy.

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.30"
evm_version = "prague"
optimizer = true
optimizer_runs = 200
remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/"]

[fuzz]
runs = 1000

[rpc_endpoints]
whitechain_sepolia = "https://rpc.testnet.whitechain.io"
```

## 2. ERC-20 — fixed supply

The whole supply is minted once to `recipient`. There is no owner and no mint function, so the
supply can never change. `ERC20Permit` adds gasless approvals (EIP-2612).

`src/WhitechainToken.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Permit} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

/// @notice Fixed-supply ERC-20. The whole supply is minted once to `recipient`; there is no owner and no mint function.
contract WhitechainToken is ERC20, ERC20Permit {
    constructor(string memory name_, string memory symbol_, uint256 initialSupply, address recipient)
        ERC20(name_, symbol_)
        ERC20Permit(name_)
    {
        _mint(recipient, initialSupply);
    }
}
```

If the user needs a mintable token, add `Ownable` and an `onlyOwner` `mint` function, and point
them to [security.md](security.md) for owner-key handling. Do not add minting by default.

## 3. ERC-721 — owner-minted NFT

Sequential token IDs from `0`, one shared base URI (`tokenURI(n)` = `baseURI + n`), minting
restricted to `initialOwner`.

`src/WhitechainNFT.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/// @notice Owner-minted ERC-721 with sequential token IDs and a shared base URI.
contract WhitechainNFT is ERC721, Ownable {
    uint256 private _nextTokenId;
    string private _baseTokenURI;

    constructor(string memory name_, string memory symbol_, string memory baseURI_, address initialOwner)
        ERC721(name_, symbol_)
        Ownable(initialOwner)
    {
        _baseTokenURI = baseURI_;
    }

    function safeMint(address to) external onlyOwner returns (uint256 tokenId) {
        tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
    }

    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }
}
```

## 4. Test before deploying

Copy the test files from [testing.md](testing.md), then:

```bash
forge build
forge test
forge test --match-contract Fork --fork-url whitechain_sepolia
```

## 5. Deploy and verify

Follow [deploy-contract.md](deploy-contract.md). Both templates take constructor arguments, so
`--constructor-args` goes **last**:

```bash
forge create src/WhitechainToken.sol:WhitechainToken \
  --rpc-url https://rpc.testnet.whitechain.io \
  --account <account-name> \
  --broadcast \
  --verify \
  --verifier blockscout \
  --verifier-url https://explorer.testnet.whitechain.io/api/ \
  --constructor-args "My Token" MTK 1000000000000000000000000 <recipient-address>
```

```bash
forge create src/WhitechainNFT.sol:WhitechainNFT \
  --rpc-url https://rpc.testnet.whitechain.io \
  --account <account-name> \
  --broadcast \
  --verify \
  --verifier blockscout \
  --verifier-url https://explorer.testnet.whitechain.io/api/ \
  --constructor-args "My NFT" MNFT ipfs://<cid>/ <owner-address>
```

After deploy, read the state back before reporting success:

```bash
cast call <token> "totalSupply()(uint256)" --rpc-url https://rpc.testnet.whitechain.io
cast call <token> "balanceOf(address)(uint256)" <recipient-address> --rpc-url https://rpc.testnet.whitechain.io
```

## Error handling

| Error | Cause | Action |
| --- | --- | --- |
| `Source "@openzeppelin/contracts/..." not found` | OpenZeppelin not installed or remapping missing | Run the `forge install` step and check the `remappings` line in `foundry.toml`. |
| `invalid opcode` / deploy reverts with no reason | Often an EVM target newer than `prague` | Check `evm_version = "prague"` in `foundry.toml` and rebuild; if it persists, debug the constructor. |
| Verification: bytecode mismatch | `solc`, optimizer, or `evm_version` differ from the deployed build | Rebuild with the exact `foundry.toml` used at deploy and re-verify — see [verify-contract.md](verify-contract.md). |
| `Constructor argument count mismatch` | `--constructor-args` not last | Move it to the end of the command. |
| `ERC721InvalidReceiver` on `safeMint` | Recipient is a contract without `onERC721Received` | Mint to an EOA or to a contract that implements `IERC721Receiver`. |
