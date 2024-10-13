# Midas - Instant Minter/Redeemer Report
This is issuing RWA tokens. This contest focuses on the new smart contract codebase that introduces new exciting features like the ability to instantly mint or redeem tokens, alongside new oracles.

* **nSLOC**: 1,405 
* **Contest details**: https://audits.sherlock.xyz/contests/495

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: [Criteria for Issue Validity | Sherlock V2](https://docs.sherlock.xyz/audits/real-time-judging/judging)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[M-01](#m-01)|Protocol will fail to enforce proper token redemption limits, leading to potential over-redemptions by users|🍋|

---

## <a name="m-01">[M-01]</a> Protocol will fail to enforce proper token redemption limits, leading to potential over-redemptions by users

* **Severity**: Medium
* Source: [merlinboii - Protocol will fail to enforce proper token redemption limits, leading to potential over-redemptions by users.](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer-judging/issues/34)
---

### Summary
The missing check and update of the token operation allowance in the [RedemptionVault._approveRequest()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVault.sol#L313-L348) when the admin fulfills the redemption request will allow users to redeem the payment token beyond its allowance by using the [RedemptionVault.redeemRequest()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVault.sol#L194-L204).

---
### Root Cause
In [RedemptionVault._approveRequest()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVault.sol#L313-L348), there is a missing call to [_requireAndUpdateAllowance()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/abstract/ManageableVault.sol#L492-L506) when the admin fulfills the token redemption request.

[RedemptionVault._approveRequest()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVault.sol#L313-L348)

```solidity
File: RedemptionVault.sol
313:     function _approveRequest(
314:         uint256 requestId,
315:         uint256 newMTokenRate,
316:         bool isSafe
317:     ) internal {
---
328:         if (request.tokenOut != MANUAL_FULLFILMENT_TOKEN) {
329:             uint256 tokenDecimals = _tokenDecimals(request.tokenOut);
330: 
331:             uint256 amountTokenOutWithoutFee = _truncate(
332:                 (request.amountMToken * newMTokenRate) / request.tokenOutRate,
333:                 tokenDecimals
334:             );
335: 
336:             _tokenTransferFromTo(
337:                 request.tokenOut,
338:                 requestRedeemer,
339:                 request.sender,
340:                 amountTokenOutWithoutFee, 
341:                 tokenDecimals
342:             );
343:         }
---
348:     }
```

[_requireAndUpdateAllowance()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/abstract/ManageableVault.sol#L493-L506)

```solidity
File: ManageableVault.sol
493:      * @dev check if operation exceed token allowance and update allowance
494:      * @param token address of token
495:      * @param amount operation amount (decimals 18)
496:      */
497:     function _requireAndUpdateAllowance(address token, uint256 amount)
498:         internal
499:     {   
500:         uint256 prevAllowance = tokensConfig[token].allowance;
501:         if (prevAllowance == MAX_UINT) return;
502: 
503:         require(prevAllowance >= amount, "MV: exceed allowance");   //@audit 004
504: 
505:         tokensConfig[token].allowance -= amount;
506:     }
```

---
### Internal pre-conditions
The token allowance configuration needs to be set to a specific amount (not `MAX_UINT`) by the admin, as the allowance will only be checked and updated if a specific amount is set. For tokens with an allowance set to `MAX_UINT`, the check and update will be bypassed.

[_requireAndUpdateAllowance()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/abstract/ManageableVault.sol#L493-L506)

```solidity
File: ManageableVault.sol
493:      * @dev check if operation exceed token allowance and update allowance
494:      * @param token address of token
495:      * @param amount operation amount (decimals 18)
496:      */
497:     function _requireAndUpdateAllowance(address token, uint256 amount)
498:         internal
499:     {   
500:         uint256 prevAllowance = tokensConfig[token].allowance;
501: @     if (prevAllowance == MAX_UINT) return;
502: 
503:         require(prevAllowance >= amount, "MV: exceed allowance");   //@audit 004
504: 
505:         tokensConfig[token].allowance -= amount;
506:     }
```

---
### External pre-conditions
The vulnerability allows any user who can call the [`RedemptionVault.redeemRequest()`](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVault.sol#L194-L204). The normal requirements for this function to be executed are as follows:

1. Users need to hold `mToken`.
2. The [`RedemptionVault.redeemRequest()`](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVault.sol#L194-L204) must not be paused.
3. Users must not be blacklisted or sanctioned, and they must be on the green list (if enabled).

---
### Impact
* The protocol suffers an approximate loss of tokens due to the inability to enforce proper redemption limits.
* The protocol cannot accurately enforce token allowance checks, causing over-redemptions by users.

---
### Mitigation
Apply the [_requireAndUpdateAllowance()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/abstract/ManageableVault.sol#L493-L506) function in the [RedemptionVault._approveRequest()](https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVault.sol#L313-L348) to enforce proper allowance checks for payment token redemptions.

```diff
File: RedemptionVault.sol
313:     function _approveRequest(
314:         uint256 requestId,
315:         uint256 newMTokenRate,
316:         bool isSafe
317:     ) internal {
---
328:         if (request.tokenOut != MANUAL_FULLFILMENT_TOKEN) {
329:             uint256 tokenDecimals = _tokenDecimals(request.tokenOut);
330: 
331:             uint256 amountTokenOutWithoutFee = _truncate(
332:                 (request.amountMToken * newMTokenRate) / request.tokenOutRate,
333:                 tokenDecimals
334:             );
335: 
+336:             _requireAndUpdateAllowance(request.tokenOut, amountTokenOutWithoutFee);
337: 
338:             _tokenTransferFromTo(
339:                 request.tokenOut,
340:                 requestRedeemer,
341:                 request.sender,
342:                 amountTokenOutWithoutFee,   //>18
343:                 tokenDecimals
344:             );
345:         }
---
350:     }
```
---