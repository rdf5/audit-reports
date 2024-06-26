# Tokensoft - Findings Report
Prepared by: Roberto Delgado Ferrezuelo

# Table of contents

- [Findings](#findings)
  - [Medium](#medium)
    - [\[M-1\] Empty input bytes array in `PerAddressContinuousVestingMerkle::claim` will result in a wrong claim execution](#m-1-empty-input-bytes-array-in-peraddresscontinuousvestingmerkleclaim-will-result-in-a-wrong-claim-execution)

# Findings

## Medium

### [M-1] Empty input bytes array in `PerAddressContinuousVestingMerkle::claim` will result in a wrong claim execution

#### Summary
An incorrect function call associated to the vesting schedule will result in an incorrect distribution of tokens.

#### Vulnerability Detail
In the new per address implementation, the start, end and cliff periods are assigned individually instead of registering a unique record for all addresses on storage. Therefore, when calling _executeClaim in the snippet below, which in turns calls PerAddressContinuousVesting::getVestedFraction, the function will revert.

#### Impact

This will ultimately affect the core functionality of the vesting schedule and result in a loss of users funds. See PoC associated:

<details>
	
<summary>Code</summary>

Place this in packages/hardhat/test/foundry.

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.21;

import "forge-std/Test.sol";
import { console } from "forge-std/console.sol";
import { PerAddressContinuousVestingMerkle } from "../../contracts/claim/PerAddressContinuousVestingMerkle.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract ContinuousVestingPerAddress is Test {
	PerAddressContinuousVestingMerkle peraddresscontinuousvestingmerkle;
	function setUp() public {
		IERC20 token = IERC20(address(1));
		uint256 _total = 1 ether;
		string memory _uri = "string";
		bytes32 _merkleRoot = 0xf58b1fa04cf42dbf6d5ee77876c2936da42f408dd84c0e9ec6d0caacff19b481;
		uint160 _maxDelayTime = 1 days;
		uint256 _voteFactor = 1 ether;
		peraddresscontinuousvestingmerkle = new PerAddressContinuousVestingMerkle(
			token,
			_total,
			_uri,
			_voteFactor,
			_merkleRoot,
			_maxDelayTime
		);
	}

	// leaf = 0xc3c4a97077083128df50b493a1bd6dcf4ae081ed36a9e823f04501c9b4afa4dd
	// proof = 0x46c4c3c710f690fb060e405ff8c93f37c87532049f4961a3bce0338eb57d5051
	// root =  0xf58b1fa04cf42dbf6d5ee77876c2936da42f408dd84c0e9ec6d0caacff19b481

	function test_erroneousInputData() public {
		uint256 index = 1;
		address beneficiary = address(1);
		uint256 totalAmount = 1 ether;
		uint256 start = block.timestamp;
		uint256 cliff = block.timestamp + 1 days / 2;
		uint256 end = block.timestamp + 1 days;
		bytes32[] memory merkleProof = new bytes32[](1);
		merkleProof[
			0
		] = 0x46c4c3c710f690fb060e405ff8c93f37c87532049f4961a3bce0338eb57d5051;

		bytes32 leaf = keccak256(
			abi.encodePacked(index, beneficiary, totalAmount, start, cliff, end)
		);

		console.logBytes32(leaf);

		// bytes memory correctedInput = abi.encode(start, cliff, end);

		vm.expectRevert();

		uint vestedFraction = peraddresscontinuousvestingmerkle
			.getVestedFraction(address(1), end, new bytes(0));
		console.log(vestedFraction);
	}
}
```

</details>

#### Code Snippet
LOCs: https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/67edd899612619b0acefdcb0783ef7a8a75caeac/contracts/packages/hardhat/contracts/claim/PerAddressContinuousVestingMerkle.sol#L67

```javascript
	function claim(
        ...
		uint256 claimedAmount = _executeClaim(
			beneficiary,
			totalAmount,
@>		new bytes(0) // @audit should be abi.encode(start,cliff,end)
		);
		...
	}
```

#### Tool used
Manual Review

#### Recommendation
Include the encoded data with the function call:

```diff
	function claim(
        ...
		uint256 claimedAmount = _executeClaim(
			beneficiary,
			totalAmount,
--		new bytes(0) // @audit should be abi.encode(start,cliff,end)
++      abi.encode(start,cliff,end)
		);
		...
	}
```
