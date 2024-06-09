# Tokensoft - Findings Report
Prepared by: Roberto Delgado Ferrezuelo

# Table of contents

- [Findings](#findings)
  - [Medium](#medium)
    - [\[M-1\] Empty input bytes array in `PerAddressContinuousVestingMerkle::claim` will result in a wrong claim execution](#m-1-empty-input-bytes-array-in-peraddresscontinuousvestingmerkleclaim-will-result-in-a-wrong-claim-execution)

# Findings

## Medium

### [M-1] Empty input bytes array in `PerAddressContinuousVestingMerkle::claim` will result in a wrong claim execution

## Impact
In the new per address implementation, the start, end and cliff periods are assigned individually instead of registering a unique record for all addresses on storage. Therefore, when calling `executeClaim` in the snippet below, the claimable amount will be calculated using a null bytes object instead of the encoded version of the correct reference periods. This will ultimately affect the core functionality of the vesting schedule and result in an incorrect distribution of tokens.

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

## Tool used

Manual Review

## Recommendation

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