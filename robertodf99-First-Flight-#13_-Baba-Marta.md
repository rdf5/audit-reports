# First Flight #13: Baba Marta - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Function `MartenitsaMarketplace::collectReward` fails to update the amount of collected rewards properly.](#H-01)
    - ### [H-02. Missing access control in `MartenitsaToken::updateCountMartenitsaTokensOwner`](#H-02)
- ## Medium Risk Findings
    - ### [M-01. `MartenitsaVoting::voteCounts` and `MartenitsaVoting::hasVoted` are not reset after the winner is announced](#M-01)
    - ### [M-02. The array `MartenitsaEvent::participants` is not emptied after stopping the event and it can become too expensive to finish it later on.](#M-02)
    - ### [M-03. Users can directly transfer martenitsas via ´MartenitsaToken::transferFrom´ and their balance will not be updated.](#M-03)
- ## Low Risk Findings
    - ### [L-01. Users cannot cancel existing listings](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #13

### Dates: Apr 11th, 2024 - Apr 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 3
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Function `MartenitsaMarketplace::collectReward` fails to update the amount of collected rewards properly.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L106

## Summary
When users acquire enough martenitsa tokens they can claim the corresponding amount of health tokens by calling `MartenitsaMarketplace::collectReward`. This function updates the amount being claimed and it is discounted in future redemptions, however the way in which this value is updated is not correct.
## Vulnerability Details
`MartenitsaMarketplace::_collectedRewards` stores how many health tokens a user has claimed, this value is updated after each function call as indicated below:

```javascript
    function collectReward() external {
        require(
            !martenitsaToken.isProducer(msg.sender),
            "You are producer and not eligible for a reward!"
        );
        uint256 count = martenitsaToken.getCountMartenitsaTokensOwner(
            msg.sender
        );
        uint256 amountRewards = (count / requiredMartenitsaTokens) -
            _collectedRewards[msg.sender];
        if (amountRewards > 0) {
@>            _collectedRewards[msg.sender] = amountRewards;
            healthToken.distributeHealthToken(msg.sender, amountRewards);
        }
    }
```

The number of health tokens to which the user is eligible is calculated as the substraction of the coefficient of the total number of owned martenitsas divided by the minimum required to claim one health token and the tokens claimed in the previous call. Since this update mechanism only takes into account the previous snapshot instead of the full history of claims, every health token acquired before the claim `n-1` where `n` represents the current number of claims, is rewarded for free to the user.

## Impact
Users have access to unlimited health tokens by just minting 6 martentitsas. The process can be described into more detail as follows:
- `bob` acquires three martentisas.
- `bob` claims his health token. `MartenitsaMarketplace::_collectedRewards` is updated to 1.
- `bob` acquires another three martentitas.
- `bob` claims another health token. `MartenitsaMarketplace::_collectedRewards` is updated to 1, ignoring the previously claimed health token.
- Since `MartenitsaMarketplace::_collectedRewards` will only look at the previous snapshot, now the health token corresponding to claim `n-2` will be rewarded for free in the next claim. 
See PoC below.

<details>
<summary>Code</summary>

Place this in `MartenitsaMarketplace.t.sol`.
```javascript
    function test_userCanClaimFreeHealthTokens() public eligibleForReward {
        vm.prank(bob);
        marketplace.collectReward();
        vm.startPrank(chasy);
        for (uint256 i = 3; i < 6; i++) {
            martenitsaToken.createMartenitsa("bracelet");
            martenitsaToken.approve(address(marketplace), i);
            marketplace.makePresent(bob, i);
        }
        vm.stopPrank();
        vm.startPrank(bob);
        marketplace.collectReward();
        marketplace.collectReward(); // 1st health token is rewarded again
        assertEq(healthToken.balanceOf(bob), (2 + 1) * 10 ** 18);
    }
```
</details>


## Tools Used
Foundry and manual review.
## Recommendations
Implement a mechanism which compounds the previously claimed rewards instead of considering only the previous snapshot:
```diff
...
        if (amountRewards > 0) {
-            _collectedRewards[msg.sender] = amountRewards;
+            _collectedRewards[msg.sender] += amountRewards;
            healthToken.distributeHealthToken(msg.sender, amountRewards);
        }
...
```
## <a id='H-02'></a>H-02. Missing access control in `MartenitsaToken::updateCountMartenitsaTokensOwner`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaToken.sol#L62-L70

## Summary
The function `MartenitsaToken::updateCountMartenitsaTokensOwner` allows any user to adjust their martenitsa token balances, whether increasing or decreasing them.

## Vulnerability Details
The function  `MartenitsaToken::updateCountMartenitsaTokensOwner` is used to update the amount of martenitsas owned by users with non-producer role. These users are eligible to claim health tokens once they possess a minimum required amount of martenitsas by calling `MartenitsaMarketplace::collectReward`, and this function relies on `MartenitsaToken::getCountMartenitsaTokensOwner` which can be manipulated through `MartenitsaToken::updateCountMartenitsaTokensOwner`. Therefore, only the marketplace should have permission to call this last function, however, the current contract code overlooks this crucial check.

## Impact
Any arbitrary user can make use of this function to increase or decrease balances as they see fit.

One attack vector could be the increase in the own attacker's balance to later claim health tokens without actually owning the minimum amount of martenitsas required. The steps to perform such attack can be delineated as simple as:

- `attacker` calls three consecutive times to the function `MartenitsaToken::updateCountMartenitsaTokensOwner` passing his/her address and `add` as the function call parameters.
- Immediately after calls `MartenitsaMarketplace::collectReward`. As a result now the attacker owns 1 health token and has acquired 0 martenitsas.

See PoC below:

<details>
<summary>Code</summary>

Place this in `MartenitsaToken.t.sol`.
```javascript
    function test_attackerCanIncreaseBalance() public {
        address attacker = makeAddr("attacker");
        vm.startPrank(attacker);
        for (uint256 i; i < 3; i++) {
            martenitsaToken.updateCountMartenitsaTokensOwner(attacker, "add");
        }
        marketplace.collectReward();
        assertEq(healthToken.balanceOf(attacker), 10 ** 18);
    }
```
</details>

A second attack vector would imply the attacker reducing other users' martenitsas token balance. For this attack the attacker would proceed as follows:
- `victim` mints the minimum amount required of martenitsas to claim the health token.
- `attacker` calls three times `MartenitsaToken::updateCountMartenitsaTokensOwner` passing the victim's address and `sub` as the function call parameters. 
- `victim` calls `MartenitsaMarketplace::collectReward` but since his/her balance is 0 as a result of the attacker's calls, he/she does not have the right to claim for the corresponding health token.

See PoC below:

<details>
<summary>Code</summary>

Place this in `MartenitsaToken.t.sol`.
```javascript
    function test_attackerDecreaseVictimBalance() public eligibleForReward {
        address attacker = makeAddr("attacker");
        address victim = bob;
        vm.startPrank(attacker);
        for (uint256 i; i < 3; i++) {
            martenitsaToken.updateCountMartenitsaTokensOwner(victim, "sub");
        }
        vm.stopPrank();
        vm.prank(victim);
        marketplace.collectReward();
        assertEq(healthToken.balanceOf(victim), 0);
    }
```
</details>

## Tools Used
Foundry, manual review.
## Recommendations
Restrict the access to the function to only the marketplace address. For example, a function can be added to set the address of the marketplace and then check at the beginning of `MartenitsaToken::updateCountMartenitsaTokensOwner` if the caller matches the marketpace address, if not, revert. It can be easily implemented using a require:
```diff
import {MartenitsaMarketplace} from "./MartenitsaMarketplace.sol";
...
    function setMarketAddress(address martenitsaMarketplace) public onlyOwner {
        _martenitsaMarketplace = MartenitsaMarketplace(martenitsaMarketplace);
    }
...
function updateCountMartenitsaTokensOwner(
        address owner,
        string memory operation
    ) external {
+        require(
+            msg.sender == address(_martenitsaMarketplace),
+            "Caller should be marketplace!"
+       );
...
```

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. `MartenitsaVoting::voteCounts` and `MartenitsaVoting::hasVoted` are not reset after the winner is announced            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L57-L74

## Summary
The mappings where the count of votes are stored and where it is recorded if the users have particpated in the current voting is not reset after the winner is announced. This means that users can only vote once and the likelihood of early minted tokens of being selected as future winners is higher since they will accrue points since their first voting phase.
## Vulnerability Details
This vulnerability has two main implications, first, martenitsas with lower token IDs will begin next rounds with the already accumulated points and therefore with some advantage over newly minted or previously less voted tokens, reducing the fairness of the competition. Second, once users participate in a voting, if they want to particpate in future votings they will have to do it from a different address.
## Impact
To illustrate the first implication, imagine the following scenario:
- `chasy` mints token ID `0` before the first voting phase begins.
- The voting begins and `bob` and `someUser` vote for her token. Token ID `0` has accumulated 2 points.
- The voting concludes and `chasy` is announced as the winner.
- After finishing the first voting, `jack` mints token ID `1`.
- The new voting begins and now only `anotherUser` votes and decides to vote for token ID `1`. Token ID `1` has 1 vote.
- The voting concludes and since the voting count has not been previously reset, `chasy` is announced again as the winner with no votes in this phase.

See PoC below.

<details>
<summary>Code</summary>

Place this into `MartenitsaVoting.t.sol`.

```javascript
    function test_votesNotReset() public listMartenitsa {
        // someUser and bob vote for alice token
        address someUser = makeAddr("someUser");
        vm.prank(someUser);
        voting.voteForMartenitsa(0);
        vm.prank(bob);
        voting.voteForMartenitsa(0);
        vm.warp(block.timestamp + 1 days + 1);
        vm.recordLogs();
        voting.announceWinner();

        Vm.Log[] memory entries = vm.getRecordedLogs();
        address winner = address(uint160(uint256(entries[0].topics[2])));
        assert(winner == chasy);

        // 2nd round starts
        voting.startVoting();

        // 2nd round, jack mints a new token
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        marketplace.listMartenitsaForSale(1, 1 wei);
        vm.stopPrank();

        // now a third user votes for jack
        address anotherUser = makeAddr("anotherUser");
        vm.prank(anotherUser);
        voting.voteForMartenitsa(1);

        // since jack has the only vote in the second voting and it has concluded, he should be the winner, however...
        vm.warp(block.timestamp + 1 days + 1);
        vm.recordLogs();
        voting.announceWinner();

        entries = vm.getRecordedLogs();
        winner = address(uint160(uint256(entries[0].topics[2])));
        assert(winner == chasy);
    }
```

</details>

For the second implication the following scenario can be considered:
- `chasy` mints token ID `0` and the voting begins.
- `bob` votes for token ID `0`.
- The voting concludes and `chasy` is announced as the winner.
- The second voting begins and no one mints a new token.
- `bob` wants to vote again for `chasy`'s token, however the transaction reverts as the mapping `MartenitsaVoting::hasVoted` is still set to true.

See PoC below.

<details>
<summary>Code</summary>

Place this into `MartenitsaVoting.t.sol`.
```javascript
    function test_userCanOnlyVoteOnce() public listMartenitsa {
        vm.prank(bob);
        voting.voteForMartenitsa(0);
        vm.warp(block.timestamp + 1 days + 1);
        voting.announceWinner();

        // 2nd round starts
        voting.startVoting();

        // 2nd round, bob votes again for the same token, but he cannot
        vm.expectRevert("You have already voted");
        vm.prank(bob);
        voting.voteForMartenitsa(0);
    }
```
</details>

## Tools Used
Foundry and manual review.
## Recommendations
The fix to reset the vote count is easy and it does not require to change the logic of the contract, however for the partcipants votes it requires to add an extra array and a for loop which are described in detail in the code below.

<details>
<summary>Code</summary>

Adjust this in `MartenitsaVoting`.

```diff
...
+    address[] voters;
...
function voteForMartenitsa(uint256 tokenId) external {
...
        hasVoted[msg.sender] = true;
+      voters.push(msg.sender);
...
    function announceWinner() external onlyOwner {
        require(
            block.timestamp >= startVoteTime + duration,
            "The voting is active"
        );

        uint256 winnerTokenId;
        uint256 maxVotes = 0;

        for (uint256 i = 0; i < _tokenIds.length; i++) {
            if (voteCounts[_tokenIds[i]] > maxVotes) {
                maxVotes = voteCounts[_tokenIds[i]];
                winnerTokenId = _tokenIds[i];
            }
+           delete voteCounts[_tokenIds[i]];
        }

        list = _martenitsaMarketplace.getListing(winnerTokenId);
        _healthToken.distributeHealthToken(list.seller, 1);
        hasVoted[msg.sender] = false;

+       uint256 votersLength = voters.length;

+       for (uint256 i; i < votersLength; i++) {
+           hasVoted[voters[i]] = false;
+       }

        emit WinnerAnnounced(winnerTokenId, list.seller);
    }
...
```

</summary>
## <a id='M-02'></a>M-02. The array `MartenitsaEvent::participants` is not emptied after stopping the event and it can become too expensive to finish it later on.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaEvent.sol#L60-L65

## Summary
When the event finishes, the owner of the contract has to call the function `MartenitsaEvent::stopEvent` to remove the producer role from each participant. This is done by iterating in a for loop through each particpant and setting the value of the mapping `MartenitsaEvent::isProducer` to false. Since the array `MartenitsaEvent::participants` used in the for loop is not emptied after the event finishes it can become too expensive in the future to call this function. 
## Vulnerability Details
Every time a participant joins the event, its address is added to the array `MartenitsaEvent::participants`. When the deadline is met and the owner calls `MartenitsaEvent::stopEvent` to set back to the previous state the roles of the users, it forgets to empty this array. Since the rolling back to the previous role is performed using a for loop, as the array becomes bigger, calling this function may become economically inviable. This is worsened by the poor gas optimization techniques being used such as not caching the array length in memory.
## Impact
As a consequence of not emptying the array, the gas consumed in a function call to `MartenitsaEvent::stopEvent` will become more and more expensive. To get an estimate of the magnitude, the test below is executed for different number of particpants, determined by the iterations in the for loops.

<details>
<summary>Code</summary>

Place this into `MartenitsaEvent.t.sol`.
```javascript
    function test_stopEventDOS() public {
        martenitsaEvent.startEvent(1 days);
        vm.startPrank(chasy);
        uint256 k;

        for (uint256 i = 1; i < 100001; i++) {
            for (uint256 j; j < 3; j++) {
                address receiver = address(uint160(i));
                martenitsaToken.createMartenitsa("bracelet");
                martenitsaToken.approve(address(marketplace), k);
                marketplace.makePresent(receiver, k);
                k = k + 1;
            }
        }
        vm.stopPrank();
        for (uint256 i = 1; i < 100001; i++) {
            vm.startPrank(address(uint160(i)));
            marketplace.collectReward();
            healthToken.approve(address(martenitsaEvent), 10 ** 18);
            martenitsaEvent.joinEvent();
            vm.stopPrank();
        }
        vm.warp(block.timestamp + 1 days + 1);
        uint256 gasBefore = gasleft();
        martenitsaEvent.stopEvent();
        uint256 gasAfter = gasleft();
        console.log("gas used: %d", gasBefore - gasAfter);
    }
```
</details>

The gas consumed in the test for the different iterations are:

- 1,000 iterations: 780,180 gas
- 10,000 iterations: 7,791,180 gas
- 100,000 iterations: 77,901,180 gas

It can be observed that the gas increases at a rate of approximately 779 gas per particpant joining the event. Considering that Ethereum is the chosen network to deploy the protocol, it can be something very problematic during periods of high network congestion as the gas price will be considerably high.

## Tools Used
Foundry and manual review.
## Recommendations
Implement an optimized code that deletes the participants array and caches the array length in memory to avoid reading from storage in every iteration.

<details>
<summary>Code</summary>

```javascript
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");
        uint256 partcipantsLength = participants.length;
        for (uint256 i = 0; i < partcipantsLength; i++) {
            isProducer[participants[i]] = false;
            delete participants[i];
        }
    }
```
</details>
## <a id='M-03'></a>M-03. Users can directly transfer martenitsas via ´MartenitsaToken::transferFrom´ and their balance will not be updated.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/lib/openzeppelin-contracts/contracts/token/ERC721/ERC721.sol#L137-L147

## Summary
One of the functionalitites of the Baba Marta protocol is to make presents to friends, for this reason the function `MartenitsaMarketplace::makePresent` is included. However, the contract `MartenitsaToken` fails to override the function `transferFrom` from the imported dependencies. This allows users to transfer martenitsa tokens without their balances in the mapping `MartenitsaToken::countMartenitsaTokensOwner` being updated.
## Vulnerability Details
Users can decide to transfer martenitsa tokens using `MartenitsaToken::transferFrom`. While the mapping `MartenitsaToken::countMartenitsaTokensOwner` remains unchanged, the new address will be set as the owner in the inherited mapping from the dependency `MartenitsaToken::_owners`, leading to incorrect information being stored which can result in unintended functionalities.
## Impact
Incorrect information will be stored in the variable `MartenitsaToken::countMartenitsaTokensOwner`, this can lead to situations such as the following:
- `bob` acquires three items and transfers one token via `MartenitsaToken::transferFrom`. While now he is registered as the owner of two tokens and the minimum amount to claim a health token is three, he can still do it with one less.
See PoC below.
<details>
<summary>Code</summary>

Place this in `MartenitsaToken.t.sol`.
```javascript
    function test_BobClaimsHealthTokenWith2Martenitsas()
        public
        eligibleForReward
    {
        vm.startPrank(bob);
        martenitsaToken.transferFrom(bob, chasy, 2);
        marketplace.collectReward();
        assertEq(healthToken.balanceOf(bob), 10 ** 18);
    }
```
</details>

- Another scenario implies `bob` sending a token to `chasy` via `MartenitsaToken::transferFrom`, if `chasy` is just in possession of this token and tries to make a present to `jack`. The transaction will throw an arithmetic overflow error since the balance in `MartenitsaToken::countMartenitsaTokensOwner` is 0 and therefore it will result in -1 tokens owned by `chasy` after the transfer. See PoC below.

<details>
<summary>Code</summary>

Place this in `MartenitsaToken.t.sol`.
```javascript
import {Test, console, stdError} from "forge-std/Test.sol";
...
    function test_makePresentReverts() public hasMartenitsa {
        vm.prank(bob);
        martenitsaToken.transferFrom(bob, chasy, 0);
        vm.prank(chasy);
        vm.expectRevert(stdError.arithmeticError);
        marketplace.makePresent(jack, 0);
    }
```
</details>

## Tools Used
Foundry and manual review.
## Recommendations
Override the function `transferFrom` so that it reverts upon invocation. This way users have to go through `MartenitsaMarketplace::makePresent` to transfer their items. See example below:

```javascript
...
error MartenitsaToken__CannotDirectlyTransfer();
...
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public override {
        revert MartenitsaToken__CannotDirectlyTransfer();
    }
...
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Users cannot cancel existing listings            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L1-L120

## Summary
Users can list their martenitsa tokens by calling `MartenitsaMarketplace::listMartenitsaForSale`, however once listed they cannot remove the listing.
## Vulnerability Details
The only way for users not willing to sell their tokens will be to set the price to a extremely high value so that no one is able to pay for them.
## Impact 
The intended functionality of the the field `forSale` within the struct `MartenitsaMarketplace::Listing` is invalidated in the current contract design as this value can never be set to false.
## Tools Used
Foundry and manual review.
## Recommendations
Add a function which allows to cancel the listing:

```javascript
    function cancelListing(uint256 tokenId) public {
        require(
            (tokenIdToListing[tokenId].seller == msg.sender) &&
                (tokenIdToListing[tokenId].forSale == true),
            "You do not own this NFT or it is not listed for sale"
        );

        tokenIdToListing[tokenId].forSale = false;
        tokenIdToListing[tokenId].price = 0;
    }
```
Note that the price is set to 0 as well to make the process more efficient and get a gas refund by freeing up some memory space.


