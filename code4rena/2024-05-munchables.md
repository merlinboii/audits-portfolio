# Munchables - LockManager Report
Munchables is a GameFi project with a twist.

The objective of the game is to earn as many Munch Points as possible. In crypto terms, you could call this "point farming".

Built on top of Blast, Munchables leverages the unique on-chain primitives to create a reward-filled journey.
Players collect Munchables and keep them safe, fed and comfortable in their snuggery.
Once in a snuggery, a Munchable can start earning rewards for that player.
A variety of factors influence the rewards earned, so players will have to be smart when choosing which Munchables to put in their snuggery and the strategies they use to play the game.

* **nSLOC**: 413 
* **Contest details**: https://code4rena.com/audits/2024-05-munchables

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: [Severity Categorization | Code4rena](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[H-01](#h-01)| Players can reduce the lock duration less than the current unlock time|🍅|
|[H-02](#h-02)| Donation to extend the unlocking time of other players |🍅|
|[M-01](#m-01)| Players can gain more NFTs benefiting from that past remainder in subsequent locks |🍋|
|[M-02](#m-02)| Inconsistent conditions for approving and disapproving the proposal price |🍋|
|[L-01](#l-01)| Inconsistent conditions for approving and disapproving the proposal price |🫑|

---

## <a name="h-01">[H-01]</a> Players can reduce the lock duration less than the current unlock time

* **Severity**: High
* Source: [Players can reduce the lock duration less than the current unlock time](https://github.com/code-423n4/2024-05-munchables-findings/issues/72)

---
### Description
Players can reduce the lock duration to be less than the current unlock time via the [`setLockDuration()`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272) contract

Since the [`setLockDuration()`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272) extends the `unlockTime` from the `lastLockTime`, the `unlockTime` can be reduced.

Location: [LockManager::setLockDuration()L245-L272](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272)

```solidity
        function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();

        playerSettings[msg.sender].lockDuration = uint32(_duration);
        // update any existing lock
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                // check they are not setting lock time before current unlocktime
                if (
                    uint32(block.timestamp) + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }

->L263          uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
                    .lastLockTime;
->L265          lockedTokens[msg.sender][tokenContract].unlockTime =
                    lastLockTime +
                    uint32(_duration);
            }
        }

        emit LockDuration(msg.sender, _duration);
    }
```
---
### Impact
Players can reduce the `unlockingTime` to unlock their locked tokens faster than the previous `unlockingTime`.

This impact is multiplied when they reduce the unlocking time during the lock drop period. By doing so, players can unlock and use the funds from that unlock to reenter during the lock drop and receive more NFTs.

The following core function(s) take an effect of this issue:
* [LockManager::setLockDuration()L245-L272](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272)


### Proof of Concept
#### Prerequisite
  - Using the `address(0)` as a token contract represents locking the native token
  - The player `lockDuration` is `604800` (7 days)
---
1. The player locks ethers for themselves at on `2024-01-11, 00:00:00` by entering the [`lock`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L297-L309) function

The `lockedToken` struct updates as follows:
* `lockedToken.lastLockTime` = uint32(`2024-01-11, 00:00:00`);
* `lockedToken.unlockTime` = uint32(`2024-01-11, 00:00:00`) + uint32(`7 days`); = **2024-01-018, 00:00:00**


2. On the `2024-01-15, 00:00:00`, the player execute [`setLockDuration()`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272) by passing the `_duration` as **4 days**:

```solidity
    LockManager.setLockDuration(
        uint256 _duration: 4 days
    )
```

The condition check will be pass and not revert as:
```solidity
// L256 - 261

    if (
        uint32(block.timestamp) + uint32(_duration) <
        lockedTokens[msg.sender][tokenContract].unlockTime
        // `2024-01-15, 00:00:00` + 4 days < `2024-01-18, 00:00:00`
        // `2024-01-19, 00:00:00` < `2024-01-18, 00:00:00`
        // FALSE
    ) {
        revert LockDurationReducedError();
    }
```

The `lockedToken.unlockTime` is updated as 
* `lockedToken.unlockTime`\
= `lockedToken.lastLockTime + uint32(_duration);`\
= `2024-01-11, 00:00:00` + 4 days\
=  **2024-01-15, 00:00:00**

[![](https://mermaid.ink/img/pako:eNqFk21rwjAQx79KCAgKrSRt7dNr2asJg_nGIYysOWuwTSRNh0787ktsnU8bu1fXu_x_ubteDrhQHHCOSyaNWUpkzQhTAXpWxUbIEs1FDZWQ0OU4M_CkdM0MQgtr_mzmT6ddju1E0-cGC39Q-wPeJRoojFASDVBlocNRFz1f8AJaKI4ouljOrOATPMSohwISRD6hPrV-0hMdB_hcbUCOK9YYx3KFXmtrUUFjlHSY4AbzyGhl1VXjIO_0d0h4BUkt5L65Bk51TFvNXOTc50PYFYkKLcwtPrrgCZ30Rb7zXoaGkR3-vhldhtQzeCdfGdAPEBTxf4Z9prDJtZJelH-PKUDXv-yxofi-IezhGux-CG4X7uDwS2zWUMMS59b9YI31lvJoz7HWqNe9LHBudAse1qot1zhfsaqxX-3W7eFUsFKz-ie6ZRLnB7zDOSXZOMyymE7SMI1IHIYe3uM8INGYhkmcTqylMSXp0cNfSlkEGdtDlCZRmIRxECckO_HeTsmuBODCKD3rnsvp1Ry_AXDb84Y?type=png)](https://mermaid.live/edit#pako:eNqFk21rwjAQx79KCAgKrSRt7dNr2asJg_nGIYysOWuwTSRNh0787ktsnU8bu1fXu_x_ubteDrhQHHCOSyaNWUpkzQhTAXpWxUbIEs1FDZWQ0OU4M_CkdM0MQgtr_mzmT6ddju1E0-cGC39Q-wPeJRoojFASDVBlocNRFz1f8AJaKI4ouljOrOATPMSohwISRD6hPrV-0hMdB_hcbUCOK9YYx3KFXmtrUUFjlHSY4AbzyGhl1VXjIO_0d0h4BUkt5L65Bk51TFvNXOTc50PYFYkKLcwtPrrgCZ30Rb7zXoaGkR3-vhldhtQzeCdfGdAPEBTxf4Z9prDJtZJelH-PKUDXv-yxofi-IezhGux-CG4X7uDwS2zWUMMS59b9YI31lvJoz7HWqNe9LHBudAse1qot1zhfsaqxX-3W7eFUsFKz-ie6ZRLnB7zDOSXZOMyymE7SMI1IHIYe3uM8INGYhkmcTqylMSXp0cNfSlkEGdtDlCZRmIRxECckO_HeTsmuBODCKD3rnsvp1Ry_AXDb84Y)

📍 **Finally, the player can reduce the unlock time from `2024-01-018, 00:00:00` to `2024-01-015, 00:00:00`**
---
### Tools Used
* Manual Review

---
### Recommended Mitigation Steps
Update the condition checks at [L256-L261](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L256-L261) in the `LockManager` contract to use the `lockedToken.lastLockTime` instead.

This adjustment ensures that players cannot decrease the unlock time in any way, even after the unlock timestamp has passed.

```diff
    function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();

        playerSettings[msg.sender].lockDuration = uint32(_duration);
        // update any existing lock
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                // check they are not setting lock time before current unlocktime
+                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract].lastLockTime;
                if (
-                   uint32(block.timestamp) + uint32 (_duration) <
+                   lastLockTime + uint32 (_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }

-                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
-                    .lastLockTime;
                lockedTokens[msg.sender][tokenContract].unlockTime =
                    lastLockTime +
                    uint32(_duration);
            }
        }

        emit LockDuration(msg.sender, _duration);
    }
```

---

## <a name="h-02">[H-02]</a> Donation to extend the unlocking time of other players

* **Severity**: High
* Source: [Donation to extend the unlocking time of other players](https://github.com/code-423n4/2024-05-munchables-findings/issues/553)

---
### Description
The [lockOnBehalf()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L275-L294) allows any user to donate 1 wei to extend the unlocking time of any player's locks.

Location: [LockManager::lockOnBehalf()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L275-L294)
```solidity
        function lockOnBehalf(
        address _tokenContract,
        uint256 _quantity,
        address _onBehalfOf
    )
        external
        payable
        notPaused
        onlyActiveToken(_tokenContract)
        onlyConfiguredToken(_tokenContract)
        nonReentrant
    {
        address tokenOwner = msg.sender;
        address lockRecipient = msg.sender;
        if (_onBehalfOf != address(0)) {
            lockRecipient = _onBehalfOf;
        }

        _lock(_tokenContract, _quantity, tokenOwner, lockRecipient);
    }
```
Location: [LockManager::_lock()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L311-L398)

```solidity
    function _lock(
        address _tokenContract,
        uint256 _quantity,
        address _tokenOwner,
        address _lockRecipient
    ) private {

        // SNIPPED 
        
        lockedToken.lastLockTime = uint32(block.timestamp);
        lockedToken.unlockTime =
            uint32(block.timestamp) +
            uint32(_lockDuration);
    }
```
---

## <a name="m-01">[M-01]</a> Players can gain more NFTs benefiting from that past remainder in subsequent locks

* **Severity**: Medium
* Source: [Players can gain more NFTs benefiting from that past remainder in subsequent locks](https://github.com/code-423n4/2024-05-munchables-findings/issues/73)

---
### Description
The remaining locked tokens continue to accumulate after the player unlocks their tokens, whether partially or fully.

As a result, the player benefits by retrieving more NFTs when they lock their tokens again during the lock drop period or in the next lock drop period.

Location: [LockManager::_lock()L344 - L380](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L344-L380)

```solidity 

    uint256 quantity = _quantity + lockedToken.remainder;
    uint256 remainder;
    uint256 numberNFTs;
    uint32 _lockDuration = playerSettings[_lockRecipient].lockDuration;

    // SNIPPED

    if (    
        lockdrop.start <= uint32(block.timestamp) &&
        lockdrop.end >= uint32(block.timestamp)
    ) {
        if (
            _lockDuration < lockdrop.minLockDuration ||
            _lockDuration >
            uint32(configStorage.getUint(StorageKey.MaxLockDuration))
        ) revert InvalidLockDurationError();
        if (msg.sender != address(migrationManager)) {
            // calculate number of nfts
-->         remainder = quantity % configuredToken.nftCost; 
            numberNFTs = (quantity - remainder) / configuredToken.nftCost;

            if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

            // Tell nftOverlord that the player has new unopened Munchables
            nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
        }
    }

  // SNIPPED 

->  lockedToken.remainder = remainder;
    lockedToken.quantity += _quantity;
```

Location: [LockManager::unlock()L401 - L427](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L401-L427)
```solidity 
    function unlock(
        address _tokenContract,
        uint256 _quantity
    ) external notPaused nonReentrant {
        LockedToken storage lockedToken = lockedTokens[msg.sender][
            _tokenContract
        ];
        if (lockedToken.quantity < _quantity)
            revert InsufficientLockAmountError();
        if (lockedToken.unlockTime > uint32(block.timestamp))
            revert TokenStillLockedError();

        // force harvest to make sure that they get the schnibbles that they are entitled to
        accountManager.forceHarvest(msg.sender);

        lockedToken.quantity -= _quantity;

        // send token
        if (_tokenContract == address(0)) {
            payable(msg.sender).transfer(_quantity);
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity);
        }

        emit Unlocked(msg.sender, _tokenContract, _quantity);
    }
```
---
### Impact
During the lock drop period or in the subsequent lock drop period, the player retrieves more NFTs in proportion to the quantity locked. The remaining tokens from previous locks enhance the new locking quantity when relocked until the end of the lock period.

Moreover, it probably assumes that the `lockDuration` can either be **equal or greater than** the lock drop duration (`lockDuration >= lockdrop.start - lockdrop.end`), allowing players to reenter by each lock drop, or significantly **less than** the lock drop duration (`lockDuration < lockdrop.start - lockdrop.end`), enabling players to reenter during the same lockdrop.

Therefore, the likelihood of the issue is based on how offset players can reenter and take advantage of the remainders.

The following core functions take an effect of this issue:
* [_lock()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L311-L398)
* [lockOnBehalf()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L275-L294)
* [lock()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L297-L309)
* [unlock()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L401-L427)

---
### Proof of Concept
#### Prerequisite
  - The `LockManager` contract has zero balance. 
  - The actions taken during the any active lock drop period.
  - Using the `address(0)` as a token contract represents locking the native token
    - `_tokenContract` = `address(0)`
  - Assuming the configuration of `nftCost`of the `address(0)` is 3 ethers (**3e18**)
    - `configuredTokens[_tokenContract: address(0)`] = `3e18`
---
#### Partially Unlocking Case
1. Lock **10e18** (`_quantity`) tokens
as [`uint256 quantity = _quantity + lockedToken.remainder :: L344`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L344)

The state updates as follows:
* `quantity` = 10e18(`_quantity`) + 0 (`lockedToken.remainder`) = **10e18** 
* `remainder` = 10e18 % 3e18 = 1e18
* `numberNFTs` = (10e18 - 1e18) / 3e18 = **3**

Updates to `lockedToken` struct and `LockManager` contract balance:
* `lockedToken.remainder = remainder` = **1e18**
* `lockedToken.quantity += _quantity` = 0 + 10e18 = **10e18**
* `address(LockManager).balance` = **10e18**

📍 This shows that locking of **10e18** tokens can retrieve **3** NFTs.

```solidity 
Location: LockManager.sol::_lock()

    // add remainder from any previous lock
--> L344:   uint256 quantity = _quantity + lockedToken.remainder;
            uint256 remainder;
            uint256 numberNFTs;
            uint32 _lockDuration = playerSettings[_lockRecipient].lockDuration;

    // SNIPPED

    if (    
        lockdrop.start <= uint32(block.timestamp) &&
        lockdrop.end >= uint32(block.timestamp)
    ) {
        if (
            _lockDuration < lockdrop.minLockDuration ||
            _lockDuration >
            uint32(configStorage.getUint(StorageKey.MaxLockDuration))
        ) revert InvalidLockDurationError();
        if (msg.sender != address(migrationManager)) {
            // calculate number of nfts
--> L363:   remainder = quantity % configuredToken.nftCost; 
            numberNFTs = (quantity - remainder) / configuredToken.nftCost;

            if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

            // Tell nftOverlord that the player has new unopened Munchables
            nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
        }
    }

  // SNIPPED 

-> L379 lockedToken.remainder = remainder;
        lockedToken.quantity += _quantity;
```

🕔 After the `lockDuration` has passed, the player can unlock the locked tokens.

2. Unlock **5e18** tokens **partially**:
* [`L416::lockedToken.quantity -= _quantity`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L416) = 10e18 - 5e18 = **5e18**
* `address(LockManager).balance` = **5e18**

3. Lock **5e18** tokens back again
as [`uint256 quantity = _quantity + lockedToken.remainder (prev remainder#1);`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L344)
* `quantity` = 5e18(`_quantity`) + 1e18 (`lockedToken.remainder`) = **6e18** 
* `remainder` = 6e18 % 3e18 = 0
* `numberNFTs` = (6e18 - 0) / 3e18 = **2**

Updates to `lockedToken` struct and `LockManager` contract balance:
* `lockedToken.remainder = remainder` = **0**
* `lockedToken.quantity += _quantity` = 5e18 + 5e18 = **10e18**
* `address(LockManager).balance` = **10e18**

📍 In total, the player retrieves **5** NFTs (**3** from the first locking + **2** from the second locking) by locking **10e18** tokens, benefiting from the remainder of the previous locking.

---

#### Fully Unlocking Case
---------------------------
1. Lock **13e18** (`_quantity`) tokens
as [`uint256 quantity = _quantity + lockedToken.remainder :: L344`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L344)

The state updates as follows:
* `quantity` = 13e18(`_quantity`) + 0 (`lockedToken.remainder`) = **10e18** 
* `remainder` = 13e18 % 3e18 = 1e18
* `numberNFTs` = (13e18 - 1e18) / 3e18 = **4** 📍

Updates to `lockedToken` struct and `LockManager` contract balance:
* `lockedToken.remainder = remainder` = **1e18** 📍
* `lockedToken.quantity += _quantity` = 0 + 13e18 = **13e18** 📍
* `address(LockManager).balance` = **13e18** 📍

📍 This shows that locking of **13e18** tokens can retrieve **4** NFTs.

🕔 After the `lockDuration` has passed, the player can unlock the locked tokens.

2. Unlock **13e18** tokens **fully**:
* [`L416::lockedToken.quantity -= _quantity`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L416) = 13e18 - 13e18 = **0** 📍
* `address(LockManager).balance` = **0** 📍

3. Lock **11e18** tokens
as [`uint256 quantity = _quantity + lockedToken.remainder (prev remainder#1);`](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L344)
* `quantity` = 11e18(`_quantity`) + 1e18 (`lockedToken.remainder`) = **12e18** 
* `remainder` = 12e18 % 3e18 = 0
* `numberNFTs` = (12e18 - 0) / 3e18 = **4** 📍

Updates to `lockedToken` and `LockManager` contract balance:
* `lockedToken.remainder = remainder` = **0** 📍
* `lockedToken.quantity += _quantity` = 0 + 11e18 = **11e18** 📍
* `address(LockManager).balance` = **11e18** 📍

📍 In total, the player retrieves **4** new NFTs from locking 11e18 tokens. Compared to the first locking of **13e18** tokens, the second lock yields similar power to gain **4** NFTs, benefiting from the remainder of the previous locking round.

---
### Tools Used
* Manual Review

---
### Recommended Mitigation Steps
Manage the remaining tokens when the player unlocks locked tokens. Alternatively, use other methods to handling the total amount of NFTs and locked quantity during the lock drop.

---

## <a name="m-02">[M-02]</a> Inconsistent conditions for approving and disapproving the proposal price

* **Severity**: Medium
* Source: [Inconsistent conditions for approving and disapproving the proposal price](https://github.com/code-423n4/2024-05-munchables-findings/issues/554)

---
### Description
The approval and disapproval proposer price mechanism does not allow the `PriceFeed_X` that was approved first to become disapproved. 

However, this logic does not implement the vice versa scenario, as the mechanism allows `PriceFeed_X` that was disapproved first to become approved afterward. 

This results in an inconsistent mechanism and incorrectly tracks `usdUpdateProposal.approvalsCount` and `usdUpdateProposal.disapprovalsCount`.

Location: [LockManager::approveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L177-L207)
```solidity
File: ..
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
---

## <a name="l-01">[L-01]</a> Players can set their `lockDuration` less than the minimum lock duration

* **Severity**: Low
* Source: [QA-Report: Players can set their `lockDuration` less than the minimum lock duration](https://github.com/code-423n4/2024-05-munchables-findings/issues/74)

---
### Description
The [setLockDuration()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272) allows player to set their `lockDuration` less than the `lockdrop.minLockDuration`

Location: [LockManager::setLockDuration()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272)

```solidity
    function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();

        playerSettings[msg.sender].lockDuration = uint32(_duration);
        // update any existing lock
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                // check they are not setting lock time before current unlocktime
                if (
                    uint32(block.timestamp) + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }

                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
                    .lastLockTime;
                lockedTokens[msg.sender][tokenContract].unlockTime =
                    lastLockTime +
                    uint32(_duration);
            }
        }

        emit LockDuration(msg.sender, _duration);
    }
```
---