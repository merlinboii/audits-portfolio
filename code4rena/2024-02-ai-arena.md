# AI Arena Report
AI Arena is a PvP platform fighting game where the fighters are AIs that were trained by humans. In the web3 version of our game, these fighters are tokenized via the FighterFarm.sol smart contract. Each fighter NFT within this smart contract contains the following:

Physical attributes: Determines the visual appearance
Generation: This primarily affects the visual appearance
Weight: Determines the battle attributes
Element: Determines its special abilities
Fighter Type: Indicates whether its a regular Champion or Dendroid
Model Data: Comprising of the model type and model hash
Players are able to enter their NFT fighters into ranked battle to earn rewards in our native token $NRN. Our token is an ERC20 token, as defined in the Neuron.sol smart contract. During deployment, we grant our RankedBattle.sol smart contract the MINTER and STAKER roles in order to facilitate our reward system. Additionally, the FighterFarm.sol and GameItems.sol smart contracts are granted the SPENDER role to allow for in-game purchases with our native token.

Players are only able to earn $NRN in our game by staking their tokens and winning. However, it is important to note that it is possible for players to lose part of their stake if they perform poorly. Additionally, to level the playing field, we take the square root of the amount staked to calculate the stakingFactor, which is used in the points calculation after each ranked match. To learn more about our reward mechanism, please click here.

Lastly, each wallet has voltage that it has to manage. Every 24 hours from the start of their first initiated match for the day, voltage will be replenished back to 100. Each ranked battle costs 10 voltage units. If a player runs out of voltage they either have to wait until it naturally replenishes or they can purchase a battery from our GameItems.sol smart contract. Each battery will fill voltage back to 100.

NOTE: Our core game logic runs off-chain via our servers. We essentially use our game server as the oracle for ranked match results.

* **nSLOC**: 1271 
* **Contest details**: https://code4rena.com/audits/2024-02-ai-arena

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: [Severity Categorization | Code4rena](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[H-01](#h-01)| `ERC1155::safeBatchTransferFrom()` can bypass the transfer of non-transferable game items |🍅|
|[H-02](#h-02)| The overload `ERC721::safeTransferFrom()` can bypass the transfer of staking fighters |🍅|
|[H-03](#h-03)| Modulo zero revert on `FighterFarm::_createNewFighter()` for the next generations of fighters. |🍅|
|[H-04](#h-04)| Predictable fighter rerolled traits and exceeding rerolls `maxRerollsAllowed` in `FighterFarm::reRoll()` |🍅|
|[M-01](#m-01)| `UNSTAKED` fighters can have a staking balance, bypassing the condition check in staking and fighter transfer verification. |🍋|
|[M-02](#m-02)| Picked winners potentially unable to claim their rewards |🍋|
|[M-03](#m-03)| Lack of resetting accessible of crucial roles |🍋|
|[M-04](#m-04)| Lack of role revocation functionality in `Neuron` Contract |🍋|
|[QA-Report](#qa)| Low and Informational findings |🫑,🫐|
|[Gas-Report](#gas)| Gas Optimizations |⛽️|

---

## <a name="h-01">[H-01]</a> `ERC1155::safeBatchTransferFrom()` can bypass the transfer of non-transferable game items

* **Severity**: High
* Source: [`ERC1155::safeBatchTransferFrom()` can bypass the transfer of non-transferable game items](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1083)

---
### Description
the players can transfer **non-transferable** game items using [ERC1155::safeBatchTransferFrom()](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.0/contracts/token/ERC1155/ERC1155.sol#L134-L146), bypassing this verification shown in the [GameItems::safeTransferFrom()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L291-L303).

```solidity
require(allGameItemAttributes[tokenId].transferable);
```

The `ERC1155::safeBatchTransferFrom()@v4.7.0`

```solidity 
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public virtual override {
        require(
            from == _msgSender() || isApprovedForAll(from, _msgSender()),
            "ERC1155: caller is not token owner nor approved"
        );
        _safeBatchTransferFrom(from, to, ids, amounts, data);
    }
```

---
### Impact
The ability to bypass the verification of transferable game items, allows players to possess a specific quantity of items to progress through the game.

For example, the protocol team might choose to temporarily mark battery items as **non-transferable** to prevent the dumping of batteries from alternate accounts for replenishing and then transferring the items to the main player account for initiating battles.

---
### Proof of Concept
**Setup**
* Put the snippet below into the protocol test suite: `test/GameItems.t.sol` 
* Run test:
`forge test --mt testBypassTransferGameItems -vvv`


The full coded PoC shown below: 
<details>
  <summary>Details PoC</summary>

```solidity
    function testBypassTransferGameItems() public {
        (, ,bool transferable, , ,) = _gameItemsContract.allGameItemAttributes(0);
        console.log("Initial Battery item transferable flag: %s\n", transferable);

        //update battery items to be non-transferable
        _gameItemsContract.adjustTransferability(0, false);
        console.log("=== Admin adjust battery to non-transferable  ===");
         (, ,transferable, , ,) = _gameItemsContract.allGameItemAttributes(0);
        console.log("Battery item transferable flag: %s\n", transferable);
        uint256 dailyAllowance = 10;
        
        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");
        _fundUserWith4kNeuronByTreasury(alice);

        // mint the battery items
        vm.startPrank(alice);
        _gameItemsContract.mint(0, dailyAllowance);
        assertEq(_gameItemsContract.balanceOf(alice, 0), dailyAllowance);
        console.log("=== Alice mint Battery  ===");
        console.log("Alice Battery items balance: %s", _gameItemsContract.balanceOf(alice, 0));
        console.log("Bob Battery items balance: %s\n", _gameItemsContract.balanceOf(bob, 0));

        // alice attemp to transfer to Bob
        uint256 transferAmount = _gameItemsContract.balanceOf(alice, 0);

            // Fail on `safeTransferFrom`
        vm.expectRevert();
        _gameItemsContract.safeTransferFrom(alice, bob, 0, transferAmount, "");

            // Success on `safeBatchTransferFrom`
        uint256[] memory craftIds = new uint256[](1);
        craftIds[0] = 0;
        uint256[] memory craftAmounts = new uint256[](1);
        craftAmounts[0] = transferAmount;

        _gameItemsContract.safeBatchTransferFrom(alice, bob, craftIds, craftAmounts, "");
        vm.stopPrank();

        console.log("=== Alice transfer Battery to Bob via safeBatchTransferFrom() ===");
        console.log("Alice Battery items balance: %s", _gameItemsContract.balanceOf(alice, 0));
        console.log("Bob Battery items balance: %s\n", _gameItemsContract.balanceOf(bob, 0));

        console.log("=== Transfer of non-tranferable items SUCCESS ===");
        assertEq(_gameItemsContract.balanceOf(bob, 0), transferAmount);
    }
```
</details>

Results of running the test:
```
Running 1 test for test/GameItems.t.sol:GameItemsTest
[PASS] testBypassTransferGameItems() (gas: 197141)
Logs:
  Initial Battery item transferable flag: true

  === Admin adjust battery to non-transferable  ===
  Battery item transferable flag: false

  === Alice mint Battery  ===
  Alice Battery items balance: 10
  Bob Battery items balance: 0

  === Alice transfer Battery to Bob via safeBatchTransferFrom() ===
  Alice Battery items balance: 0
  Bob Battery items balance: 10

  === Transfer of non-tranferable items SUCCESS ===

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.58ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

---
### Tools Used
* Manual Review
* Foundry

---
### Recommended Mitigation Steps
Recommend overriding the `ERC1155::_beforeTokenTransfer()` to verify transferable items in every item transfer.

Overriding `ERC1155::_beforeTokenTransfer()` will execute the hook and verify transferable items in both `safeTransferFrom()` and `safeBatchTransferFrom()` executions. 

Therefore, applying the recommended code can remove the need for overriding `GameItems.safeTransferFrom()`.

```diff
File: GameItems.sol

+    function _beforeTokenTransfer(
+        address operator,
+        address from,
+        address to,
+        uint256[] memory ids,
+        uint256[] memory amounts,
+        bytes memory data
+    ) internal override(ERC1155) {
+        if (from != address(0) && to != address(0)){
+            for (uint256 i = 0; i < ids.length; ++i) {
+                require(allGameItemAttributes[ids[i]].transferable);
+            }
+        }
+    }
```
---

## <a name="h-02">[H-02]</a> The overload `ERC721::safeTransferFrom()` can bypass the transfer of staking fighters

* **Severity**: High
* Source: [The overload `ERC721::safeTransferFrom()` can bypass the transfer of staking fighters](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1084)

---
### Description
When the players stake `NRN` for their fighters, those fighters are flagged as being stake and **unable to transfer**. 

However, the players can transfer fighters while they are staked by using overload [ERC721::safeTransferFrom()](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.0/contracts/token/ERC721/ERC721.sol#L175-L183), bypassing this verification shown in the [FighterFarm::transferFrom()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L333-L348) and [FighterFarm::safeTransferFrom()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L350-L365).

* The [FighterFarm::_ableToTransfer()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L539-L545)

```solidity
File: FighterFarm.sol

    function _ableToTransfer(uint256 tokenId, address to) private view returns(bool) {
        return (
          _isApprovedOrOwner(msg.sender, tokenId) &&
          balanceOf(to) < MAX_FIGHTERS_ALLOWED &&
          !fighterStaked[tokenId]
        );
    }
```

* The overload `ERC721::safeTransferFrom()@v4.7.0`

```solidity
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) public virtual override {
        require(_isApprovedOrOwner(_msgSender(), tokenId), "ERC721: caller is not token owner nor approved");
        _safeTransfer(from, to, tokenId, data);
    }
```

---
### Impact
Bypassing the fighter transfer verification that is designed to prevent the transfer of fighters when they are being staked, leading to conflicts in game mechanics and enabling the staked fighter to be traded in the secondary market.

---
### Proof of Concept
**Setup**
* Put the snippet below into the protocol test suite: `test/FighterFarm.t.sol` 
* Run test:
`forge test --mt testTransferringFighterWhileStaked_SUCCESS -vvv`


The full coded PoC shown below: 
<details>
  <summary>Details PoC</summary>

```solidity
    function testTransferringFighterWhileStaked_SUCCESS() public {
        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");
        console.log("=== Addresses ===");
        console.log("Alice: %s", alice);
        console.log("Bob: %s\n", bob);

        _mintFromMergingPool(alice);
        _fighterFarmContract.addStaker(_ownerAddress);

        // flag fighter as being staked
        vm.prank(_ownerAddress);
        _fighterFarmContract.updateFighterStaking(0, true);
        assertEq(_fighterFarmContract.fighterStaked(0), true);

        console.log("Staking status of Fighter ID: 0 : %s\n", _fighterFarmContract.fighterStaked(0));
        
        vm.prank(alice);
        // using `ER721::safeTransferFrom(address,address,uint256,bytes)`
        _fighterFarmContract.safeTransferFrom(alice, bob, 0, "");
        assertEq(_fighterFarmContract.ownerOf(0) != alice, true);
        assertEq(_fighterFarmContract.ownerOf(0), bob);

        console.log("=== Alice transfer the staking Fighter:0 to Bob ===");
        console.log("Owner of Fighter:0 : %s (%s)", _fighterFarmContract.ownerOf(0), _fighterFarmContract.ownerOf(0) == bob? "Bob" : (_fighterFarmContract.ownerOf(0) == alice? "Alice" : "None"));
    }
```
</details>

Results of running the test:
```
Running 1 test for test/FighterFarm.t.sol:FighterFarmTest
[PASS] testTransferringFighterWhileStaked_SUCCESS() (gas: 511035)
Logs:
  === Addresses ===
  Alice: 0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea
  Bob: 0x4dBa461cA9342F4A6Cf942aBd7eacf8AE259108C

  Staking status of Fighter ID: 0 : true

  === Alice transfer the staking Fighter:0 to Bob ===
  Owner of Fighter:0 : 0x4dBa461cA9342F4A6Cf942aBd7eacf8AE259108C (Bob)

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.99ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

---
### Tools Used
* Manual Review
* Foundry

---
### Recommended Mitigation Steps
Recommend applying the verification into the override `FighterFarm::_beforeTokenTransfer()` to verify transferable fighters in every fighter transfer.

Applying the recommended code below will execute the hook and verify transferable fighters in `transferFrom()`, `safeTransferFrom()`, and overloaded `safeTransferFrom()` executions. 

Therefore, applying the recommended code can remove the need for overriding `FighterFarm.transferFrom()` and `FighterFarm.safeTransferFrom()`.

```diff
File: FighterFarm.sol

    function _beforeTokenTransfer(address from, address to, uint256 tokenId)
        internal
        override(ERC721, ERC721Enumerable)
    {   
+        if(from != address(0) && to != address(0)){
+            require(_ableToTransfer(tokenId, to), "unable to transfer");
+        }
        super._beforeTokenTransfer(from, to, tokenId);
    }
```
---

## <a name="h-03">[H-03]</a> Modulo zero revert on `FighterFarm::_createNewFighter()` for the next generations of fighters.

* **Severity**: High
* Source: [Modulo zero revert on `FighterFarm::_createNewFighter()` for the next generations of fighters.](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1088)

---
### Description
There is no update to the `FighterFarm::numElements[]` when a new generation of fighters occurs via [FighterFarm::incrementGeneration()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L129-L134) as the `FighterFarm::numElements[]` is hardcoded to support only `generation:0` at during deployment.

```solidity
File: FighterFarm.sol

    constructor(address ownerAddress, address delegatedAddress, address treasuryAddress_)
        ERC721("AI Arena Fighter", "FTR")
    {
        _ownerAddress = ownerAddress;
        _delegatedAddress = delegatedAddress;
        treasuryAddress = treasuryAddress_;
-->     numElements[0] = 3;
    } 
```


```solidity
File: FighterFarm.sol

    function incrementGeneration(uint8 fighterType) external returns (uint8) {
        require(msg.sender == _ownerAddress);
        generation[fighterType] += 1;
        maxRerollsAllowed[fighterType] += 1;
        return generation[fighterType];
    }
```

Let's imagine that when the generation of the `fighterType:0 (champions)` is increased to `generation:1` => `generation[fighterType] == 1`.

There is no `numElements[1]` support, making 
`numElements[generation[fighterType]]` => `numElements[1] == 0`
which consequently creates a `modulo by zero` revert in 
```solidity
    uint256 element = dna % 0; //REVERT
```

---
### Impact
The transaction reverts on the fighter's `element` calculation in [FighterFarm::_createFighterBase()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L458-L474)

Consequently, it's unable to create the new generation fighter in case the input `customAttributes == 100`, triggering the execution of [FighterFarm::_createFighterBase()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L458-L474).

The following functions are impacted by this issue:
* [FighterFarm::claimFighters()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L191-L222)
* [FighterFarm::redeemMintPass()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L233-L262)
* [FighterFarm::mintFromMergingPool()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L313-L331)
* [FighterFarm::reRoll()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L370-L391)
* [FighterFarm::_createNewFighter()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L484-L531)


```solidity
File: FighterFarm.sol

    function _createFighterBase(
        uint256 dna, 
        uint8 fighterType
    ) 
        private 
        view 
        returns (uint256, uint256, uint256) 
    {
-->     uint256 element = dna % numElements[generation[fighterType]];
        uint256 weight = dna % 31 + 65;
        uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
        return (element, weight, newDna);
    }
```

---
### Proof of Concept
**Setup**
* Put the snippet below into the protocol test suite: `test/FighterFarm.t.sol` 
* Run test:
`forge test --mt testCreateFighterRevertWhenNoSupportNumElements -vvv`


The full coded PoC shown below: 
<details>
  <summary>Details PoC</summary>

* The PoC code

```solidity
    function testCreateFighterRevertWhenNoSupportNumElements() public {
        // increase the generation of both fighter type: 0
        _fighterFarmContract.incrementGeneration(0);
        assertEq(_fighterFarmContract.generation(0), 1);
        assertEq(_fighterFarmContract.generation(1), 0);

        uint8[2] memory numToMint = [1, 0];
        bytes memory claimSignature = abi.encodePacked(
            hex"407c44926b6805cf9755a88022102a9cb21cde80a210bc3ad1db2880f6ea16fa4e1363e7817d5d87e4e64ba29d59aedfb64524620e2180f41ff82ca9edf942d01c"
        );
        string[] memory claimModelHashes = new string[](1);
        claimModelHashes[0] = "ipfs://bafybeiaatcgqvzvz3wrjiqmz2ivcu2c5sqxgipv5w2hzy4pdlw7hfox42m";

        string[] memory claimModelTypes = new string[](1);
        claimModelTypes[0] = "original";

        // Right sender of signature should be able to claim fighter => revert
        vm.expectRevert(stdError.divisionError);
        _fighterFarmContract.claimFighters(numToMint, claimSignature, claimModelHashes, claimModelTypes);

    }
```
</details>

Results of running the test:
```
Running 1 test for test/FighterFarm.t.sol:FighterFarmTest
[PASS] testCreateFighterRevertWhenNoSupportNumElements() (gas: 91208)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.60ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

---
### Tools Used
* Manual Review
* Foundry

---
### Recommended Mitigation Steps
Recommend updating the `FighterFarm::numElements[]` to support all generation increases.

---

## <a name="h-04">[H-04]</a> Predictable fighter rerolled traits and exceeding rerolls `maxRerollsAllowed` in `FighterFarm::reRoll()`

* **Severity**: High
* Source: [Predictable fighter rerolled traits and exceeding rerolls `maxRerollsAllowed` in `FighterFarm::reRoll()`](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1093)

---
### Description
The [FighterFarm::reRoll()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L370-L391)  allows the player to specify the `fighterType` to the [FighterFarm::reRoll()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L370-L391), which can lead to predictable fighter rerolled traits and
ability to reroll fighters more than the maximum allowance for each `fighterType`.

```solidity
File: FighterFarm.sol

    function reRoll(uint8 tokenId, uint8 fighterType) public {
        require(msg.sender == ownerOf(tokenId));
-->     require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
        require(_neuronInstance.balanceOf(msg.sender) >= rerollCost, "Not enough NRN for reroll");

        _neuronInstance.approveSpender(msg.sender, rerollCost);
        bool success = _neuronInstance.transferFrom(msg.sender, treasuryAddress, rerollCost);
        if (success) {
            numRerolls[tokenId] += 1;
            uint256 dna = uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])));
-->         (uint256 element, uint256 weight, uint256 newDna) = _createFighterBase(dna, fighterType);
            fighters[tokenId].element = element;
            fighters[tokenId].weight = weight;
-->         fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
                newDna,
                generation[fighterType],
                fighters[tokenId].iconsType,
                fighters[tokenId].dendroidBool
            );
            _tokenURIs[tokenId] = "";
        }
    } 
```

```solidity
File: FighterFarm.sol

    function _createFighterBase(
        uint256 dna, 
        uint8 fighterType
    ) 
        private 
        view 
        returns (uint256, uint256, uint256) 
    {
        uint256 element = dna % numElements[generation[fighterType]];
        uint256 weight = dna % 31 + 65;
-->     uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
        return (element, weight, newDna);
    }
```

```solidity
File: AiArenaHelper.sol

    function createPhysicalAttributes(
        uint256 dna, 
        uint8 generation, 
        uint8 iconsType, 
        bool dendroidBool
    ) 
        external 
        view 
        returns (FighterOps.FighterPhysicalAttributes memory) 
    {
        if (dendroidBool) {
            return FighterOps.FighterPhysicalAttributes(99, 99, 99, 99, 99, 99);
        } else {
            uint256[] memory finalAttributeProbabilityIndexes = new uint[](attributes.length);

            uint256 attributesLength = attributes.length;
            for (uint8 i = 0; i < attributesLength; i++) {
                if (
                  i == 0 && iconsType == 2 || // Custom icons head (beta helmet)
                  i == 1 && iconsType > 0 || // Custom icons eyes (red diamond)
                  i == 4 && iconsType == 3 // Custom icons hands (bowling ball)
                ) {
                    finalAttributeProbabilityIndexes[i] = 50;
                } else {
-->                 uint256 rarityRank = (dna / attributeToDnaDivisor[attributes[i]]) % 100;
                    uint256 attributeIndex = dnaToIndex(generation, rarityRank, attributes[i]);
                    finalAttributeProbabilityIndexes[i] = attributeIndex;
                }
            }
            return FighterOps.FighterPhysicalAttributes(
                finalAttributeProbabilityIndexes[0],
                finalAttributeProbabilityIndexes[1],
                finalAttributeProbabilityIndexes[2],
                finalAttributeProbabilityIndexes[3],
                finalAttributeProbabilityIndexes[4],
                finalAttributeProbabilityIndexes[5]
            );
        }
    }
```

---
### Impact
Impact of the Issue:

* Predictable rerolled fighter attributes.

* Ability to reroll fighters more than the maximum allowance for each `fighterType`.

---
### Proof of Concept
Let's assume that 
* The current generation of each fighter type is 0.
* Alice has 1 champion fighter 
    * `id: 0`
    * `fighterType:0`
    * `dendroidBool: false`.
* Global [FighterFarm::maxRerollsAllowed[fighterType:0]: 3](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L36).

Alice rerolls her fighter by passing the `fighterType as 1` (dendroid type), which is inconsistent with her current fighter type.

```
% FighterFarm.reRoll(tokenId: 0, fighterType: 1) 
```

The sequence and result of execution will be as follows:
1. The `maxRerollsAllowed[fighterType: 1]` will be checked instead of `maxRerollsAllowed[fighterType: 0]` which is the actual type of the given fighter.


```solidity
File: FighterFarm.sol

    function reRoll(uint8 tokenId, uint8 fighterType) public {
        require(msg.sender == ownerOf(tokenId));
-->     require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);

    // SNIPPED

    }
```

2. The `newDna` will be set to 1 

```
uint256 newDna = fighterType(1) == 0 ? dna : uint256(fighterType(1));
uint256 newDna = uint256(fighterType(1))
```

```solidity
File: FighterFarm.sol

    function reRoll(uint8 tokenId, uint8 fighterType) public {

        // SNIPPED

        if (success) {
            numRerolls[tokenId] += 1;
            uint256 dna = uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])));
-->         (uint256 element, uint256 weight, uint256 newDna) = _createFighterBase(dna, fighterType);

            // SNIPPED

        }
    } 
```

```solidity
File: FighterFarm.sol

    function _createFighterBase(
        uint256 dna, 
        uint8 fighterType
    ) 
        private 
        view 
        returns (uint256, uint256, uint256) 
    {
        uint256 element = dna % numElements[generation[fighterType]];
        uint256 weight = dna % 31 + 65;
-->     uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
        return (element, weight, newDna);
    }
```

3. This results in the new rerolled fighter for her fighter always being represented as:
```
FighterOps.FighterPhysicalAttributes(1, 1, 1, 1, 1, 1)
```
as calling the [AiArenaHelper.createPhysicalAttributes()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/AiArenaHelper.sol#L83-L121) as follows:

```
fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
                newDna: 1, 
                generation[fighterType],
                fighters[tokenId].iconsType,
                fighters[tokenId].dendroidBool
            );
```

```solidity
File: FighterFarm.sol

    function reRoll(uint8 tokenId, uint8 fighterType) public {
        
        // SNIPPED

        if (success) {
            numRerolls[tokenId] += 1;
            uint256 dna = uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])));
            (uint256 element, uint256 weight, uint256 newDna) = _createFighterBase(dna, fighterType);
            fighters[tokenId].element = element;
            fighters[tokenId].weight = weight;
-->         fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
                newDna, // 1
                generation[fighterType],
                fighters[tokenId].iconsType,
                fighters[tokenId].dendroidBool
            );
            _tokenURIs[tokenId] = "";
        }
    } 
```

```solidity
File: AiArenaHelper.sol

    function createPhysicalAttributes(
        uint256 dna, 
        uint8 generation, 
        uint8 iconsType, 
        bool dendroidBool
    ) 
        external 
        view 
        returns (FighterOps.FighterPhysicalAttributes memory) 
    {
        if (dendroidBool) {
            return FighterOps.FighterPhysicalAttributes(99, 99, 99, 99, 99, 99);
        } else {
            uint256[] memory finalAttributeProbabilityIndexes = new uint[](attributes.length);

            uint256 attributesLength = attributes.length;
            for (uint8 i = 0; i < attributesLength; i++) {
                if (
                  i == 0 && iconsType == 2 || // Custom icons head (beta helmet)
                  i == 1 && iconsType > 0 || // Custom icons eyes (red diamond)
                  i == 4 && iconsType == 3 // Custom icons hands (bowling ball)
                ) {
                    finalAttributeProbabilityIndexes[i] = 50;
                } else {
-->                 uint256 rarityRank = (dna / attributeToDnaDivisor[attributes[i]]) % 100;
                    // uint256 rarityRank = (1 / attributeToDnaDivisor[attributes[i]]) % 100;
                    // uint256 rarityRank = 0
                    uint256 attributeIndex = dnaToIndex(generation, rarityRank, attributes[i]);
                    finalAttributeProbabilityIndexes[i] = attributeIndex;
                }
            }
            return FighterOps.FighterPhysicalAttributes(
                finalAttributeProbabilityIndexes[0],
                finalAttributeProbabilityIndexes[1],
                finalAttributeProbabilityIndexes[2],
                finalAttributeProbabilityIndexes[3],
                finalAttributeProbabilityIndexes[4],
                finalAttributeProbabilityIndexes[5]
            );
        }
    }

    // SNIPPED

    function dnaToIndex(uint256 generation, uint256 rarityRank, string memory attribute) 
        public 
        view 
        returns (uint256 attributeProbabilityIndex) 
    {
        uint8[] memory attrProbabilities = getAttributeProbabilities(generation, attribute);
        
        uint256 cumProb = 0;
        uint256 attrProbabilitiesLength = attrProbabilities.length;
        for (uint8 i = 0; i < attrProbabilitiesLength; i++) {
            cumProb += attrProbabilities[i];
            if (cumProb >= rarityRank) { // 0 >= 0
                attributeProbabilityIndex = i + 1; // 0+1
                break;
            }
        }
        return attributeProbabilityIndex;
    }
```

---
### Tools Used
* Manual Review

---
### Recommended Mitigation Steps
Recommend updating the `FighterFarm::reRoll()` to use the current `fighterType` of the given fighter in the reroll mechanism. 

```diff
File: FighterFarm.sol

-   function reRoll(uint8 tokenId, uint8 fighterType) public {
+   function reRoll(uint8 tokenId) public {
+       uint8 fighterType = fighters[tokenId].dendroidBool ? 1 : 0;
        require(msg.sender == ownerOf(tokenId));
        require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
        require(_neuronInstance.balanceOf(msg.sender) >= rerollCost, "Not enough NRN for reroll");

        _neuronInstance.approveSpender(msg.sender, rerollCost);
        bool success = _neuronInstance.transferFrom(msg.sender, treasuryAddress, rerollCost);
        if (success) {
            numRerolls[tokenId] += 1;
            uint256 dna = uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])));
            (uint256 element, uint256 weight, uint256 newDna) = _createFighterBase(dna, fighterType);
            fighters[tokenId].element = element;
            fighters[tokenId].weight = weight;
            fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
                newDna,
                generation[fighterType],
                fighters[tokenId].iconsType,
                fighters[tokenId].dendroidBool
            );
            _tokenURIs[tokenId] = "";
        }
    } 
```
---

## <a name="m-01">[M-01]</a> `UNSTAKED` fighters can have a staking balance, bypassing the condition check in staking and fighter transfer verification.

* **Severity**: Medium
* Source: [`UNSTAKED` fighters can have a staking balance, bypassing the condition check in staking and fighter transfer verification.](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1087)

---
### Description
The [RankedBattle::unstakeNRN()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L270-L290) allows the fighters to unstake all their `amountStaked` and be flagged as `UNSTAKED`, while those fighters still hold the remaining `stakeAtRisk` that has the potential to increase their `amountStaked` in their next battles, resulting in `UNSTAKED` fighters potentially having `amountStaked > 0`. 

This allows them to 
1. Having a staking balance without being flagged as `STAKED` status.
2. Bypassing the fighter transfer verification
3. Staking more in other rounds without flagging back to the `STAKED` status.

---
### Impact
Impact of the Issue:

* Bypassing the fighter transfer verification that is designed to prevent the transfer of fighters when they are being staked, leading to conflicts in game mechanics and enabling the staked fighter to be traded in the secondary market.

* Conflicting logic regarding unstaked fighters as the term `UNSTAKED` fighter should refer to the fighters having no remaining stake.

---
### Proof of Concept
Consider the following scenario
0. Let's assume that the `fighter01` is being staked and then it play through the battles create some `stakeAtRisk`.

1. The owner of the `fighter01` unstake **all** `amountStaked[fighter01]` by calling [RankedBattle::unstakeNRN()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L270-L290) and leaving some `stakeAtRisk` remaining, leading to that fighter will be flagged as `UNSTAKED` and gain the ability to transfer.

```solidity
File: RankedBattle.sol

    function unstakeNRN(uint256 amount, uint256 tokenId) external {
        require(_fighterFarmInstance.ownerOf(tokenId) == msg.sender, "Caller does not own fighter");
        if (amount > amountStaked[tokenId]) {
            amount = amountStaked[tokenId];
        }
        amountStaked[tokenId] -= amount;
        globalStakedAmount -= amount;
        stakingFactor[tokenId] = _getStakingFactor( //@note if user not stake it will be 1
            tokenId, 
            _stakeAtRiskInstance.getStakeAtRisk(tokenId)
        );
        _calculatedStakingFactor[tokenId][roundId] = true;
        hasUnstaked[tokenId][roundId] = true;
        bool success = _neuronInstance.transfer(msg.sender, amount);
        if (success) {
-->          if (amountStaked[tokenId] == 0) {
-->              _fighterFarmInstance.updateFighterStaking(tokenId, false);
            }
            emit Unstaked(msg.sender, amount);
        }
    }
```
2. When the `fighter01` win in the next battle at this round, `amountStaked[fighter01]` will be increased to reclaim its left `stakeAtRisk` through the [RankedBattle::updateBattleRecord()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L322-L349) --> [RankedBattle::_addResultPoints()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L416-L500).

However, the current staking status of the `fighter01` still be `UNSTAKED`, while the `amountStaked[fighter01] > 0`

```solidity
File: RankedBattle.sol

    if (curStakeAtRisk > 0) {
-->     _stakeAtRiskInstance.reclaimNRN(curStakeAtRisk, tokenId, fighterOwner); 
-->     amountStaked[tokenId] += curStakeAtRisk; // increase amountStaked
    }
```
3. As the `fighter01` is already flagged as `UNSTAKED`, and `amountStaked[fighter01] > 0`, when they call [RankedBattle::stakeNRN()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L244-L265) in the next round (the check of unstake status of this round is also passed), the execution will bypass this check and not flag the `fighter01` back to `STAKED`.

```solidity
File: RankedBattle.sol
    function stakeNRN(uint256 amount, uint256 tokenId) external {
        require(amount > 0, "Amount cannot be 0");
        require(_fighterFarmInstance.ownerOf(tokenId) == msg.sender, "Caller does not own fighter");
        require(_neuronInstance.balanceOf(msg.sender) >= amount, "Stake amount exceeds balance");
        require(hasUnstaked[tokenId][roundId] == false, "Cannot add stake after unstaking this round");

        _neuronInstance.approveStaker(msg.sender, address(this), amount);
        bool success = _neuronInstance.transferFrom(msg.sender, address(this), amount);
        if (success) {
-->         if (amountStaked[tokenId] == 0) { //Bypass this check
                _fighterFarmInstance.updateFighterStaking(tokenId, true);
            }
            amountStaked[tokenId] += amount;
            globalStakedAmount += amount;
            stakingFactor[tokenId] = _getStakingFactor(
                tokenId, 
                _stakeAtRiskInstance.getStakeAtRisk(tokenId)
            );
            _calculatedStakingFactor[tokenId][roundId] = true;
            emit Staked(msg.sender, amount);
        }
    }
```

And able to bypass this fighter transfer verification:

```solidity
File: FighterFarm.sol

    function _ableToTransfer(uint256 tokenId, address to) private view returns(bool) {
        return (
          _isApprovedOrOwner(msg.sender, tokenId) &&    //> approval pass
          balanceOf(to) < MAX_FIGHTERS_ALLOWED &&   //> destination not reach the MAX
          !fighterStaked[tokenId]   //> not stake/ not lock
        );
    }
```

**Setup**
* Put the snippet below into the protocol test suite: `test/RankedBattle.t.sol` 
* Run test:
`forge test --mt testIncorrectlyFlagOfStakingStatus -vvv`


The full coded PoC shown below: 
<details>
  <summary>Details PoC</summary>

* The PoC code

```solidity
    function testIncorrectlyFlagOfStakingStatus() public {
        //STAKE
        address player = vm.addr(3);
        _mintFromMergingPool(player);
        _fundUserWith4kNeuronByTreasury(player);
        console.log("== STAKED at round 0 ==");
        vm.prank(player);
        _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, 0); //amount id
        console.log("STAKED Flag #1: %s\n", _fighterFarmContract.fighterStaked(0));

        address player2 = vm.addr(4);
        _mintFromMergingPool(player2);
        _fundUserWith4kNeuronByTreasury(player2);
        vm.prank(player2);
        _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, 1); //amount id

        //ANOTHER player Win => to increase the point in order to set new round
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(1, 50, 0, 1500, false);

        //BATTLE-1-Lost
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(0, 50, 2, 1500, false);

        //Unstake All
        console.log("== UNSTAKED at round 0 ==");
        vm.prank(player);
        _rankedBattleContract.unstakeNRN(3_000 * 10 ** 18, 0);
        console.log("STAKED Flag #2: %s\n", _fighterFarmContract.fighterStaked(0));

        console.log("== _GAME_SERVER_ADDRESS set new round ==\n");
        //BATTLE-2-Win
        vm.prank(address(_GAME_SERVER_ADDRESS));
        _rankedBattleContract.updateBattleRecord(0, 50, 0, 1500, false);
        _rankedBattleContract.setNewRound();

        console.log("== STAKED at round 1 ==");
        vm.prank(player);
        _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, 0);
        console.log("STAKED Flag #3: %s\n", _fighterFarmContract.fighterStaked(0));
    }
```
</details>

Results of running the test:
```
Running 1 test for test/RankedBattle.t.sol:RankedBattleTest
[PASS] testPoC17() (gas: 1646900)
Logs:
  == STAKED at round 0 ==
  STAKED Flag #1: true

  == UNSTAKED at round 0 ==
  STAKED Flag #2: false

  == _GAME_SERVER_ADDRESS set new round ==

  == STAKED at round 1 ==
  STAKED Flag #3: false


Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.68ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

---
### Tools Used
* Manual Review
* Foundry

---
### Recommended Mitigation Steps
I would recommend two approaches:

1. As the current behavior of unstaking allows fighters to unstake all eligible `amountStaked` and can be transferred even if the fighter has `stakeAtRisk` in the current round, which has the potential to increase their `amountStaked` in the next battles.

    I would recommend changing the unstake condition to flag back as `UNSTAKED` by including all potential stake factors: `amountStaked` and `stakeAtRisk` to ensure all staking and potential staking amounts are included to clarify marking as `UNSTAKED`.

```diff
File: RankedBattle.sol

    function unstakeNRN(uint256 amount, uint256 tokenId) external {
        require(_fighterFarmInstance.ownerOf(tokenId) == msg.sender, "Caller does not own fighter");
        if (amount > amountStaked[tokenId]) {
            amount = amountStaked[tokenId];
        }
        amountStaked[tokenId] -= amount;
        globalStakedAmount -= amount;
        stakingFactor[tokenId] = _getStakingFactor(
            tokenId, 
            _stakeAtRiskInstance.getStakeAtRisk(tokenId)
        );
        _calculatedStakingFactor[tokenId][roundId] = true;
        hasUnstaked[tokenId][roundId] = true;
        bool success = _neuronInstance.transfer(msg.sender, amount);
        if (success) {
+           uint256 stakeAtRisk = _stakeAtRiskInstance.getStakeAtRisk(tokenId);
+           if (amountStaked[tokenId] + stakeAtRisk == 0) {
                _fighterFarmInstance.updateFighterStaking(tokenId, false);
            }
            emit Unstaked(msg.sender, amount);
        }
    }
```

This still allows fighters to unstake all eligible `amountStaked`, and it will prevent the fighter from being flagged as `UNSTAKED` and transferred, in case they have only `stakeAtRisk` remaining.

2. In case there is a need to maintain the same behavior.

    I would recommend applying the condition to flag fighters back to `STAKED` whenever the `amountStaked[tokenId]` increases while they hold the `UNSTAKED` flag (this may consume more gas than the first approach).

```diff
File: RankedBattle.sol

    function _addResultPoints(
        uint8 battleResult, 
        uint256 tokenId, 
        uint256 eloFactor, 
        uint256 mergingPortion,
        address fighterOwner
    ) 
        private 
    {
        uint256 stakeAtRisk;
        uint256 curStakeAtRisk;
        uint256 points = 0;

        /// Check how many NRNs the fighter has at risk
        stakeAtRisk = _stakeAtRiskInstance.getStakeAtRisk(tokenId);

        /// Calculate the staking factor if it has not already been calculated for this round 
        if (_calculatedStakingFactor[tokenId][roundId] == false) {
            stakingFactor[tokenId] = _getStakingFactor(tokenId, stakeAtRisk);
            _calculatedStakingFactor[tokenId][roundId] = true;
        }

        /// Potential amount of NRNs to put at risk or retrieve from the stake-at-risk contract
        curStakeAtRisk = (bpsLostPerLoss * (amountStaked[tokenId] + stakeAtRisk)) / 10**4;
        if (battleResult == 0) {
            /// If the user won the match

            /// If the user has no NRNs at risk, then they can earn points
            if (stakeAtRisk == 0) {
                points = stakingFactor[tokenId] * eloFactor;
            }

            /// Divert a portion of the points to the merging pool
            uint256 mergingPoints = (points * mergingPortion) / 100;
            points -= mergingPoints;
            _mergingPoolInstance.addPoints(tokenId, mergingPoints);

            /// Do not allow users to reclaim more NRNs than they have at risk
            if (curStakeAtRisk > stakeAtRisk) {
                curStakeAtRisk = stakeAtRisk;
            }

            /// If the user has stake-at-risk for their fighter, reclaim a portion
            /// Reclaiming stake-at-risk puts the NRN back into their staking pool
            if (curStakeAtRisk > 0) {
                _stakeAtRiskInstance.reclaimNRN(curStakeAtRisk, tokenId, fighterOwner);
                amountStaked[tokenId] += curStakeAtRisk;
+               if (!_fighterFarmInstance.fighterStaked(tokenId)) {
+                   _fighterFarmInstance.updateFighterStaking(tokenId, true);
+               }
            }

            /// Add points to the fighter for this round
            accumulatedPointsPerFighter[tokenId][roundId] += points;
            accumulatedPointsPerAddress[fighterOwner][roundId] += points;
            totalAccumulatedPoints[roundId] += points;
            if (points > 0) {
                emit PointsChanged(tokenId, points, true);
            }
        } else if (battleResult == 2) {

            // SNIPPED

        }
    }
```
---

## <a name="m-02">[M-02]</a> Picked winners potentially unable to claim their rewards

* **Severity**: Medium
* Source: [Picked winners potentially unable to claim their rewards](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1097)

---
### Description
The winners selected by the [MergingPool::pickWinner()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L118-L132) potentially be unable to claim at least some of their rewards (whenever the winner's owner balance is less than the [FighterFarm::MAX_FIGHTERS_ALLOWED](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L33)) if their balance is insufficient for their maximum rewards from their all rewards.

Since the [MergingPool::pickWinner()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L118-L132) selects the winner based on the fighter ID, it allows the same winner's owner to be included multiple times in the `MergingPool::currentWinnerAddresses[]`.

```solidity
File: MergingPool.sol

    function pickWinner(uint256[] calldata winners) external {
        require(isAdmin[msg.sender]);
        require(winners.length == winnersPerPeriod, "Incorrect number of winners");
        require(!isSelectionComplete[roundId], "Winners are already selected");
        uint256 winnersLength = winners.length;
        address[] memory currentWinnerAddresses = new address[](winnersLength);
        for (uint256 i = 0; i < winnersLength; i++) {
-->         currentWinnerAddresses[i] = _fighterFarmInstance.ownerOf(winners[i]);
            totalPoints -= fighterPoints[winners[i]];
            fighterPoints[winners[i]] = 0;
        }
-->     winnerAddresses[roundId] = currentWinnerAddresses;
        isSelectionComplete[roundId] = true;
        roundId += 1;
    }
```

```solidity
File: MergingPool.sol

    function claimRewards(
        string[] calldata modelURIs, 
        string[] calldata modelTypes,
        uint256[2][] calldata customAttributes
    ) 
        external 
    {
        uint256 winnersLength;
        uint32 claimIndex = 0;
        uint32 lowerBound = numRoundsClaimed[msg.sender];
        for (uint32 currentRound = lowerBound; currentRound < roundId; currentRound++) {
            numRoundsClaimed[msg.sender] += 1;
            winnersLength = winnerAddresses[currentRound].length;
-->         for (uint32 j = 0; j < winnersLength; j++) {
                if (msg.sender == winnerAddresses[currentRound][j]) {
-->                 _fighterFarmInstance.mintFromMergingPool(
                        msg.sender,
                        modelURIs[claimIndex],
                        modelTypes[claimIndex],
                        customAttributes[claimIndex]
                    );
                    claimIndex += 1;
                }
            }
        }
        if (claimIndex > 0) {
            emit Claimed(msg.sender, claimIndex);
        }
    }
```

---
### Impact
Winners selected by the [MergingPool::pickWinner()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L118-L132) be unable to claim some or all of their rewards if their balance is insufficient to cover their maximum rewards.

This issue prevents players from receiving rewards due to their balance being unsatisfactory within the claim rewards iteration

---
### Tools Used
* Manual Review

---
### Proof of Concept
Let's consider the following scenario:
1. Bob owns 9 fighters, and two of his fighters are selected as winners for rewards. 
This assumes `MAX_FIGHTERS_ALLOWED` is set to 10, `roundId` is 0, and `winnersPerPeriod` is 2.
```
winnerAddresses[roundId: 0] = [Bob'address, Bob'address];
```

2. Bob tries to claim his rewards through the [MergingPool::claimRewards()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L139-L167)

3. Unfortunately, Bob is unable to claim any rewards due to a revert caused by the require statement in the [FighterFarm::_createNewFighter()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L484-L531).

Imagine within Bob's `MergingPool::claimRewards()` transaction:
* After the 1st iteration of [MergingPool::claimRewards()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L139-L167), Bob's fighter balance becomes 10.
* During the 2nd iteration, a **revert** occurs due to `require(balanceOf(Bob): 10 < MAX_FIGHTERS_ALLOWED: 10);`.

**As a result, Bob is unable to claim at least one reward from two eligible rewards because the claim rewards transaction reverts, even though he is eligible to claim at least one reward since Bob's balance has not reached the maximum to claim that one.**

```solidity
File: FighterFarm.sol

function _createNewFighter(
        address to, 
        uint256 dna, 
        string memory modelHash,
        string memory modelType, 
        uint8 fighterType,
        uint8 iconsType,
        uint256[2] memory customAttributes
    ) 
        private 
    {  
-->    require(balanceOf(to) < MAX_FIGHTERS_ALLOWED);
        
        // SNIPPED

    }
```
---
### Recommended Mitigation Steps
Implement partially claim functionality within the `MergingPool` contract, or update the [MergingPool::claimRewards()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L139-L167) function to support the mentioned case in this issue.

---

## <a name="m-03">[M-03]</a> Lack of resetting accessible of crucial roles

* **Severity**: Medium
* Source: [Lack of resetting accessible of crucial roles](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/2077)

---
### Description
The [FighterFarm](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol) and [GameItems](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol) contracts do not designate an address holding the staker role `hasStakerRole` and burning role `allowedBurningAddresses`, respectively.

*There is 2 instances of this issue:*

```solidity
File: FighterFarm.sol

    function addStaker(address newStaker) external {
        require(msg.sender == _ownerAddress);
-->     hasStakerRole[newStaker] = true;
    }
```

```solidity
File: GameItems.sol

    function setAllowedBurningAddresses(address newBurningAddress) public {
        require(isAdmin[msg.sender]);
-->     allowedBurningAddresses[newBurningAddress] = true;
    }
```

---
### Recommendation
Update the logic to flexibly set the accessible status as follows:

```diff
File: FighterFarm.sol

-   function addStaker(address newStaker) external {
+   function setStaker(address newStaker, bool status) external {
        require(msg.sender == _ownerAddress);
-       hasStakerRole[newStaker] = true;
+       hasStakerRole[newStaker] = status;
    }
```

```diff
File: GameItems.sol

-    function setAllowedBurningAddresses(address newBurningAddress) public {
+    function setAllowedBurningAddresses(address newBurningAddress, bool status) public {
        require(isAdmin[msg.sender]);
-       allowedBurningAddresses[newBurningAddress] = true;
+       allowedBurningAddresses[newBurningAddress] = status;
    }
```

---

## <a name="m-04">[M-04]</a> Lack of role revocation functionality in `Neuron` Contract

* **Severity**: Medium
* Source: [Lack of role revocation functionality in `Neuron` Contract](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/2075)

---
### Description
The [Neuron](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol) contract does not designate an account holding the[AccessControl::DEFAULT_ADMIN_ROLE](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.0/contracts/access/AccessControl.sol#L57).

Therefore, if the contract owner mistakenly assigns roles such as `MINTER_ROLE`, `STAKER_ROLE`, or `SPENDER_ROLE` to unintended addresses, there is **no mechanism to revoke these roles from those addresses**.


```solidity
--> function revokeRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
        _revokeRole(role, account);
    }
```

*There is one instance of this issue:*

```solidity
File: Neuron.sol

    contract Neuron is ERC20, AccessControl {

        // SNIPPED

    }
```
---
### Recommendation
Assign an account holding the `AccessControl::DEFAULT_ADMIN_ROLE` in the `Neuron` contract to enable role management.

Additionally, implement logic to revoke the `AccessControl::DEFAULT_ADMIN_ROLE` when transferring ownership

```diff
File: Neuron.sol

    constructor(address ownerAddress, address treasuryAddress_, address contributorAddress)
        ERC20("Neuron", "NRN")
    {
        _ownerAddress = ownerAddress;
        treasuryAddress = treasuryAddress_;
        isAdmin[_ownerAddress] = true;
        _mint(treasuryAddress, INITIAL_TREASURY_MINT);
        _mint(contributorAddress, INITIAL_CONTRIBUTOR_MINT);
+       _grantRole(DEFAULT_ADMIN_ROLE, _ownerAddress);
    } 

    //SNIPPED

    function transferOwnership(address newOwnerAddress) external {
        require(msg.sender == _ownerAddress);
+       _revokeRole(DEFAULT_ADMIN_ROLE, _ownerAddress);
        _ownerAddress = newOwnerAddress;
+       _grantRole(DEFAULT_ADMIN_ROLE, newOwnerAddress);
    }
```
---

## <a name="qa">[QA Report]</a> Low and Informational findings
| ID | Description | Severity |
| :-: | - | :-: |
| [L-01](#l-01) | Previous owner retains administrative abilities after ownership transfer | 🫑 |
| [L-02](#l-02) | The `Neuron::burnFrom()` does not support infinite approval | 🫑 |
| [L-03](#l-03) | Lack of setter functions for crucial external Addresses | 🫑 |
| [L-06](#l-04) | Lack of customized replenishment times for game items | 🫑 |
| [N-01](#n-01) | Recommend explicitly setting the `GameItems::itemsRemaining` when the supply is not finite | 🫐 |
| [N-02](#n-02) | Enhance consistency in the `GameItems::remainingSupply()` when querying items of non-finite supply | 🫐 |

\* Source: [AI-Arena: QA Report](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1100)

---

### **[L-01] Previous owner retains administrative abilities after ownership transfer**<a name="l-01"></a>

As the first owner retrieves the admin accessibility at the deployment of the contract, however, the `transferOwnership()` does not reset the admin accessibility of the previous owner when transferring ownership. 

This leads to the previous owner still retaining administrative abilities until the new owner resets their accessibility via `adjustAdminAccess()`.

*There is 5 instances of this issue:*

```solidity
File: GameItems.sol

    function transferOwnership(address newOwnerAddress) external {
        require(msg.sender == _ownerAddress);
        _ownerAddress = newOwnerAddress;
    }
```

```solidity
File: MergingPool.sol

    function transferOwnership(address newOwnerAddress) external {
        require(msg.sender == _ownerAddress);
        _ownerAddress = newOwnerAddress;
    }
```

```solidity
File: Neuron.sol

    function transferOwnership(address newOwnerAddress) external {
        require(msg.sender == _ownerAddress);
        _ownerAddress = newOwnerAddress;
    }
```

```solidity
File: RankedBattle.sol

    function transferOwnership(address newOwnerAddress) external {
        require(msg.sender == _ownerAddress);
        _ownerAddress = newOwnerAddress;
    }
```

```solidity
File: VoltageManager.sol

    function transferOwnership(address newOwnerAddress) external {
        require(msg.sender == _ownerAddress);
        _ownerAddress = newOwnerAddress;
    }
```

#### Recommendation
Update the `transferOwnership()` to reset the admin accessibility of the previous owner by adding a line of code to revoke their admin access.

The functions that should be updated for this issue are:
* [GameItems::transferOwnership()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L108-L111)
* [MergingPool::transferOwnership()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol#L89-L92)
* [Neuron::transferOwnership()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L85-L88)
* [RankedBattle::transferOwnership()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L167-L170)
* [VoltageManager::transferOwnership()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/VoltageManager.sol#L64-L67)

The example recommendation code: 

```diff
File: GameItems.sol

        function transferOwnership(address newOwnerAddress) external {
        require(msg.sender == _ownerAddress);
+       isAdmin[_ownerAddress] = false;
        _ownerAddress = newOwnerAddress;
    }
```

---

### **[L-02] The `Neuron::burnFrom()` does not support infinite approval**<a name="l-02"></a>

The [Neuron::burnFrom()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L196-L204) does not support infinite approval, which is inconsistent with the behavior of [ERC20::transferFrom()](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.0/contracts/token/ERC20/ERC20.sol#L158-L167) when the spender performs on behalf of the token owner.

*There is one instance of this issue:*

```solidity
File: Neuron.sol

    function burnFrom(address account, uint256 amount) public virtual {
        require(
            allowance(account, msg.sender) >= amount, 
            "ERC20: burn amount exceeds allowance"
        );
        uint256 decreasedAllowance = allowance(account, msg.sender) - amount;
        _burn(account, amount);
        _approve(account, msg.sender, decreasedAllowance);
    }
```

#### Recommendation
Implement the functionality to support infinite approval within the `Neuron::burnFrom()` function, and also refactoring the function to consume gas more efficiently.

```diff
File: Neuron.sol

    function burnFrom(address account, uint256 amount) public virtual {
+        uint256 currentAllowance = allowance(account, msg.sender);
+        if (currentAllowance != type(uint256).max) {
            require(
-                allowance(account, msg.sender) >= amount
+                currentAllowance >= amount,
                "ERC20: burn amount exceeds allowance"
            );
+           unchecked {
+               _approve(account, msg.sender, currentAllowance - amount);
+           }
+       }
-       uint256 decreasedAllowance = allowance(account, msg.sender) - amount;
        _burn(account, amount);
-       _approve(account, msg.sender, decreasedAllowance);
    }
```

---

### **[L-03] Lack of setter functions for crucial external Addresses**<a name="l-03"></a>

The `treasuryAddress_` and `delegatedAddress` addresses are external contracts used to retrieve the NRN and act as signers of the claim fighter messages, respectively.

These two states lack setter functions, which means their addresses cannot be updated. Considering these addresses could potentially be compromised, adding the ability to update them would mitigate this risk.

*There is 4 contracts of this issue:*

* The [FighterFarm](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol) contract
* The [GameItems](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol) contract
* The [Neuron](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol) contract
* The [StakeAtRisk](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/StakeAtRisk.sol) contract

#### Recommendation
Implementing setter functions for the `treasuryAddress_` and `delegatedAddress` states to allow for their update is recommended. 

These setter functions should be restricted to high-level roles to ensure proper control over the modification of these critical addresses.

The contracts that should be updated for this issue are:
* The [FighterFarm](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol) contract
* The [GameItems](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol) contract
* The [Neuron](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol) contract
* The [StakeAtRisk](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/StakeAtRisk.sol) contract

---

### **[L-04] Lack of customized replenishment times for game items**<a name="l-06"></a>

Each game item should have its replenishment time specified, taking into account factors such as the item's rarity or special abilities. This ensures that the time required to replenish each item is customized to its unique characteristics within the game.

*There is one instance of this issue:*

```solidity
File: GameItems.sol

function _replenishDailyAllowance(uint256 tokenId) private {
        allowanceRemaining[msg.sender][tokenId] = allGameItemAttributes[tokenId].dailyAllowance;
-->     dailyAllowanceReplenishTime[msg.sender][tokenId] = uint32(block.timestamp + 1 days);
    }
```

#### Recommendation
Update the logic to dynamically set the `dailyAllowanceReplenishTime` for each game items:

```diff
File: GameItems.sol

function _replenishDailyAllowance(uint256 tokenId) private {
        allowanceRemaining[msg.sender][tokenId] = allGameItemAttributes[tokenId].dailyAllowance;
-       dailyAllowanceReplenishTime[msg.sender][tokenId] = uint32(block.timestamp + 1 days);
+       dailyAllowanceReplenishTime[msg.sender][tokenId] = uint32(block.timestamp + allGameItemAttributes[tokenId].dailyAllowanceReplenishPeriod);
    }
```

```diff
File: GameItems.sol

    struct GameItemAttributes {
        string name;
        bool finiteSupply;
        bool transferable;
        uint256 itemsRemaining;
        uint256 itemPrice;
        uint256 dailyAllowance;
+       uint256 dailyAllowanceReplenishPeriod;
    }  
```

```diff
File: GameItems.sol

    function createGameItem(
        string memory name_,
        string memory tokenURI,
        bool finiteSupply,
        bool transferable,
        uint256 itemsRemaining,
        uint256 itemPrice,
        uint16 dailyAllowance,
+       uint256 dailyAllowanceReplenishPeriod,
    ) 
        public 
    {
        require(isAdmin[msg.sender]);
        allGameItemAttributes.push(
            GameItemAttributes(
                name_,
                finiteSupply,
                transferable,
                itemsRemaining,
                itemPrice,
                dailyAllowance,
+               dailyAllowanceReplenishPeriod
            )
        );
        if (!transferable) {
          emit Locked(_itemCount);
        }
        setTokenURI(_itemCount, tokenURI);
        _itemCount += 1;
    }
```

---

### **[N-01] Recommend explicitly setting the `GameItems::itemsRemaining` when the supply is not finite** <a name="n-01"></a>

The [GameItems::createGameItem()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L208-L235) could enhancing the explicit setup of  `itemsRemaining` when creating non-finite supply items.

*There is one instance of this issue:*

```solidity
File: GameItems.sol

    function createGameItem(
        string memory name_,
        string memory tokenURI,
        bool finiteSupply,
        bool transferable,
        uint256 itemsRemaining,
        uint256 itemPrice,
        uint16 dailyAllowance
    ) 
        public 
    {
        require(isAdmin[msg.sender]);
        allGameItemAttributes.push(
            GameItemAttributes(
                name_,
                finiteSupply,
                transferable,
                itemsRemaining,
                itemPrice,
                dailyAllowance
            )
        );
        if (!transferable) {
          emit Locked(_itemCount);
        }
        setTokenURI(_itemCount, tokenURI);
        _itemCount += 1;
    }
```

#### Recommendation
Explicitly set the `GameItems::itemsRemaining` when the supply is not finite to infinite `itemsRemaining` supply.

```diff
File: GameItems.sol

    function createGameItem(
        string memory name_,
        string memory tokenURI,
        bool finiteSupply,
        bool transferable,
        uint256 itemsRemaining,
        uint256 itemPrice,
        uint16 dailyAllowance
    ) 
        public 
    {
        require(isAdmin[msg.sender]);
        allGameItemAttributes.push(
            GameItemAttributes(
                name_,
                finiteSupply,
                transferable,
+               finiteSupply ? itemsRemaining: type(uint256).max,
                itemPrice,
                dailyAllowance
            )
        );
        if (!transferable) {
          emit Locked(_itemCount);
        }
        setTokenURI(_itemCount, tokenURI);
        _itemCount += 1;
    }
```

---

### **[N-02] Enhance consistency in the `GameItems::remainingSupply()` when querying items of non-finite supply**<a name="n-02"></a>

The [GameItems::remainingSupply()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol#L279-L281) could be improved to provide clearer support for querying the supply of each item by returning the `finiteSupply` status along with the supply value.

This can explicitly assist in off-chain retrieval of `remainingSupply` of each item and facilitate better decision-making regarding the management of items as a non-finite supply items.

*There is one instance of this issue:*

```solidity
File: GameItems.sol

    function remainingSupply(uint256 tokenId) public view returns (uint256) {
        return allGameItemAttributes[tokenId].itemsRemaining;
    }
```

#### Recommendation
Update the setting the `GameItems::remainingSupply()` to return `itemsRemaining` amount along with the item's `finiteSupply` status.

```diff
File: GameItems.sol

    function remainingSupply(uint256 tokenId) public view returns (uint256, bool) {
+       return (allGameItemAttributes[tokenId].itemsRemaining, allGameItemAttributes[tokenId].finiteSupply);
    }
```

---

## <a name="gas">[Gas Report]</a> Gas Optimizations
| ID | Description |
| :-: | - |
| [G-01](#g-01) | Recommend removing the redundant logic |
| [G-02](#g-02) | Recommend refactoring to execute point addition logic for nonzero values |
| [G-03](#g-03) | Recommend reusing the functionality of `AccessControl` for role management |
| [G-04](#g-04) | Recommend enhancing `GameItems` contract with `burnBatch` functionality |
| [G-05](#g-05) | Recommend implementing separate reward claiming functionality within the `MergingPool` contract |
| [G-06](#g-06) | Recommend refactoring FighterFarm::_createNewFighter() to implement Inline Emission |

\* Source: [AI-Arena: Gas Report](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1098)

---

### **[G-01] Recommend removing the redundant logic**<a name="g-01"></a>

The [AiArenaHelper::constructor()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/AiArenaHelper.sol#L41-L52) contains redundant logic, specifically at line [AiArenaHelper::L49](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/AiArenaHelper.sol#L49), where attribute probabilities are set. 

```solidity
File: AiArenaHelper.sol

49: attributeProbabilities[0][attributes[i]] = probabilities[i];
```

This logic redundant the functionality found in the [AiArenaHelper::addAttributeProbabilities()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/AiArenaHelper.sol#L131-L139).

```solidity
File: AiArenaHelper.sol

function addAttributeProbabilities(uint256 generation, uint8[][] memory probabilities) public {
        require(msg.sender == _ownerAddress);
        require(probabilities.length == 6, "Invalid number of attribute arrays");

        uint256 attributesLength = attributes.length;
        for (uint8 i = 0; i < attributesLength; i++) {
-->         attributeProbabilities[generation][attributes[i]] = probabilities[i];
        }
    }
```

*There is one instance of this issue:*

```solidity
File: AiArenaHelper.sol

    constructor(uint8[][] memory probabilities) {
        _ownerAddress = msg.sender;

        // Initialize the probabilities for each attribute
        addAttributeProbabilities(0, probabilities);

        uint256 attributesLength = attributes.length;
        for (uint8 i = 0; i < attributesLength; i++) {
            attributeProbabilities[0][attributes[i]] = probabilities[i];
            attributeToDnaDivisor[attributes[i]] = defaultAttributeDivisor[i];
        }
    } 
```

---

### **[G-02] Recommend refactoring to execute point addition logic for nonzero values**<a name="g-02"></a>

The point addition logic at [RankedBattle::L466-468](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L466-L468) and [RankedBattle::L485-487](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/RankedBattle.sol#L485-L487) can be optimized by executing it conditionally, where `points > 0`, to reduce gas consumption from unnecessary addition or subtraction operations when the amount is zero.

*There is two instances of this issue:*

```solidity
File: RankedBattle.sol

    function _addResultPoints(
        uint8 battleResult, 
        uint256 tokenId, 
        uint256 eloFactor, 
        uint256 mergingPortion,
        address fighterOwner
    ) 
        private 
    {
        // SNIPPED

        if (battleResult == 0) {
            /// If the user won the match

            /// If the user has no NRNs at risk, then they can earn points
            if (stakeAtRisk == 0) {
                points = stakingFactor[tokenId] * eloFactor;
            }

            /// Divert a portion of the points to the merging pool
            uint256 mergingPoints = (points * mergingPortion) / 100;
            points -= mergingPoints;
            _mergingPoolInstance.addPoints(tokenId, mergingPoints);

            /// Do not allow users to reclaim more NRNs than they have at risk
            if (curStakeAtRisk > stakeAtRisk) {
                curStakeAtRisk = stakeAtRisk;
            }

            /// If the user has stake-at-risk for their fighter, reclaim a portion
            /// Reclaiming stake-at-risk puts the NRN back into their staking pool
            if (curStakeAtRisk > 0) {
                _stakeAtRiskInstance.reclaimNRN(curStakeAtRisk, tokenId, fighterOwner);
                amountStaked[tokenId] += curStakeAtRisk;
            }

            /// Add points to the fighter for this round
-->         accumulatedPointsPerFighter[tokenId][roundId] += points;
-->         accumulatedPointsPerAddress[fighterOwner][roundId] += points;
-->         totalAccumulatedPoints[roundId] += points;
            if (points > 0) {
                emit PointsChanged(tokenId, points, true);
            }
        } else if (battleResult == 2) {

            // SNIPPED

        }
    } 
```

```solidity
File: RankedBattle.sol

    function _addResultPoints(
        uint8 battleResult, 
        uint256 tokenId, 
        uint256 eloFactor, 
        uint256 mergingPortion,
        address fighterOwner
    ) 
        private 
    {
        // SNIPPED

        if (battleResult == 0) {

            // SNIPPED

        } else if (battleResult == 2) {
            /// If the user lost the match

            /// Do not allow users to lose more NRNs than they have in their staking pool
            if (curStakeAtRisk > amountStaked[tokenId]) {
                curStakeAtRisk = amountStaked[tokenId];
            }
            if (accumulatedPointsPerFighter[tokenId][roundId] > 0) {
                /// If the fighter has a positive point balance for this round, deduct points 
                points = stakingFactor[tokenId] * eloFactor;
                if (points > accumulatedPointsPerFighter[tokenId][roundId]) {
                    points = accumulatedPointsPerFighter[tokenId][roundId];
                }
-->             accumulatedPointsPerFighter[tokenId][roundId] -= points;
-->             accumulatedPointsPerAddress[fighterOwner][roundId] -= points;
-->             totalAccumulatedPoints[roundId] -= points;
                if (points > 0) {
                    emit PointsChanged(tokenId, points, false);
                }
            } else {
                /// If the fighter does not have any points for this round, NRNs become at risk of being lost
                bool success = _neuronInstance.transfer(_stakeAtRiskAddress, curStakeAtRisk);
                if (success) {
                    _stakeAtRiskInstance.updateAtRiskRecords(curStakeAtRisk, tokenId, fighterOwner);
                    amountStaked[tokenId] -= curStakeAtRisk;
                }
            }
        }
    } 
```

---

### **[G-03] Recommend reusing the functionality of `AccessControl` for role management**<a name="g-03"></a>

Since the [Neuron](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol) contract inherits the [AccessControl](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.0/contracts/access/AccessControl.sol), it can reuse the provided [AccessControl::grantRole()](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.0/contracts/access/AccessControl.sol#L144-L146) to assign specific roles to certain addresses instead of implementing a separate role assignment function.

This approach can reduce the contract size and deployment gas costs.

*There is one instance of this issue:*

```solidity
File: Neuron.sol

    contract Neuron is ERC20, AccessControl {

        // SNIPPED

    }
```

---

### **[G-04] Recommend enhancing `GameItems` contract with `burnBatch` functionality**<a name="g-04"></a>

As the [GameItems](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/GameItems.sol) contract introduce the ability to burn items, it can be improved by implementing the external `GameItems::burnBatch()` to support safer gas consumption during batch burning operations for multiple items.

The contract that can be used as a reference:
* [ERC1155Burnable](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.0/contracts/token/ERC1155/extensions/ERC1155Burnable.sol)

*There is one instance of this issue:*

```solidity
File: GameItems.sol

    contract GameItems is ERC1155 {

        // SNIPPED

    }
```

---

### **[G-05] Recommend implementing separate reward claiming functionality within the `MergingPool` contract**<a name="g-05"></a>

As the [MergingPool](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/MergingPool.sol) contract introduces the ability to batch claim rewards for multiple rounds, it can be improved by implementing the external `MergingPool::claimReward()` to support claiming rewards for specific rounds. 

Consider the scenario where a player has never won in many previous rounds. To claim their rewards, they would need to iterate through all rounds and winners arrays, resulting in significant gas consumption.

This improvement would provide safer gas consumption and prevent denial of service scenarios, particularly in cases where a winner has not won in many previous rounds.

*There is one instance of this issue:*

```solidity
File: MergingPool.sol

    function claimRewards(
        string[] calldata modelURIs, 
        string[] calldata modelTypes,
        uint256[2][] calldata customAttributes
    ) 
        external 
    {
        uint256 winnersLength;
        uint32 claimIndex = 0;
        uint32 lowerBound = numRoundsClaimed[msg.sender];
-->     for (uint32 currentRound = lowerBound; currentRound < roundId; currentRound++) {
            numRoundsClaimed[msg.sender] += 1;
            winnersLength = winnerAddresses[currentRound].length;
            for (uint32 j = 0; j < winnersLength; j++) {
                if (msg.sender == winnerAddresses[currentRound][j]) {
                    _fighterFarmInstance.mintFromMergingPool(
                        msg.sender,
                        modelURIs[claimIndex],
                        modelTypes[claimIndex],
                        customAttributes[claimIndex]
                    );
                    claimIndex += 1;
                }
            }
        }
        if (claimIndex > 0) {
            emit Claimed(msg.sender, claimIndex);
        }
    }
```

---

### **[G-06] Recommend refactoring FighterFarm::_createNewFighter() to implement Inline Emission**<a name="g-06"></a>

As the [FighterFarm::fighterCreatedEmitter()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterOps.sol#L53-L62) is solely used to emit an event when a fighter is created within [FighterFarm::_createNewFighter()#L530](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L530), the [FighterFarm::_createNewFighter()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/FighterFarm.sol#L484-L531) function can be optimized by refactoring to inline the emission.

*There is one instance of this issue:*

```solidity
File: FighterFarm.sol

    function _createNewFighter(
        address to, 
        uint256 dna, 
        string memory modelHash,
        string memory modelType, 
        uint8 fighterType,
        uint8 iconsType,
        uint256[2] memory customAttributes
    ) 
        private 
    {  
        require(balanceOf(to) < MAX_FIGHTERS_ALLOWED);
        uint256 element; 
        uint256 weight;
        uint256 newDna;
        if (customAttributes[0] == 100) {
            (element, weight, newDna) = _createFighterBase(dna, fighterType);
        }
        else {
            element = customAttributes[0];
            weight = customAttributes[1];
            newDna = dna;
        }
        uint256 newId = fighters.length;

        bool dendroidBool = fighterType == 1;
        FighterOps.FighterPhysicalAttributes memory attrs = _aiArenaHelperInstance.createPhysicalAttributes(
            newDna,
            generation[fighterType],
            iconsType,
            dendroidBool
        );
        fighters.push(
            FighterOps.Fighter(
                weight,
                element,
                attrs,
                newId,
                modelHash,
                modelType,
                generation[fighterType],
                iconsType,
                dendroidBool
            )
        );
        _safeMint(to, newId);
-->     FighterOps.fighterCreatedEmitter(newId, weight, element, generation[fighterType]);
    }
```

---