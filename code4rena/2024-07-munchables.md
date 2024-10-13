# Munchables - LandManager Report
Munchables is a GameFi project with a twist.

The objective of the game is to earn as many Munch Points as possible. In crypto terms, you could call this "point farming".

Built on top of Blast, Munchables leverages the unique on-chain primitives to create a reward-filled journey. Players collect Munchables and keep them safe, fed and comfortable in their snuggery.
Once in a snuggery, a Munchable can start earning rewards for that player. A variety of factors influence the rewards earned, so players will have to be smart when choosing which Munchables to put in their snuggery and the strategies they use to play the game.

* **nSLOC**: 277 
* **Contest details**: https://code4rena.com/audits/2024-07-munchables

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: [Severity Categorization | Code4rena](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[H-01](#h-01)| Incomplete Update of `ToilerState` in `LandManager::transferToUnoccupiedPlot()`|🍅|
|[H-02](#h-02)| Discrepancy in plot ID validation allows invalid plots to farm schnibbles |🍅|
|[H-03](#h-03)| `schnibblesTotal` calculation with the `finalBonus` can result in underflow and incorrect value |🍅|
|[H-04](#h-04)| Incorrect timestamp updating for invalid plots due to USD price fluctuation |🍅|
|[H-05](#h-05)| Incomplete Update of `ToilerState` in `LandManager::transferToUnoccupiedPlot()`|🍅|
|[M-01](#m-01)| Users can farm on zero-tax land if the landlord locked tokens before the LandManager deployment |🍋|

\* **H-05** is similar to **H-01** but judged as a separate issue due to its distinct root cause. It was rewarded separately, covering all related roots.

---

## <a name="h-01">[H-01]</a> Incomplete Update of `ToilerState` in `LandManager::transferToUnoccupiedPlot()`

* **Severity**: High
* Source: [#317 - Incomplete Update of ToilerState in LandManager::transferToUnoccupiedPlot()](https://github.com/code-423n4/2024-07-munchables-findings/issues/317)
---

### Links to affected code
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L199-L226
---

### Description
The [`LandManager::transferToUnoccupiedPlot()`](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L199-L226) function does not properly update the `ToilerState` for the given `tokenId`.

```solidity 
File: src/managers/LandManager.sol

199:     function transferToUnoccupiedPlot(
200:         uint256 tokenId,
201:         uint256 plotId
202:     ) external override forceFarmPlots(msg.sender) notPaused {
203:         (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
204:         ToilerState memory _toiler = toilerState[tokenId];
205:         uint256 oldPlotId = _toiler.plotId;
206:         uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
207:         if (_toiler.landlord == address(0)) revert NotStakedError();
208:         if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
209:         if (plotOccupied[_toiler.landlord][plotId].occupied)
210:             revert OccupiedPlotError(_toiler.landlord, plotId);
211:         if (plotId >= totalPlotsAvail) revert PlotTooHighError();
212: 
213:@>       toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]    //@audit only update the latestTaxRate state
214:             .currentTaxRate;
215:         plotOccupied[_toiler.landlord][oldPlotId] = Plot({
216:             occupied: false,
217:             tokenId: 0
218:         });
219:         plotOccupied[_toiler.landlord][plotId] = Plot({
220:             occupied: true,
221:             tokenId: tokenId
222:         });
223: 
224:         emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
225:         emit FarmPlotTaken(toilerState[tokenId], tokenId);
226:     }
```

The `ToilerState` struct:
``` soldiity
    struct ToilerState {    
        uint256 lastToilDate;   
        uint256 plotId;
        address landlord;    
        uint256 latestTaxRate;
        bool dirty;
    }
```

The [LandManager::transferToUnoccupiedPlot()](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L199-L226) is intended to allow renters to change their plot to an unoccupied plot for various reasons, such as:

1. Changing to a lesser `plotId` to avoid invalid plotId values when the landlord updates their plots.
2. Moving to a valid plot when the current `ToilerState` is dirty, preventing farming.

However, the incomplete update of the `ToilerState` when transferring to an unoccupied plot results in incorrect behavior of the `ToilerState` for the `tokenId`.

---
### Impact
The impact of this issue can manifest in multiple ways due to the incorrect update of the `ToilerState`. 

The core functions that are affected by this issue are:
#### 1. **[LandManager::unstakeMunchable()](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L181-L184)**
- **Impact**:
  - The plot that should be marked as unoccupied remains in the occupied status, preventing other users from farming at that plot.
  - The landlord loses potential benefits as the plot remains locked in an occupied state permanently.

#### 2. **[LandManager::_farmPlots()](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L251-L261)**

- **Case 1**: If the previous `plotId` before transferring is dirty, the dirty state is not updated during the transfer. This causes the token to be unable to farm, even if it is on a valid plot.
    - **Impact**: The token will be unable to farm properly, even if it is on a valid plot, due to outdated dirty state information.

- **Case 2**: When transferring from one valid `plotId` to another valid `plotId`, the function may incorrectly check the validity of the new plot based on the previous plot's state.
    - **Impact**: The farming operation may incorrectly validate the plot, leading to issues in the farming mechanism and potentially affecting rewards and operations related to the plot.


---
**Severity Clarification Aspects**:
* **Likelihood**: Can be considered **`Medium`** as the issue occurs every time a renter transfers to an unoccupied plot, making it a consistent problem in the relevant function. The frequency varies with the number of renters and their actions but is likely a common operation. 

* **Impact**: Can be considered **`Medium` or `High`** as outlined above, the issue affects key functionalities and can disrupt plot management and farming operations.

---
### Tools Used
* Manual Review

---

### Recommended Mitigation Steps
Update the `LandManager::transferToUnoccupiedPlot()` function to update all relevant fields in the `ToilerState`.

```diff
File: src/managers/LandManager.sol

199:     function transferToUnoccupiedPlot(
200:         uint256 tokenId,
201:         uint256 plotId
202:     ) external override forceFarmPlots(msg.sender) notPaused {
203:         (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
204:         ToilerState memory _toiler = toilerState[tokenId];
205:         uint256 oldPlotId = _toiler.plotId;
206:         uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
207:         if (_toiler.landlord == address(0)) revert NotStakedError();
208:         if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
209:         if (plotOccupied[_toiler.landlord][plotId].occupied)
210:             revert OccupiedPlotError(_toiler.landlord, plotId);
211:         if (plotId >= totalPlotsAvail) revert PlotTooHighError();
212: 
-213:        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
+213:         toilerState[tokenId] = ToilerState({
+214:             lastToilDate: block.timestamp,
+215:             plotId: plotId,
+216:             landlord: _toiler.landlord,
+217:             latestTaxRate: plotMetadata[landlord].currentTaxRate,
+218:             dirty: false
+219:         });
            --- SNIPPED ---
         }
```

---

## <a name="h-02">[H-02]</a> Discrepancy in plot ID validation allows invalid plots to farm schnibbles

* **Severity**: High
* Source: [Discrepancy in plot ID validation allows invalid plots to farm schnibbles](https://github.com/code-423n4/2024-07-munchables-findings/issues/31)

---
### Links to affected code
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L258

---
### Description
The validation of plot IDs in [`LandManager::_farmPlots()`](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L232-L310) is inconsistent with other functions like `stakeMunchable()` and `transferToUnoccupiedPlot()`. This discrepancy allows invalid plot IDs to bypass checks and receive full schnibbles farming, which should not be possible.

```solidity 
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---

247:         for (uint8 i = 0; i < staked.length; i++) {
                --- SNIPPED ---

253:             // use last updated plot metadata time if the plot id doesn't fit
254:             // track a dirty bool to signify this was done once
255:             // the edge case where this doesnt work is if the user hasnt farmed in a while and the landlord
256:             // updates their plots multiple times. then the last updated time will be the last time they updated their plot details
257:             // instead of the first
258:@>           if (_getNumPlots(landlord) < _toiler.plotId) {
259:                 timestamp = plotMetadata[landlord].lastUpdated;
260:                 toilerState[tokenId].dirty = true;
261:             }
                --- SNIPPED ---
310:     }
```
**Explanation:**

Using [`LandManager::stakeMunchable()`](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L131-L171) as an example, there is a check to ensure the `plotId` is **less than** `totalPlotsAvail` (total plots available), which correctly prevents staking in non-existent plots:

```solidity
File: src/managers/LandManager.sol

131:     function stakeMunchable(
132:         address landlord,
133:         uint256 tokenId,
134:         uint256 plotId
135:     ) external override forceFarmPlots(msg.sender) notPaused {
             --- SNIPPED ---
145:@>       uint256 totalPlotsAvail = _getNumPlots(landlord);
146:@>       if (plotId >= totalPlotsAvail) revert PlotTooHighError();
147: 
            --- SNIPPED ---
171:     }
```

The [L146](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L146) means that the valid range for `plotId` based on this condition is from [0:`totalPlotsAvail - 1`]. This means if `totalPlotsAvail` is, for example, `10`, the valid `plotId` would be `[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]`.

If the landlord updates plots and `totalPlotsAvail` decreases to `5`, valid `plotId` range should become `[0, 1, 2, 3, 4]`.

However, the `plotId: 5` can still act as a valid plot and farm schnibbles due to the discrepancy in the invalid plot check in [`LandManager::_farmPlots()#L258`](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L258):

```solidity 
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---

247:         for (uint8 i = 0; i < staked.length; i++) {
                --- SNIPPED ---
                
258:@>           if (_getNumPlots(landlord) /** 5 */ < _toiler.plotId /** 5 */) { //@audit FALSE => skip the checks
259:                 timestamp = plotMetadata[landlord].lastUpdated;
260:                 toilerState[tokenId].dirty = true;
261:             }
                --- SNIPPED ---
310:     }
```
**Result:** An invalid `plotId` that should be dirty and unable to gain schnibbles can pass the check and receive full schnibbles farming.

---
### Impact
An invalid `plotId` that should be marked as dirty and unable to gain schnibbles can pass the check in `LandManager::_farmPlots()` and receive full schnibbles farming, allowing users to remain on the invalid plot and gain undue benefits.

---
**Severity Clarification Aspects**:
* **Likelihood**: Can be considered **`Medium`** as the discrepancy in the validation logic always allows users to encounter this issue, but only invalid tokens at each edge of the valid `plotId` range will gain the benefit when the landlord updates their plots.

* **Impact**: Can be considered **`High`** as it directly breaks the mechanism's intention that invalid plots should be dirty and unable to farm schnibbles.

---
### Tools Used
* Manual Review

---

### Recommended Mitigation Steps
Update the invalid plot validation in the `LandManager::_farmPlots()` to be consistent with other `plotId` validation logic.

```diff
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---

247:         for (uint8 i = 0; i < staked.length; i++) {
                --- SNIPPED ---
                
-258:           if (_getNumPlots(landlord) < _toiler.plotId) {
+258:           if (_getNumPlots(landlord) <= _toiler.plotId) {
259:                 timestamp = plotMetadata[landlord].lastUpdated;
260:                 toilerState[tokenId].dirty = true;
261:             }
                --- SNIPPED ---
310:     }
```
---

## <a name="h-03">[H-03]</a> `schnibblesTotal` calculation with the `finalBonus` can result in underflow and incorrect value

* **Severity**: High
* Source: [`schnibblesTotal` calculation with the `finalBonus` can result in underflow and incorrect value](https://github.com/code-423n4/2024-07-munchables-findings/issues/32)

---
### Links to affected code
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L283-L286

---
### Description
`lastPetMunchable` probably should be removed but it doesnt change any logic really since if the munchable is in the land manager, it shouldnt be able to be pet anyway _> only effect is extend the time for pet but actually when the 

```solidity 
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---
244:@>       int256 finalBonus;
                --- SNIPPED ---
247:         for (uint8 i = 0; i < staked.length; i++) {
                --- SNIPPED ---
270:@>           finalBonus =
271:                 int16(
272:                     REALM_BONUSES[
273:                         (uint256(immutableAttributes.realm) * 5) +
274:                             uint256(landlordMetadata.snuggeryRealm)
275:                     ]
276:                 ) +
277:                 int16(
278:                     int8(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
279:                 );
280:             schnibblesTotal =
281:                 (timestamp - _toiler.lastToilDate) *
282:                 BASE_SCHNIBBLE_RATE;
283:@>           schnibblesTotal = uint256(
284:                 (int256(schnibblesTotal) +
285:                     (int256(schnibblesTotal) * finalBonus)) / 100
286:             );

                --- SNIPPED ---
310:     }
```

---
### Impact
The incorrect calculation of `schnibblesTotal` due to the flawed formula can result in:
- Users receive incorrect amounts of schnibbles, either more or less than intended, which disrupts the intended reward system.
- The formula's ****underflow** could lead to unfair distribution of rewards, affecting the overall balance and fairness of the farming mechanism.
---
**Severity Clarification Aspects**:
* **Likelihood**: Can be considered **`High`** as the incorrect calculation formula is used in every schnibbles calculation involving `finalBonus`. This means the issue will frequently occur whenever `finalBonus` is applied, impacting numerous transactions.

* **Impact**: Can be considered **`High`** as it affects the fundamental reward distribution mechanism. Incorrect schnibbles values can cause significant discrepancies in the rewards, affecting the economic balance and fairness for all users.

---
### Proof of Concept
**Prerequisites**
- The `finalBonus` is stored in `int256` and can be both **positive** and **negative**. It is calculated from the values in `REALM_BONUSES[x] (min: -10, max: 10)` and `RARITY_BONUSES[y] (min: 0, max: 50)`. This can be verified by the values configured on-chain in the live [ConfigStorage](https://blastscan.io/address/0xEf173BB4b36525974BF6711357D2c6C12b8001eC#code) contract (`StorageKey.RealmBonuses: 34`, `StorageKey.RarityBonuses: 35`). Thus, `finalBonus` can in the range from min: `-10` to max: `60`.

- The issue lies in the calculation formula. The following examples illustrate the incorrect results and can can be simply unit tested in an environment like Remix IDE.

---

**For a POSITIVE `finalBonus` value:**

1. Given `schnibblesTotal` is `100e18` and `finalBonus` is `10`:
    - The expected result is `100e18` + `10%` = `110e18`.

2. Following the formula execution order, the result would be `11e18`:
    ```solidity
    schnibblesTotal = uint256(
                    (int256(schnibblesTotal) +
                        (int256(schnibblesTotal) * finalBonus)) / 100
                )
    ```

    **Execution steps**:
    1. `(int256(schnibblesTotal) * finalBonus)`   ___ (1)
    2. `int256(schnibblesTotal) + (1)`            ___ (2)
    3. `(2) / 100`                                ___ (3)
    4. `uint256((3))`                             ___ (4)

    **Detailed calculation**:
    1. `(int256(100e18) * 10) = 1000e18`          ___ (1)
    2. `int256(100e18) + (1000e18) = 1100e18`    ___ (2)
    3. `(1100e18) / 100 = 11e18`                 ___ (3)
    4. **`uint256(11e18) = 11e18`                    ___ (4)**

    **Result:** Incorrect result as **`expected (110e18) != actual (11e18)`**

---

**For a NEGATIVE `finalBonus` value:**

1. Given `schnibblesTotal` is `100e18` and `finalBonus` is `-10`:
    - The expected result is `100e18` - `10%` = `90e18`.

2. Following the formula execution order, the result would be **UNDERFLOW**:
    ```solidity
    schnibblesTotal = uint256(
                    (int256(schnibblesTotal) +
                        (int256(schnibblesTotal) * finalBonus)) / 100
                )
    ```

    **Execution steps**:
    1. `(int256(schnibblesTotal) * finalBonus)`   ___ (1)
    2. `int256(schnibblesTotal) + (1)`            ___ (2)
    3. `(2) / 100`                                ___ (3)
    4. `uint256((3))`                             ___ (4)

    **Detailed calculation**:
    1. `(int256(100e18) * (-10)) = -1000e18`       ___ (1)
    2. `int256(100e18) + (-1000e18) = -900e18`     ___ (2)
    3. `(-900e18) / 100 = -9e18`                  ___ (3)
    4. **`uint256(-9e18) = UNDERFLOW`                ___ (4)**

    **Result:** Incorrect result as **`expected (90e18) != actual (UNDERFLOW)`**

---

### Tools Used
* Manual Review

---
### Recommended Mitigation Steps
Update the formula to correctly apply the `finalBonus`:

```diff 
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---
-283:           schnibblesTotal = uint256(
-284:                 (int256(schnibblesTotal) +
-285:                     (int256(schnibblesTotal) * finalBonus)) / 100
-286:             );
+283:           schnibblesTotal = uint256(
+284:                 int256(schnibblesTotal) * int256(100 + finalBonus) / 100
+285:             );

                --- SNIPPED ---
310:     }
```
---

## <a name="h-04">[H-04]</a> Incorrect timestamp updating for invalid plots due to USD price fluctuation

* **Severity**: High
* Source: [Incorrect timestamp updating for invalid plots due to USD price fluctuation](https://github.com/code-423n4/2024-07-munchables-findings/issues/37)

---
### Links to affected code
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L258-L282
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LockManager.sol#L484-L492
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LockManager.sol#L523-L524

---
### Description
Varying by USD price of the locked tokens proposed by oracle can make the Plot smaller and dirty, but the landlord's `plotMetadata.timestamp` does not update when the price changes.

```solidity 
File: src/managers/LandManager.sol

344:     function _getNumPlots(address _account) internal view returns (uint256) {
345:@>       return lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT;
346:     }
```

As the `_getNumPlots()` can vary due to two factors: `lock size` (landlord) and `USD price` of the locked token (proposed by a trusted oracle), the USD price could lower the return of `lockManager::getLockedWeightedValue()` for the same lock amount.

```solidity 
File: src/managers/LockManager.sol
On-chain deployed: 0xEA091311Fc07139d753A6BBfcA27aB0224854Bae (Blast)

468:     /// @inheritdoc ILockManager
469:     function getLockedWeightedValue(
470:         address _player
471:     ) external view returns (uint256 _lockedWeightedValue) {
                --- SNIPPED ---
481:                 uint256 deltaDecimal = 10 **
482:                     (18 -
483:                         configuredTokens[configuredTokenContracts[i]].decimals);
484:                 lockedWeighted +=
485:                     (deltaDecimal *
486:                         lockedTokens[_player][configuredTokenContracts[i]]
487:                             .quantity *
488:@>                       configuredTokens[configuredTokenContracts[i]]
489:                             .usdPrice) /
490:                     1e18;
491:             }
492:         }
493: 
494:         _lockedWeightedValue = lockedWeighted;
495:     }

        --- SNIPPED ---

510:     /*******************************************************
511:      ** INTERNAL FUNCTIONS
512:      ********************************************************/
513: 
514:     function _execUSDPriceUpdate() internal {
                --- SNIPPED ---
520:             for (uint256 i; i < updateTokensLength; i++) {
521:                 address tokenContract = usdUpdateProposal.contracts[i];
522:                 if (configuredTokens[tokenContract].nftCost != 0) {
523:@>                   configuredTokens[tokenContract].usdPrice = usdUpdateProposal
524:                         .proposedPrice;
525: 
                --- SNIPPED ---
534:         }
535:     }

```

As a result of not updating the landlord's `plotMetadata.timestamp` when the USD price changes, the `plotMetadata.timestamp` that reflects the timestamp for when the plots become dirty is incorrect. This causes the user to unfairly gain schnibbles form an incorrect time tracking.

```solidity 
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---

247:         for (uint8 i = 0; i < staked.length; i++) {
                --- SNIPPED ---
                
258:             if (_getNumPlots(landlord) < _toiler.plotId) {
259:@>               timestamp = plotMetadata[landlord].lastUpdated;
260:                 toilerState[tokenId].dirty = true;
261:             }
                --- SNIPPED ---
280:@>           schnibblesTotal =
281:@>               (timestamp - _toiler.lastToilDate) *
282:@>               BASE_SCHNIBBLE_RATE;
                --- SNIPPED ---
310:     }
```

### Impact
Users gain schnibbles based on an incorrect time period due to outdated timestamps, leading to potential unfair.

---
**Severity Clarification Aspects**:
* **Likelihood:** Can be considered **`Medium` or `High`** since the tokens that can be locked may have fluctuations in USD price. Although the locked assets listed on-chain include `USDB` (a stablecoin), they also allow fluctuating assets like `WETH`, `BLAST`, so price fluctuation is a concern.

* **Impact:** Can be considered **`High`**. The issue significantly impacts users by leading to incorrect schnibbles calculations and unfair rewards.

---
### Proof of Concept
For the `tokenId` that stakes on the last available plotId of the landlord, this issue could potentially occur.

**Revert on Farming due to Underflow in Use Lesser Timestamp**
1. Landlord has 10 plots available. => `plotMetadata.lastUpdated: block 9`.

2. User stakes at `plot 9` (the maximum available plotId).

3. User farms to gain schnibbles and pays tax to the landlord as expected. => `lastToilDate: block 19`.

4. Assuming the proposed USD price change occurs at `block 30`, and the **USD price in the `LockManager` has decreased**. The landlord still locks 10 plots and does not perform any activity to update their `plotMetadata`. 

5. User attempts to farm again, but `_getNumPlots()` returns a smaller value, causing that `tokenId` to become dirty.
    - `timestamp: plotMetadata.lastUpdated = block 9`.
    - Reverts as `schnibblesTotal = (timestamp(block 9) - lastToilDate (block 19)) * BASE_SCHNIBBLE_RATE`.

**Not Revert but Only Use Lesser Timestamp for Calculation**
1. Landlord has 10 plots available. => `plotMetadata.lastUpdated: block 9`.

2. User stakes at plot 9 (the maximum available plotId).

3. User farms to gain schnibbles and pays tax to the landlord as expected. => `lastToilDate: block 19`.

4. Landlord calls `LockManager::setLockDuration` at `block 20` (without changing any lock amount). => `plotMetadata.lastUpdated: block 20`.

5. Assuming the proposed USD price change occurs at `block 30`, and **the USD price in the `LockManager` has decreased**. The landlord still locks 10 plots and does not perform any activity to update their `plotMetadata`. 

6. User attempts to farm again, but `_getNumPlots()` returns a smaller value, causing that `tokenId` to become dirty.
    - `timestamp: plotMetadata.lastUpdated = block 20`.
    - `schnibblesTotal = (timestamp(block 20) - lastToilDate (block 19)) * BASE_SCHNIBBLE_RATE`.

In both cases, the invalid plot seems to use the wrong `timestamp` retrieved from `plotMetadata.lastUpdated`, which should actually reflect the action that made them invalid.

---
### Tools Used
* Manual Review

---
### Recommended Mitigation Steps
Update `plotMetadata.lastUpdated` whenever there is a change in the USD price that impacts the number of plots (`numPlots`). This ensures that the timestamp reflects the correct state of the plots.

However, since the `LockManager` is already deployed, handling the configuration off-chain might be required. To propose the price, it requires an actor that holds the `PriceFeed` Role. When the new price is applied, they should update the related landlord.

Currently, there are three tokens available for locking, and the number of plots is retrieved from three different quantities of locked tokens, depending on each landlord. **When updating the landlord's `plotMetadata`, ensure that the token price updates are related to the landlord's locked tokens that is used in calculated the number of plots**.

---

## <a name="h-05">[H-05]</a> Incomplete Update of `ToilerState` in `LandManager::transferToUnoccupiedPlot()`

* **Severity**: High
* Source: [#34 - Incomplete Update of `ToilerState` in `LandManager::transferToUnoccupiedPlot()`](https://github.com/code-423n4/2024-07-munchables-findings/issues/34)
---

### Links to affected code
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L199-L226

---
### Description
The [`LandManager::transferToUnoccupiedPlot()`](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L199-L226) function does not properly update the `ToilerState` for the given `tokenId`.

```solidity 
File: src/managers/LandManager.sol

199:     function transferToUnoccupiedPlot(
200:         uint256 tokenId,
201:         uint256 plotId
202:     ) external override forceFarmPlots(msg.sender) notPaused {
203:         (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
204:         ToilerState memory _toiler = toilerState[tokenId];
205:         uint256 oldPlotId = _toiler.plotId;
206:         uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
207:         if (_toiler.landlord == address(0)) revert NotStakedError();
208:         if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
209:         if (plotOccupied[_toiler.landlord][plotId].occupied)
210:             revert OccupiedPlotError(_toiler.landlord, plotId);
211:         if (plotId >= totalPlotsAvail) revert PlotTooHighError();
212: 
213:@>       toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]    //@audit only update the latestTaxRate state
214:             .currentTaxRate;
215:         plotOccupied[_toiler.landlord][oldPlotId] = Plot({
216:             occupied: false,
217:             tokenId: 0
218:         });
219:         plotOccupied[_toiler.landlord][plotId] = Plot({
220:             occupied: true,
221:             tokenId: tokenId
222:         });
223: 
224:         emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
225:         emit FarmPlotTaken(toilerState[tokenId], tokenId);
226:     }
```

The `ToilerState` struct:
``` soldiity
    struct ToilerState {    
        uint256 lastToilDate;   
        uint256 plotId;
        address landlord;    
        uint256 latestTaxRate;
        bool dirty;
    }
```

The [LandManager::transferToUnoccupiedPlot()](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L199-L226) is intended to allow renters to change their plot to an unoccupied plot for various reasons, such as:

1. Changing to a lesser `plotId` to avoid invalid plotId values when the landlord updates their plots.
2. Moving to a valid plot when the current `ToilerState` is dirty, preventing farming.

However, the incomplete update of the `ToilerState` when transferring to an unoccupied plot results in incorrect behavior of the `ToilerState` for the `tokenId`.

---
### Impact
The impact of this issue can manifest in multiple ways due to the incorrect update of the `ToilerState`. 

The core functions that are affected by this issue are:
#### 1. **[LandManager::unstakeMunchable()](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L181-L184)**
- **Impact**:
  - The plot that should be marked as unoccupied remains in the occupied status, preventing other users from farming at that plot.
  - The landlord loses potential benefits as the plot remains locked in an occupied state permanently.

#### 2. **[LandManager::_farmPlots()](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L251-L261)**

- **Case 1**: If the previous `plotId` before transferring is dirty, the dirty state is not updated during the transfer. This causes the token to be unable to farm, even if it is on a valid plot.
    - **Impact**: The token will be unable to farm properly, even if it is on a valid plot, due to outdated dirty state information.

- **Case 2**: When transferring from one valid `plotId` to another valid `plotId`, the function may incorrectly check the validity of the new plot based on the previous plot's state.
    - **Impact**: The farming operation may incorrectly validate the plot, leading to issues in the farming mechanism and potentially affecting rewards and operations related to the plot.


---
**Severity Clarification Aspects**:
* **Likelihood**: Can be considered **`Medium`** as the issue occurs every time a renter transfers to an unoccupied plot, making it a consistent problem in the relevant function. The frequency varies with the number of renters and their actions but is likely a common operation. 

* **Impact**: Can be considered **`Medium` or `High`** as outlined above, the issue affects key functionalities and can disrupt plot management and farming operations.

---
### Tools Used
* Manual Review

---

### Recommended Mitigation Steps
Update the `LandManager::transferToUnoccupiedPlot()` function to update all relevant fields in the `ToilerState`.

```diff
File: src/managers/LandManager.sol

199:     function transferToUnoccupiedPlot(
200:         uint256 tokenId,
201:         uint256 plotId
202:     ) external override forceFarmPlots(msg.sender) notPaused {
203:         (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
204:         ToilerState memory _toiler = toilerState[tokenId];
205:         uint256 oldPlotId = _toiler.plotId;
206:         uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
207:         if (_toiler.landlord == address(0)) revert NotStakedError();
208:         if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
209:         if (plotOccupied[_toiler.landlord][plotId].occupied)
210:             revert OccupiedPlotError(_toiler.landlord, plotId);
211:         if (plotId >= totalPlotsAvail) revert PlotTooHighError();
212: 
-213:        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
+213:         toilerState[tokenId] = ToilerState({
+214:             lastToilDate: block.timestamp,
+215:             plotId: plotId,
+216:             landlord: _toiler.landlord,
+217:             latestTaxRate: plotMetadata[landlord].currentTaxRate,
+218:             dirty: false
+219:         });
            --- SNIPPED ---
         }
```
---

## <a name="m-01">[M-01]</a> Users can farm on zero-tax land if the landlord locked tokens before the LandManager deployment

* **Severity**: Medium
* Source: [Users can farm on zero-tax land if the landlord locked tokens before the LandManager deployment](https://github.com/code-423n4/2024-07-munchables-findings/issues/30)
---
### Links to affected code
* https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L131-L171

---
### Description
The [`LandManager::stakeMunchable()`](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L131-L171) **does not verify whether the `plotMetadata[landlord]` is set or not**.

Since the `LockManager` contract is already deployed on-chain ([0xEA091311Fc07139d753A6BBfcA27aB0224854Bae (Blast)](https://blastscan.io/address/0xEA091311Fc07139d753A6BBfcA27aB0224854Bae)), accounts that have locked their tokens before the `LandManager` deployment will need to trigger `LandManager::triggerPlotMetadata()` to update their `LandManager.plotMetadata` after the `LandManager` is deployed.

* The current `AccountManager` interacts with the `LockManager` on-chain but does not introduce interaction with the `LandManager` during `AccountManager::forceHarvest()`.

    * On-chain: [`AccountManager::forceHarvest()`](https://blastscan.io/address/0xcee5fa48beb21dc9b3abefe414f5cb419bef940e#code#F1#L126)
    * Planned upgraded version: [`AccountManager::forceHarvest()`](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/AccountManager.sol#L154)

As a result, **users can stake their tokens on land with a 0% tax rate if the landlord has not updated their `plotMetadata`** and **farm schnibbles without paying tax to the landlord until the landlord’s plot metadata is initialized in the `LandManager` contract (via `triggerPlotMetadata()` and `updatePlotMetadata()`)**.

```solidity
File: src/managers/LandManager.sol

131:     function stakeMunchable(
132:         address landlord,
133:         uint256 tokenId,
134:         uint256 plotId
135:     ) external override forceFarmPlots(msg.sender) notPaused {
136:         (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
137:         if (landlord == mainAccount) revert CantStakeToSelfError();
138:         if (plotOccupied[landlord][plotId].occupied)
139:             revert OccupiedPlotError(landlord, plotId);
140:         if (munchablesStaked[mainAccount].length > 10)
141:             revert TooManyStakedMunchiesError();
142:         if (munchNFT.ownerOf(tokenId) != mainAccount)
143:             revert InvalidOwnerError();
144: 
145:         uint256 totalPlotsAvail = _getNumPlots(landlord);
146:         if (plotId >= totalPlotsAvail) revert PlotTooHighError();
147: 
148:         if (
149:             !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
150:             munchNFT.getApproved(tokenId) != address(this)
151:         ) revert NotApprovedError();
152:         munchNFT.transferFrom(mainAccount, address(this), tokenId);
153: 
154:         plotOccupied[landlord][plotId] = Plot({
155:             occupied: true,
156:             tokenId: tokenId
157:         });
158: 
159:         munchablesStaked[mainAccount].push(tokenId);
160:         munchableOwner[tokenId] = mainAccount;
161: 
162:         toilerState[tokenId] = ToilerState({
163:             lastToilDate: block.timestamp,
164:             plotId: plotId,
165:             landlord: landlord,
166:@>           latestTaxRate: plotMetadata[landlord].currentTaxRate,
167:             dirty: false
168:         });
169: 
170:         emit FarmPlotTaken(toilerState[tokenId], tokenId);
171:     }
```

```solidity 
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---

247:         for (uint8 i = 0; i < staked.length; i++) {
                --- SNIPPED ---

280:             schnibblesTotal =
281:                 (timestamp - _toiler.lastToilDate) *
282:                 BASE_SCHNIBBLE_RATE;
283:             schnibblesTotal = uint256(
284:                 (int256(schnibblesTotal) +
285:                     (int256(schnibblesTotal) * finalBonus)) / 100
286:             );
287:             schnibblesLandlord =
288:@>           (schnibblesTotal * _toiler.latestTaxRate) / //@audit _toiler.latestTaxRate will be 0
289:                 1e18;
290: 
291:             toilerState[tokenId].lastToilDate = timestamp;
292:@>           toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]    //@audit update toilerState's latestTaxRate
293:                 .currentTaxRate;
294: 
295:@>          renterMetadata.unfedSchnibbles += (schnibblesTotal -
296:                 schnibblesLandlord);
297: 
298:             landlordMetadata.unfedSchnibbles += schnibblesLandlord;
299:             landlordMetadata.lastPetMunchable = uint32(timestamp);
300:             accountManager.updatePlayer(landlord, landlordMetadata);
301:             emit FarmedSchnibbles(
302:                 _toiler.landlord,
303:                 tokenId,
304:                 _toiler.plotId,
305:                 schnibblesTotal - schnibblesLandlord,
306:                 schnibblesLandlord
307:             );
308:         }
309:@>       accountManager.updatePlayer(mainAccount, renterMetadata);
310:     }
```
---

### Impact
Users can farm schnibbles on plots with a 0% tax rate without paying any tax to the landlord, which disrupts the platform's revenue model and incentives.

---
**Severity Clarification Aspects**:
* **Likelihood**: Can be considered **`Medium`** as there are many accounts that have locked tokens in the `LandManager`. This is evident from the fact that it was deployed over 55 days ago and the locked tokens are worth more than $1 million ([On-chain: LandManager](https://blastscan.io/address/0xEA091311Fc07139d753A6BBfcA27aB0224854Bae)).

* **Impact**: Can be considered **`Medium`** as it does not directly affect the landlord’s assets but results in a loss of benefits for landlords. Renters can exploit the issue by paying no tax on farming if the landlord's tax rate is not updated. Although landlords can eventually update their rates, renters can continue farming at the previous 0% tax rate until they farm again after the update.

---
### Proof of Concept
**Prerequisites**

- Bob has locked tokens at the `LockManager`, with enough tokens for 10 plots.
- The `AccountManager` contract is upgraded to support the `LandManager` when `forceHarvest()` is called.
- The `LandManager` is deployed at `block-10`.

---

1. **`block-11`**: Alice stakes her `MunchNFT` on Bob's land with a 0% tax rate, as Bob has not triggered `LandManager::triggerPlotMetadata()` or interacted with the `LockManager` to trigger `forceHarvest()` after the `LandManager` deployment.

```solidity
    toilerState[Alice's tokenId: 10] = ToilerState({
            lastToilDate: `block-11`,
            plotId: 0,
            landlord: Bob Account,
            latestTaxRate: plotMetadata[landlord].currentTaxRate => 0,
            dirty: false
        });
```

2. **`block-12`**: Bob triggers an update to his `plotMetadata`.

```solidity
    plotMetadata[Bob Account] = PlotMetadata({
            lastUpdated: block-12,
            currentTaxRate: DEFAULT_TAX_RATE
        });
```
This `currentTaxRate` will not apply to Alice's `toilerState` until Alice farms.

3. **`block-31`**: Alice farms. 
During the `schnibblesLandlord` calculation, Alice's `_toiler.latestTaxRate` (which was 0) will be used, even though Bob updated the `currentTaxRate` at `block-12`.

```solidity 
File: src/managers/LandManager.sol

232:     function _farmPlots(address _sender) internal {
233:         (
234:             address mainAccount,
235:             MunchablesCommonLib.Player memory renterMetadata
236:         ) = _getMainAccountRequireRegistered(_sender);
237: 
                --- SNIPPED ---

247:         for (uint8 i = 0; i < staked.length; i++) {
                --- SNIPPED ---

280:@>           schnibblesTotal =
281:               (timestamp - _toiler.lastToilDate) *   //@audit `block-31` - `block-11`
282:                 BASE_SCHNIBBLE_RATE;
283:             schnibblesTotal = uint256(
284:                 (int256(schnibblesTotal) +
285:                     (int256(schnibblesTotal) * finalBonus)) / 100
286:             );
287:             schnibblesLandlord =
288:@>           (schnibblesTotal * _toiler.latestTaxRate) / //@audit _toiler.latestTaxRate will be 0
289:                 1e18;
290: 
291:             toilerState[tokenId].lastToilDate = timestamp;
292:@>           toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]    //@audit update toilerState's latestTaxRate
293:                 .currentTaxRate;
294: 
295:@>          renterMetadata.unfedSchnibbles += (schnibblesTotal -
296:                 schnibblesLandlord);
297: 
298:             landlordMetadata.unfedSchnibbles += schnibblesLandlord;
299:             landlordMetadata.lastPetMunchable = uint32(timestamp);
300:             accountManager.updatePlayer(landlord, landlordMetadata);
301:             emit FarmedSchnibbles(
302:                 _toiler.landlord,
303:                 tokenId,
304:                 _toiler.plotId,
305:                 schnibblesTotal - schnibblesLandlord,
306:                 schnibblesLandlord
307:             );
308:         }
309:@>       accountManager.updatePlayer(mainAccount, renterMetadata);
310:     }
```

**Result**: Alice benefits by retrieving the full `schnibblesTotal` from `block-11` until `block-31`, even though Bob updated his `plotMetadata` immediately after Alice staked.

---

### Tools Used
* Manual Review
---
### Recommended Mitigation Steps
Apply unset `plotMetadata.currentTaxRate` checks in the `LandManager::stakeMunchable()`

```diff
File: src/managers/LandManager.sol

131:     function stakeMunchable(
132:         address landlord,
133:         uint256 tokenId,
134:         uint256 plotId
135:     ) external override forceFarmPlots(msg.sender) notPaused {
136:         (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
137:         if (landlord == mainAccount) revert CantStakeToSelfError();
+138:        if (plotMetadata[landlord].lastUpdated == 0)
+139              revert PlotMetadataNotUpdatedError();
            --- SNIPPED ---
171:     }
```
---