# First Flight #12: Kitty Connect - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Missing tokens approval](#H-01)
    - ### [H-02. Missing access control](#H-02)
    - ### [H-03. Mapping with users' tokens ID not updated when bridging](#H-03)
    - ### [H-04. Sender's mapping with token IDs is not updated after transfer ](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Unrecoverable funds deposited accidentally](#M-01)
    - ### [M-02. Indices not updated after bridge transfer](#M-02)
- ## Low Risk Findings
    - ### [L-01. Ownership immutability](#L-01)
    - ### [L-02. Age of the cats will slightly vary in different chains](#L-02)
    - ### [L-03. Hardcoded fees address](#L-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #12: Kitty Connect

### Dates: Mar 28th, 2024 - Apr 4th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 2
   - Low: 3


# High Risk Findings

## <a id='H-01'></a>H-01. Missing tokens approval            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyBridge.sol#L53-L77

## Summary
`KittyBridge.sol::bridgeNftWithData()` will revert since no approval is being granted to the router for the fees transfer.
## Vulnerability Details
Bridging NFTs incurs a cost, payable in LINK or alternative assets. Since LINK is an ERC-20 token, it necessitates prior approval for transfer by an external party on behalf of the contract.
## Impact
Tokens cannot be bridged since a call to the function will trigger the error: `ERC20: transfer amount exceeds allowance`. Since the fees token address is hardcoded there is no way to circumvent this issue.
## Proof of Code 
Add the following to the current test suite:

<details>
<summary>Code</summary>

```javascript
function test_bridgeNFTReverts() public {
        address someUser = makeAddr("someUser");
        string
            memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";

        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(
            someUser,
            catImageIpfsHash,
            "Hehe",
            "Hehe",
            block.timestamp
        );

        uint64 otherChainSelector = 14767482510784806043;
        address destChainBridge = makeAddr("destChainBridge");

        vm.expectRevert("ERC20: transfer amount exceeds allowance");
        vm.prank(someUser);
        kittyConnect.bridgeNftToAnotherChain(
            otherChainSelector,
            destChainBridge,
            0
        );
        vm.stopPrank();
    }
```
</details>

## Tools Used
Manual review
## Recommendations
Add the following to the function `KittyBridge.sol::bridgeNftWithData`:

```javascript
s_linkToken.approve(address(router), fees);
```

## <a id='H-02'></a>H-02. Missing access control            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyBridge.sol#L53-L77

## Summary
Missing access control in the bridge contract.
## Vulnerability Details
The `KittyBridge.sol::bridgeNftWithData()` function lacks proper access control, allowing unauthorized users to call it. 
## Impact
Arbitrary users can call this function and transfer tokens bypassing the intended functionality of `KittyConnect.sol::bridgeNftToAnotherChain()` as long as the sender address and source chain are in the destination allow list. This will allow minting new NFTs in other chains just paying the fees associated to the bridging process.
## Proof of Code
Add the following to the current test suite:
<details>
<summary>Code</summary>

```javascript
function test_maliciousUserCanBridgeNFT() public {
        uint64 otherChainSelector = 14767482510784806043;
        address destChainBridge = makeAddr("destChainBridge");
        address randomUser = makeAddr("randomUser");
        string
            memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";

        bytes memory data = abi.encode(
            randomUser,
            "Hehe",
            "Hehe",
            catImageIpfsHash,
            block.timestamp,
            partnerA
        );

        vm.startBroadcast(randomUser);
        kittyBridge.bridgeNftWithData(
            otherChainSelector,
            destChainBridge,
            data
        );
        vm.stopBroadcast();
    }
```
</details>

Then run the command: `forge test --mt test_maliciousUserCanBridgeNFT --fork-url $(grep -w SEPOLIA_RPC_URL .env | cut -d '=' -f2)`

## Tools Used
Manual review.
## Recommendations
Implement access control at the beginning of the function. For example:

```javascript
if (msg.sender != kittyConnect) {
            revert KittyBridge__NotKittyConnect();
        }
```

Note: The error `KittyBridgeBase.sol::KittyBridge__NotKittyConnect()` is declared but not utilized.
## <a id='H-03'></a>H-03. Mapping with users' tokens ID not updated when bridging            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L154-L179

## Summary
The mapping with users' tokens IDs is not updated when a token is minted through the bridge.
## Vulnerability Details
Although the variable `KittyConnect.sol::s_ownerToCatsTokenId` is updated whenever a user mints or bridges an NFT, it remains stagnant when receiving NFTs from another chain. 
## Impact
When users bridge their NFTs, the mapping is solely updated in the source chain. Consequently, users are restricted from bridging a number of tokens equal to the native minted tokens minus the total minted using the bridge. This limitation arises because the function `KittyConnect.sol::bridgeNftToAnotherChain()` throws an error due to the empty array. Refer to the example below for clarification.
## Proof of Code
Add the following to the current test suite and import the helper contract `stdError`:
<details>
<summary>Code</summary>

```javascript
function test_tokenNotAddedToMapping() public {
        address sender = makeAddr("sender");
        address catOwner = makeAddr("catOwner");
        bytes memory data = abi.encode(
            catOwner,
            "meowdy",
            "ragdoll",
            "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62",
            block.timestamp,
            partnerA
        );

        vm.prank(kittyConnectOwner);
        kittyBridge.allowlistSender(networkConfig.router, true);

        Client.Any2EVMMessage memory message = Client.Any2EVMMessage({
            messageId: bytes32(0),
            sourceChainSelector: networkConfig.otherChainSelector,
            sender: abi.encode(sender),
            data: data,
            destTokenAmounts: new Client.EVMTokenAmount[](0)
        });

        vm.prank(networkConfig.router);

        kittyBridge.ccipReceive(message);

        uint256[] memory cats = kittyConnect.getCatsTokenIdOwnedBy(catOwner);

        uint64 otherChainSelector = 14767482510784806043;
        address destChainBridge = makeAddr("destChainBridge");

        vm.expectRevert(stdError.arithmeticError);
        vm.prank(catOwner);
        kittyConnect.bridgeNftToAnotherChain(
            otherChainSelector,
            destChainBridge,
            0
        );
        vm.stopPrank();
    }
```
</details>

## Tools Used
Manual review.
## Recommendations
Add the following to `KittyConnect.sol::mintBridgedNFT()`:

```javascript
s_ownerToCatsTokenId[catOwner].push(tokenId);
```
## <a id='H-04'></a>H-04. Sender's mapping with token IDs is not updated after transfer             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L181-L185

## Summary
The function `KittyConnect.sol::_updateOwnershipInfo()` neglects to remove the transferred item from the array `KittyConnect.sol::s_ownerToCatsTokenId`.
## Vulnerability Details
Upon transfer of an item to another address via `KittyConnect.sol::safeTransferFrom()`, the item is correctly added to the receiver's address mapping. However, it remains in the sender's mapping, resulting in inaccurate data.
## Impact
It will show incorrect information when calling the getter `KittyConnect.sol::getCatsTokenIdOwnedBy()`.
## Proof of Code

<details>
<summary>Code</summary>

```javascript
function test_tokenIsNotRemoved() public {
        address someUser = makeAddr("someUser");
        address otherUser = makeAddr("otherUser");
        string
            memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";

        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(
            someUser,
            catImageIpfsHash,
            "Hehe",
            "Hehe",
            block.timestamp
        );
        vm.prank(someUser);
        kittyConnect.approve(otherUser, 0);
        vm.prank(partnerA);
        kittyConnect.safeTransferFrom(someUser, otherUser, 0);
        uint256[] memory userTokenIDs = kittyConnect.getCatsTokenIdOwnedBy(
            someUser
        );
        uint256 tokenStillThere = userTokenIDs[0];

        assertEq(tokenStillThere, 0);
    }
```
</details>

## Tools Used
Manual review.

## Recommendations
Update the array accordingly in function `KittyConnect.sol::_updateOwnershipInfo()`:

```diff
function _updateOwnershipInfo(
        address currCatOwner,
        address newOwner,
        uint256 tokenId
    ) internal {
        s_catInfo[tokenId].prevOwner.push(currCatOwner);
        s_catInfo[tokenId].idx = s_ownerToCatsTokenId[newOwner].length;
        s_ownerToCatsTokenId[newOwner].push(tokenId);
+      uint256[] memory userTokenIds = s_ownerToCatsTokenId[msg.sender];
+      uint256 lastItem = userTokenIds[userTokenIds.length - 1];
+      s_ownerToCatsTokenId[msg.sender].pop(); 
+      if (idx < (userTokenIds.length - 1)) {
+           s_catInfo[lastItem].idx = idx;
+           s_ownerToCatsTokenId[msg.sender][idx] = lastItem; 
        }
}
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Unrecoverable funds deposited accidentally            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/base/KittyBridgeBase.sol#L10-L11

## Summary
Two custom errors within `KittyBridgeBase.sol` have been identified but not implemented, indicating potential issues with the recovery of user funds deposited accidentally into the contract. Additionally, the `KittyConnect.sol::TokensRedeemedForVetVisit` event is declared but not utilized in any functions, suggesting a missing functionality within the contract.
## Vulnerability Details
Currently, there is no mechanism to recover ERC-20 or native tokens sent unintentionally by users. To address this issue, two distinct functions could be added to facilitate the recovery of each type of asset. Refer to the example below for implementation details
## Impact
The absence of a recovery mechanism poses a risk of permanent loss of user funds.
## Tools Used
Manual review.
## Recommendations
Implement rescue functions such as the following:

```javascript
struct RetrieveData {
        address target;
        bytes data;
        uint256 value;
    }
    function retrieveNativeAssets(
        RetrieveData[] calldata retrieveData
    ) external payable onlyOwner {
        if (address(this).balance == 0) {
            revert KittyBridge__NothingToWithdraw();
        }
        for (uint256 i = 0; i < retrieveData.length; ++i) {
            (bool success, ) = address(retrieveData[i].target).call{
                value: retrieveData[i].value
            }(retrieveData[i].data);
            if (!success) {
                revert KittyBridge__FailedToWithdrawEth(
                    owner(),
                    retrieveData[i].target,
                    retrieveData[i].value
                );
            }
        }
    }
    function retrieveERC20Tokens(
        address token,
        address to,
        uint256 amount
    ) external onlyOwner {
        IERC20(token).transfer(to, amount);
    }
```

Find PoC associated to the rescue functions attached:

<details>
<summary>Code</summary>

```javascript
function test_recoverERC20() public {
        address someUser = makeAddr("someUser");
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        IERC20(networkConfig.link).transfer(someUser, 1 ether);
        vm.stopBroadcast();
        vm.prank(someUser);
        IERC20(networkConfig.link).transfer(address(kittyBridge), 1 ether);
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        kittyBridge.retrieveERC20Tokens(networkConfig.link, someUser, 1 ether);
        vm.stopBroadcast();
        assertEq(IERC20(networkConfig.link).balanceOf(someUser), 1 ether);
    }

function test_recoverNativeAssets() public {
        address someUser = makeAddr("someUser");
        vm.deal(someUser, 0.1 ether);
        vm.startBroadcast(someUser);
        (bool success, ) = address(kittyBridge).call{value: 0.1 ether}("");
        vm.stopBroadcast();
        KittyBridge.RetrieveData[]
            memory retrieveData = new KittyBridge.RetrieveData[](1);
        retrieveData[0] = KittyBridge.RetrieveData(someUser, "", 0.1 ether);
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        kittyBridge.retrieveNativeAssets(retrieveData);
        vm.stopBroadcast();
        assertEq(someUser.balance, 0.1 ether);
    }
```
</details>
## <a id='M-02'></a>M-02. Indices not updated after bridge transfer            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L146-L148

## Summary
The tokens' indices are not updated when transferred to other chains.
## Vulnerability Details
Upon transferring a token to another chain, adjustments are made in the `KittyConnect.sol::s_ownerToCatsTokenId` variable to remove the transferred item. If the transferred token does not occupy the last position in the array, it is replaced by the last item. However, the code fails to update the new index according to the movement of the item in the list. Subsequently, if this item is later transferred and there are elements to its right, the last element will be removed instead of the corresponding token.
## Impact
Using the getter function `KittyConnect.sol::getCatsTokenIdOwnedBy()` to query the tokens owned by the caller will yield incorrect token IDs due to the misalignment of indices caused by the flawed update mechanism.
## Tools Used
Manual review.
## Proof of Code
Add the following to the current test suite:
<details>
<summary>Code</summary>

```javascript
function test_idxNotUpdated() public {
        address someUser = makeAddr("someUser");
        string
            memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";

        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(
            someUser,
            catImageIpfsHash,
            "Hehe",
            "Hehe",
            block.timestamp
        );
        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(
            someUser,
            catImageIpfsHash,
            "Hehe",
            "Hehe",
            block.timestamp
        );

        uint64 otherChainSelector = 14767482510784806043;
        address destChainBridge = makeAddr("destChainBridge");

        uint256 idxBefore = kittyConnect.getCatInfo(1).idx;

        vm.prank(someUser);
        kittyConnect.bridgeNftToAnotherChain(
            otherChainSelector,
            destChainBridge,
            0
        );
        vm.stopPrank();

        uint256 idxAfter = kittyConnect.getCatInfo(1).idx;

        assertEq(idxBefore, idxAfter);
    }
```
</details>

## Recommendations
Update the idx accordingly:

```diff
if (idx < (userTokenIds.length - 1)) {
+          s_catInfo[lastItem].idx = idx;
            s_ownerToCatsTokenId[msg.sender][idx] = lastItem;
        }

```


# Low Risk Findings

## <a id='L-01'></a>L-01. Ownership immutability            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L29

## Summary
The `kittyConnect.sol::i_kittyConnectOwner` variable is currently declared as immutable, preventing future ownership transfers.
## Vulnerability Details
Immutable variables, once set during deployment, remain unchanged thereafter. If the owner's wallet is compromised or lost, critical functionalities may be impacted such as adding new shops, source and destinations chains or senders to the allow lists.
## Tools Used
Manual review.
## Recommendations
Consider declaring the variable as mutable. Additionally, implement a function allowing ownership transfers, restricted to only the current owner's invocation. 
## <a id='L-02'></a>L-02. Age of the cats will slightly vary in different chains            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L217-L219

## Summary
The age of a cat is determined by calculating the difference between the current timestamp and the timestamp recorded during its minting process. This timestamp, based on the current block timestamp in seconds since the Unix epoch, can vary across different blockchains.
## Vulnerability Details
Different blockchain platforms exhibit unique characteristics such as differing numbers of blocks, block production rates, and frequencies and depths of reorganizations. Consequently, when transferring a cat from one blockchain to another, slight variations in the cat's age may occur.
## Impact
The variability in the age of cats across blockchains could potentially disrupt off-chain functionalities, especially if they rely on consistent age information.
## Tools Used
Manual review.
## Recommendations
Consider conducting age calculations off-chain while retaining the date of birth on-chain for reference. This approach would ensure consistent age representation regardless of blockchain differences.
## <a id='L-03'></a>L-03. Hardcoded fees address            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyBridge.sol#L60

## Summary
Fees token address is currently hardcoded and does not allow users to select their preferred payment method.
## Vulnerability Details
The function `KittyBridge.sol::bridgeNftWithData()` is responsible for preparing the message sent to the router. Currently, the address of the token used to pay the associated fees is hardcoded. However, according to [Chainlink's Docs](https://docs.chain.link/ccip/getting-started), fees can also be paid in the native asset by setting the value to the null address and sending the corresponding payment in the blockchain's native asset.
## Impact
Users cannot decide their preferred payment method.
## Tools Used
Manual review.
## Recommendations
Do not hardcode the value and allow users to set it by themselves.


