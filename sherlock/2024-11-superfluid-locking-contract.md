# Superfluid Locker System Report
Superfluid is the money streaming protocol, powering a real-time onchain economy. Start earning every second with streaming airdrops, rewards and yield.

* **nSLOC**: 1,656
* **Contest details**: https://audits.sherlock.xyz/contests/648

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: [Criteria for Issue Validity | Sherlock V2](https://docs.sherlock.xyz/audits/real-time-judging/judging)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[H-01](#h-01)| `FluidLocker::_getUnlockingPercentage()` will cause incorrect penalty calculations, impacting all users |🍅|

---

## <a name="h-01">[H-01]</a> `FluidLocker::_getUnlockingPercentage()` will cause incorrect penalty calculations, impacting all users

* **Severity**: High
* Source: [merlinboii - `FluidLocker::_getUnlockingPercentage()` cause incorrect unlock rate calculations for vesting unlock](https://github.com/sherlock-audit/2024-11-superfluid-locking-contract-judging/issues/64)

> Root causes duplicate of [#21](https://github.com/sherlock-audit/2024-11-superfluid-locking-contract-judging/issues/21) and [#10](https://github.com/sherlock-audit/2024-11-superfluid-locking-contract-judging/issues/10)

---

### Summary

The `FluidLocker::_getUnlockingPercentage()` has a **critical flaw in its calculation**, leading to a constant unlocking percentage of **2000 basis points (20%)**. This affects how penalties are calculated for users based on their chosen unlock period, potentially leading to unfair financial outcomes.

---

### Root Cause

The issue occurs because the function's use of **incorrect scaling** and **does not properly convert days to seconds**, results in an incorrect penalty calculation. This means the function always returns the same value, regardless of the actual unlock period.

More details provided below. 

#### Formula Breakdown

Here's the problematic formula:

```solidity
function _getUnlockingPercentage(uint128 unlockPeriod) internal pure returns (uint256 unlockingPercentageBP) {
    unlockingPercentageBP = (
        _PERCENT_TO_BP
            * (
                ((80 * _SCALER) / Math.sqrt(540 * _SCALER)) * (Math.sqrt(unlockPeriod * _SCALER) / _SCALER)
                    + 20 * _SCALER
            )
    ) / _SCALER;
}   
```

- **Factor A**: `((80 * _SCALER) / Math.sqrt(540 * _SCALER))`
- **Factor B**: `(Math.sqrt(unlockPeriod * _SCALER) / _SCALER)`
- **Factor C**: `(20 * _SCALER)`
- **_PERCENT_TO_BP**: `100`
- **_SCALER**: `1e18`

The formula simplifies to:

```solidity
unlockingPercentageBP = 
(100 * (factorA * factorB + factorC)) / _SCALER;
```

#### Take a look at Factor B, it fails as follows:

The calculation for `factorB` results **rounding down to zero** because:

- The numerator `Math.sqrt(unlockPeriod * _SCALER)` is always smaller than the denominator `_SCALER` (1e18), leading to a zero result.
- For `factorB` to be non-zero, `unlockPeriod` would need to be unrealistically large (greater than 1e18 seconds).

#### Take a look at Factor A, it fails as follows:

The formula uses `540` to represent days, but it **should be using seconds for accurate calculations**.

---

### Preconditions

- **Internal**: The function is called with a valid `unlockPeriod`.
- **External**: The contract is active within the Superfluid ecosystem, and users expect accurate calculations.

---

### Impact

This flaw affects all users by providing **incorrect UnlockFlowRates calculations** in the `FluidLocker::_calculateVestUnlockFlowRates()` using when the user decides to vesting unlock as the unlock period that should vary based on the chosen period.

---

### Proof of Concept

```solidity
// Example call to demonstrate the issue
uint256 unlockPercentage = _getUnlockingPercentage(7 days);
console.log("Unlock Percentage: ", unlockPercentage); // Always outputs 2000 BP
```

#### Setup
* Put the snippet below into the protocol test suite: `fluid/packages/contracts/test/FluidLocker.t.sol::FluidLockerTTETest` 
* Run test:
`forge test --mt 'testUnlockFlowRate' -vvv`

#### Coded PoC
<details>
  <summary>Coded PoC</summary>

```solidity
    function _helperCalculateUnlockFlowRatesCustom(uint128 unlockPeriod)
        internal
        pure
        returns (uint256 unlockingPercentageBP)
    {
        /// @notice Scaler used for unlock percentage calculation
        uint256 _SCALER = 1e18;

        /// @notice Scaler used for unlock percentage calculation
        uint256 _PERCENT_TO_BP = 100;

        /// @notice Fork from the FluidLocker::_getUnlockingPercentage()
        unlockingPercentageBP = (   
            _PERCENT_TO_BP
                * (
                    ((80 * _SCALER) / Math.sqrt(540 * _SCALER)) * (Math.sqrt(unlockPeriod * _SCALER) / _SCALER)
                        + 20 * _SCALER
                )
        ) / _SCALER;

    }

    function testUnlockFlowRate() external {
        uint256 min = _helperCalculateUnlockFlowRatesCustom(_MIN_UNLOCK_PERIOD);
        uint256 max = _helperCalculateUnlockFlowRatesCustom(_MAX_UNLOCK_PERIOD);

        assertEq(min, 2000);
        assertEq(max, 2000);
    }
```
</details>

#### Result
Results of running the test:
```bash
Ran 1 test for test/FluidLocker.t.sol:FluidLockerTTETest
[PASS] testUnlockFlowRate() (gas: 8821)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.39ms (1.19ms CPU time)

Ran 1 test suite in 143.72ms (11.39ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

---

### Mitigation

Update and ensure that the formula calculation accurately reflects the intended behavior, allowing the dynamic term to contribute properly.

The following is a suggested formula update: 
(ps. please note that the formula is not tested for the edge cases of minimum unlock period and maximum unlock period and should be tested before being used in production)

```solidity
function _getUnlockingPercentage(uint128 unlockPeriod) internal pure returns (uint256 unlockingPercentageBP) {
    unlockingPercentageBP = (
        _PERCENT_TO_BP
            * (
                ((80 * Math.sqrt(unlockPeriod * _SCALER)) / Math.sqrt(_MAX_UNLOCK_PERIOD * _SCALER))
                    + 20
            )
    );
}
```