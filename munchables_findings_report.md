# Munchables - Findings Report
Prepared by: Roberto Delgado Ferrezuelo

# Table of contents

- [Findings](#findings)
  - [Medium](#medium)
    - [\[M-1\] Missing disapproval check in `LockManager.sol::approveUSDPrice` allows simultaneous approval and disapproval of a price proposal](#m-1-missing-disapproval-check-in-lockmanagersolapproveusdprice-allows-simultaneous-approval-and-disapproval-of-a-price-proposal)

# Findings

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
