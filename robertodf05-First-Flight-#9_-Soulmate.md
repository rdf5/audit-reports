# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `Airdrop.sol` is performing wrong the check on the soulmate's matrial status ](#H-01)
    - ### [H-02. Incorrect Staking Logic Implementation in `Staking.sol`](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Users can be their own soulmate](#M-01)
- ## Low Risk Findings
    - ### [L-01. Events should be emitted in the E following the CEI pattern](#L-01)
    - ### [L-02. A couple can get divorced multiple times](#L-02)
    - ### [L-03. Wrong event is emitted when soulmates are reunited](#L-03)
    - ### [L-04. Soulmate token gets only minted for the second soulmate](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 1
   - Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. `Airdrop.sol` is performing wrong the check on the soulmate's matrial status             



#### [H-2] `Airdrop.sol` is performing wrong the check on the soulmate's matrial status 

**Description:**  The soulmates can only claim the airdrop if they are not divorced, however the contract is checking its own matrial status instead of the msg.sender's.

**Impact:** Divorced soulmates will be able to claim their tokens:
```javascript
function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
        if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();
```

**Proof of Concept:** Place the following in `AirdropTest.t.sol`:

```javascript
function test_DivorcedCoupleCanClaim() public {
        _mintOneTokenForBothSoulmates();
        vm.startPrank(soulmate1);
        soulmateContract.getDivorced();
        vm.warp(block.timestamp + 1 days);
        airdropContract.claim();
        vm.stopPrank();
        assertTrue(
            loveToken.balanceOf(soulmate1) == 10 ** loveToken.decimals()
        );
    }
```


**Recommended Mitigation:** Modify `Soulmate.sol::isDivorced()` so that `Airdrop.sol` can check the specific user's status:

```javascript
function isDivorced(address soulmate) public view returns (bool) {
        return divorced[soulmate];
    }
```

## <a id='H-02'></a>H-02. Incorrect Staking Logic Implementation in `Staking.sol`            



#### [H-1] Incorrect Staking Logic Implementation in `Staking.sol`

**Description:**  The current implementation of staking logic allows users to claim more tokens than they should be entitled to. Additionally, if rewards are claimed before the completion of the last week, users lose the rewards for that week.

**Impact:** The issues with the staking logic lead to the following consequences:

1. The `claimRewards()` function assigns the wrong timestamp to users claiming rewards for the first time, resulting in incorrect reward calculations.

```javascript
if (lastClaim[msg.sender] == 0) {
            lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
                soulmateId
            );
```
For these users, the timestamp when the soulmate token was minted is assigned as the last claim. Since the rewards for each user are calculated as:

```javascript
uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
    lastClaim[msg.sender]) / 1 weeks);

if (timeInWeeksSinceLastClaim < 1)
    revert Staking__StakingPeriodTooShort();

lastClaim[msg.sender] = block.timestamp;

// Send the same amount of LoveToken as the week waited times the number of token staked
uint256 amountToClaim = userStakes[msg.sender] *
    timeInWeeksSinceLastClaim;
loveToken.transferFrom(
    address(stakingVault),
    msg.sender,
    amountToClaim
);
```

This implies that users don't need to deposit tokens in the contract to accrue love tokens, it will be enough depositing them the same day they want to claim the rewards (they have to at least wait for a week). This is in line with the next issue.

2. After the initial claim, the user's `lastClaim` timestamp is updated. Subsequently, as the reward is solely updated upon calling `Staking.sol::claimRewards()`, users can strategically deposit tokens on the same day they intend to claim rewards. This behavior implies that users don't necessarily need to maintain token deposits for longer than a week if they choose not to. This compounds with the issue highlighted earlier, exacerbating the potential for users to exploit the system by minimizing their token deposits to optimize reward accumulation.

3. In the same function `Staking.sol::claimRewards()`, users lose the tokens claimed after the completion of the last week, if they don't want to loose any of their rewards they have to claim them at exactly the end of the week:

```javascript
uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
    lastClaim[msg.sender]) / 1 weeks);
if (timeInWeeksSinceLastClaim < 1)
    revert Staking__StakingPeriodTooShort();

lastClaim[msg.sender] = block.timestamp;
```
The amount lost is equal to `(block.timestamp-lastClaim[msg.sender])%1 weeks*userStakes[msg.sender]/ 1 weeks`.

**Proof of Concept:** Place the following in `StakingTest.t.sol`:
<details>
<summary>Code</summary>

```javascript
function test_FirstClaimIsMisscalculated() public {
        uint256 balancePerSoulmates = 1 ether;
        _giveLoveTokenToSoulmates(balancePerSoulmates);
        uint256 weekOfStaking = 2;
        vm.warp(block.timestamp + weekOfStaking * 1 weeks);
        vm.startPrank(soulmate1);
        airdropContract.claim();
        loveToken.approve(address(stakingContract), 1 ether);
        stakingContract.deposit(1 ether);
        stakingContract.claimRewards();
        vm.stopPrank();
        assertTrue(loveToken.balanceOf(soulmate1) == 14 ether + 2 ether);
    }

    function test_userDepositsTokenWhenClaiming() public {
        uint256 balancePerSoulmates = 1 ether;
        _depositTokenToStake(balancePerSoulmates);
        uint256 weekOfStaking = 1;
        vm.warp(block.timestamp + weekOfStaking * 1 weeks);
        vm.startPrank(soulmate1);
        airdropContract.claim();
        loveToken.approve(address(stakingContract), 1 ether);
        stakingContract.deposit(1 ether);
        stakingContract.claimRewards();
        vm.stopPrank();
        assertTrue(loveToken.balanceOf(soulmate1) == 8 ether); // 6 accrued during a week and 2 from the staking rewards
    }

function test_stakedTokensAreLost() public {
        uint256 balancePerSoulmates = 1 ether;
        _depositTokenToStake(balancePerSoulmates);
        uint256 weekOfStaking = 2;
        vm.warp(block.timestamp + weekOfStaking * 1 weeks - 1);
        vm.prank(soulmate1);
        stakingContract.claimRewards();
        uint256 tokensLost = (((block.timestamp -
            stakingContract.lastClaim(msg.sender)) % 1 weeks) *
            balancePerSoulmates) / 1 weeks;
        assertTrue(
            loveToken.balanceOf(soulmate1) ==
                ((block.timestamp - stakingContract.lastClaim(msg.sender)) *
                    balancePerSoulmates) /
                    1 weeks -
                    tokensLost
        );
    }
```
</details>

**Recommended Mitigation:** Keep track of users' rewards in different points in time by implementing a modifier `updateReward()` and a function `earned()` as shown in the example below:

<details>
<summary>Code</summary>

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import {IVault} from "./interface/IVault.sol";
import {ISoulmate} from "./interface/ISoulmate.sol";
import {ILoveToken} from "./interface/ILoveToken.sol";

/// @title Staking Contract for LoveToken.
/// @author n0kto
contract CorrectStaking {
    /*//////////////////////////////////////////////////////////////
                                ERRORS
    //////////////////////////////////////////////////////////////*/
    error CorrectStaking__NoMoreRewards();

    /*//////////////////////////////////////////////////////////////
                            STATE VARIABLES
    //////////////////////////////////////////////////////////////*/

    ILoveToken public immutable loveToken;
    ISoulmate public immutable soulmateContract;
    IVault public immutable stakingVault;

    mapping(address user => uint256 loveToken) public userStakes;
    mapping(address => uint) public activeAt;
    uint256 public constant rewardPerToken = 1;
    mapping(address => uint) public rewardsPerAccount;

    /*//////////////////////////////////////////////////////////////
                                 EVENTS
    //////////////////////////////////////////////////////////////*/
    event Deposited(address indexed user, uint256 amount);
    event Withdrew(address indexed user, uint256 amount);
    event RewardsClaimed(address indexed user, uint256 amount);

    /*//////////////////////////////////////////////////////////////
                                 MODIFIERS
    //////////////////////////////////////////////////////////////*/
    modifier updateReward(address _account) {
        if (_account != address(0)) {
            rewardsPerAccount[_account] = earned(_account);
            activeAt[_account] = block.timestamp;
        }

        _;
    }

    /*//////////////////////////////////////////////////////////////
                               FUNCTIONS
    //////////////////////////////////////////////////////////////*/
    constructor(
        ILoveToken _loveToken,
        ISoulmate _soulmateContract,
        IVault _stakingVault
    ) {
        loveToken = _loveToken;
        soulmateContract = _soulmateContract;
        stakingVault = _stakingVault;
    }

    /// @notice Increase the userStakes variable and transfer LoveToken to this contract.
    function deposit(uint256 amount) public updateReward(msg.sender) {
        if (loveToken.balanceOf(address(stakingVault)) == 0)
            revert CorrectStaking__NoMoreRewards();
        // No require needed because of overflow protection
        userStakes[msg.sender] += amount;
        emit Deposited(msg.sender, amount);
        loveToken.transferFrom(msg.sender, address(this), amount);
    }

    /// @notice Decrease the userStakes variable and transfer LoveToken to the user withdrawing.
    function withdraw(uint256 amount) public updateReward(msg.sender) {
        // No require needed because of overflow protection
        userStakes[msg.sender] -= amount;
        emit Withdrew(msg.sender, amount);
        loveToken.transfer(msg.sender, amount);
    }

    /// @notice Claim rewards for staking.
    /// @notice Users can claim 1 token per staking token per week.
    function claimRewards() public updateReward(msg.sender) {
        if (loveToken.balanceOf(address(stakingVault)) == 0)
            revert CorrectStaking__NoMoreRewards();
        uint256 amountToClaim;
        // Dust collector
        if (
            loveToken.balanceOf(address(stakingVault)) <=
            rewardsPerAccount[msg.sender]
        ) {
            amountToClaim = loveToken.balanceOf(address(stakingVault));
        } else {
            amountToClaim = rewardsPerAccount[msg.sender];
        }

        rewardsPerAccount[msg.sender] = 0;

        emit RewardsClaimed(msg.sender, amountToClaim);

        loveToken.transferFrom(
            address(stakingVault),
            msg.sender,
            amountToClaim
        );
    }

    function earned(address _account) public view returns (uint) {
        uint256 elapsedSeconds = block.timestamp - activeAt[_account];
        return
            ((userStakes[_account] * rewardPerToken) * elapsedSeconds) /
            1 weeks +
            rewardsPerAccount[_account];
    }
}

```
</details>

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Users can be their own soulmate            



**Description:**  There is no check for whether the second soulmate's address is different from the first, therefore a user can mint two soulmate tokens.

**Impact:** The user will have double staking rate and airdrops.

**Proof of Concept:** Place the following in `SoulmateTest.t.sol`:

```javascript
function test_SameUserIsSoulmate() public {
        vm.startPrank(soulmate1);
        soulmateContract.mintSoulmateToken();
        soulmateContract.mintSoulmateToken();
        vm.stopPrank();
        assertTrue(soulmateContract.soulmateOf(soulmate1) == soulmate1);
    }
```

**Recommended Mitigation:** Check if the address of the second soulmate differs from the first:

<details>
<summary>Code</summary>

```diff
    function mintSoulmateToken() public returns (uint256) {
        // Check if people already have a soulmate, which means already have a token
        address soulmate = soulmateOf[msg.sender];
        if (soulmate != address(0))
            revert Soulmate__alreadyHaveASoulmate(soulmate);

        address soulmate1 = idToOwners[nextID][0];
        address soulmate2 = idToOwners[nextID][1];
        if (soulmate1 == address(0)) {
            idToOwners[nextID][0] = msg.sender;
            ownerToId[msg.sender] = nextID;
            emit SoulmateIsWaiting(msg.sender);
+        } else if (soulmate2 == address(0) && soulmate2 != soulmate1){
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;

            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

            _mint(msg.sender, nextID++);
        }
```
</details>


# Low Risk Findings

## <a id='L-01'></a>L-01. Events should be emitted in the E following the CEI pattern            



The contract `Staking.sol` frequently emits events after external calls, which is considered poor practice. It's advisable to emit events before executing external calls to ensure accurate event logging. For instance, in the `Staking.sol::withdraw()` function, this pattern is observed:

```diff
function withdraw(uint256 amount) public updateReward(msg.sender) {
        // No require needed because of overflow protection
        userStakes[msg.sender] -= amount;
+        emit Withdrew(msg.sender, amount);
        loveToken.transfer(msg.sender, amount);
-        emit Withdrew(msg.sender, amount);
    }
```

**Impact:** While emitting information after it has occurred might seem rational, it can introduce vulnerabilities. This approach can be exploited, potentially leading to manipulation and incorrect data being registered by monitoring systems. Ultimately, this could necessitate a code migration to rectify the issues caused by inaccurate event logging.

**Recommended Mitigation:** Place the events in the effects part of the function after the state variables have been updated but before the interactions with external elements.


## <a id='L-02'></a>L-02. A couple can get divorced multiple times            



**Description:**  A couple can call `Soulmate.sol::getDivorced()` multiple times, even when they are already divorced.

**Impact:** These may cause weird behavior in off-chain components.

**Recommended Mitigation:** Check whether the copule has already divorced:

```diff
function getDivorced() public {
+    if divorced[msg.sender] revert Soulmate__AlreadyDivorced();
    address soulmate2 = soulmateOf[msg.sender];
    divorced[msg.sender] = true;
    divorced[soulmateOf[msg.sender]] = true;
    emit CoupleHasDivorced(msg.sender, soulmate2); 
    }
```

## <a id='L-03'></a>L-03. Wrong event is emitted when soulmates are reunited            




**Description:**  The event emitted when the second soulmate calls the function `Soulmate.sol::mintSoulmateToken()` is emitting the wrong address in the second parameter. It is emitting the null address instead of the second soulmate's address:

```javascript
emit SoulmateAreReunited(soulmate1, soulmate2, nextID);
```

**Impact:** This can mess the monitoring systems tracking the activity of the contract, assigning the null address always as the second soulmate.

**Recommended Mitigation:** Update the second parameter to:

```javascript
emit SoulmateAreReunited(soulmate1, msg.sender, nextID);
```

## <a id='L-04'></a>L-04. Soulmate token gets only minted for the second soulmate            



**Description:**  The mint of the token takes place only if the else if statement is true as it can be seen in `Soulmate.sol::mintSoulmateToken()`:

```javascript
if (soulmate1 == address(0)) {
    idToOwners[nextID][0] = msg.sender;
    ownerToId[msg.sender] = nextID;
    emit SoulmateIsWaiting(msg.sender);
} else if (soulmate2 == address(0)) {
    idToOwners[nextID][1] = msg.sender;
    // Once 2 soulmates are reunited, the token is minted
    ownerToId[msg.sender] = nextID;
    soulmateOf[msg.sender] = soulmate1;
    soulmateOf[soulmate1] = msg.sender;
    idToCreationTimestamp[nextID] = block.timestamp;

    emit SoulmateAreReunited(soulmate1, soulmate2, nextID); 
    emit SoulmateAreReunited(soulmate1, msg.sender, nextID);
@>    _mint(msg.sender, nextID++); 
}
```

**Impact:** The first soulmate won't receive the NFT.

**Proof of Concept:** Add the following two lines to `SulmateTest.t.sol::test_MintNewToken`:

```javascript
assertTrue(soulmateContract.balanceOf(soulmate1) == 0);
assertTrue(soulmateContract.balanceOf(soulmate2) == 1);
```

**Recommended Mitigation:** Place the `_mint()` function in the first part of the if as well.


