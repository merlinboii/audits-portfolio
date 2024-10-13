# Biconomy - Nexus Report
The Biconomy offers an Account Abstraction toolkit that enables the simplest UX on your dApp, wallet, or appchain. Built on top of ERC 4337, Biconomy offers a full-stack solution for tapping into the power of our Smart Accounts Platform, Paymasters, and Bundlers. Nexus is a suite of contracts for Modular Smart Accounts compliant with ERC-7579 and ERC-4337, developed by Biconomy. It aims to enhance modularity and security for smart contract accounts, providing a flexible framework for various use cases.

* **nSLOC**: 1,433 
* **Contest details**: https://codehawks.cyfrin.io/c/2024-07-biconomy

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: 
* [How to Evaluate a Finding Severity | CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity)
* [How to Determine a Finding Validity | CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-determine-a-finding-validity)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[L-01](#l-01)| Create account from `RegistryFactory` contract reverts due to unsorted external `attesters[]`|🫑|

---

## <a name="l-01">[L-01]</a> Create account from `RegistryFactory` contract reverts due to unsorted external `attesters[]`

* **Severity**: Low
* Source: [Create account from `RegistryFactory` contract reverts due to unsorted external `attesters[]`](https://codehawks.cyfrin.io/c/2024-07-biconomy/s/125)
---
### Summary

Unsorted external attesters cause the `RegistryFactory` contract to be unable to create the account, as it fails to validate the given module since the non-compliant `attesters[]` are used with the [`ERC-7484`](https://eips.ethereum.org/EIPS/eip-7484).

---
### Vulnerability Details

The external `attesters[]` of the `RegistryFactory` contract are not guaranteed to be sorted when initialized in the constructor and are not sorted when the owner updates them via `addAttester()` and `removeAttester()`.

```Solidity
\\Location: RegistryFactory.sol

    function addAttester(address attester) external onlyOwner { 
@>        attesters.push(attester);  //  Potentially, it could not be sorted when added
    }

    function removeAttester(address attester) external onlyOwner {
        for (uint256 i = 0; i < attesters.length; i++) {
            if (attesters[i] == attester) {
@>              attesters[i] = attesters[attesters.length - 1];   // Potentially, it could not be sorted when removed
                attesters.pop();
                break;
            }
        }
    }
```

The external `REGISTRY` compliant with the [`ERC-7484`](https://eips.ethereum.org/EIPS/eip-7484) and used in the `isModuleAllowed()` function expects that the given external `attesters[]` are sorted to interact with the `REGISTRY.check()`.

> [ERC-7484 Required Registry functionality:: `check` functions](https://github.com/ethereum/ercs/blob/master/ERCS/erc-7484.md#check-functions)
> " The `attesters` provided **MUST** be unique and **sorted** and **the Registry MUST revert if they are not. "**

```solidity
\\Location: RegistryFactory.sol

    function isModuleAllowed(address module, uint256 moduleType) public view returns (bool) {
 @> REGISTRY.check(module, moduleType, attesters, threshold);
        return true;
    }
```

This unsorted `attesters[]` disrupts the `createAccount()` flow by reverting when invoking `isModuleAllowed()` to validate the given module and ensure all modules are whitelisted, resulting in the reversion in creating the `Nexus` account using the `RegistryFactory` contract.

```solidity
\\Location: RegistryFactory.sol

function createAccount(bytes calldata initData, bytes32 salt) external payable override returns (address payable) {

        // snipped

        // Ensure all modules are whitelisted
        for (uint256 i = 0; i < validators.length; i++) {
@>         require(isModuleAllowed(validators[i].module, MODULE_TYPE_VALIDATOR), ModuleNotWhitelisted(validators[i].module));
        }

        for (uint256 i = 0; i < executors.length; i++) {
@>         require(isModuleAllowed(executors[i].module, MODULE_TYPE_EXECUTOR), ModuleNotWhitelisted(executors[i].module));
        }

@>         require(isModuleAllowed(hook.module, MODULE_TYPE_HOOK), ModuleNotWhitelisted(hook.module));

        for (uint256 i = 0; i < fallbacks.length; i++) {
@>         require(isModuleAllowed(fallbacks[i].module, MODULE_TYPE_FALLBACK), ModuleNotWhitelisted(fallbacks[i].module));
        }

       // snipped

    }
```

---
### Impact

The unsorted `attesters[]` can cause the `RegistryFactory` contract to revert when attempting to create a new account.&#x20;

Although the **trusted** attesters are provided by the **owner** during the deployment phase, and the owner can compute and use sorted attester addresses before modification, the likelihood of this issue occurring is not considered low.&#x20;

This is due to the arbitrary nature of address (20-bytes random) and the logic in the `addAttester()` and `removeAttester()` , which shows that there is no sorting logic present, making **it highly likely for the `attesters[]` array to become unsorted when modified**.

---
### Proof of Concept

**Initial State:**

* Using the [`TrustManagerExternalAttesterList`](https://github.com/rhinestonewtf/registry/blob/main/src/core/TrustManagerExternalAttesterList.sol) contract as an example of the `REGISTRY` contract compliant with `ERC-7484`.

**Step 1**:&#x20;

Deploy the `RegistryFactory` contract and assign the `attesters` as `[0x101, 0x202, 0x303, 0x404, 0x505]` (SORTED).

```Solidity
\\Location: RegistryFactory.sol

    constructor(address implementation_, address owner_, IERC7484 registry_, address[] memory attesters_, uint8 threshold_) Stakeable(owner_) {
        require(implementation_ != address(0), ImplementationAddressCanNotBeZero());
        require(owner_ != address(0), ZeroAddressNotAllowed());
        REGISTRY = registry_;  //> TrustManagerExternalAttesterList
        attesters = attesters_;  //> [0x101, 0x202, 0x303, 0x404, 0x505]
        threshold = threshold_;
        ACCOUNT_IMPLEMENTATION = implementation_;
    }
```

**Step 2:**&#x20;

Owner removes `0x101` attester => the new attesters become \*\*UNSORTED \*\*= `[0x505, 0x202, 0x303, 0x404]` (the unsorted logic also arises in the `addAttester()`).

```solidity
\\Location: RegistryFactory.sol

    function removeAttester(address attester: 0x101) external onlyOwner {
        for (uint256 i = 0; i < attesters.length; i++) {
            if (attesters[i] == attester) {
                attesters[i] = attesters[attesters.length - 1]; //> `0x505` wil replace the value at the index that stores `0x101`
                attesters.pop();
                break;
            }
        }
    }
```

**Step 3:**&#x20;

User attempts to `createAccount()` but they obtain the revert as `REGISTRY.check(module, moduleType, attesters, threshold);` reverts.

```solidity
\\Location: RegistryFactory.sol

    function isModuleAllowed(address module, uint256 moduleType) public view returns (bool) {
 @> REGISTRY.check(module, moduleType, attesters, threshold);
        return true;
    }
```

```Solidity
\\ TrustManagerExternalAttesterList.sol (example REGISTRY)

     /**
     * @inheritdoc IERC7484
     */
    function check(address module, ModuleType moduleType, address[] calldata attesters: `[0x505, 0x202, 0x303, 0x404]`, uint256 threshold) external view {
        uint256 attestersLength = attesters.length;
        if (attestersLength == 0 || threshold == 0) {
            revert NoTrustedAttestersFound();
        } else if (attestersLength < threshold) {
            revert InsufficientAttestations();
        }

        address _attesterCache; // 0x000, used to store the PREV attester when loop
        for (uint256 i; i < attestersLength; ++i) {
            address attester = attesters[i]; //i_0, attester= attesters[0] = 0x505 || i_1, attester=attesters[1] = 0x202

@>          if (attester <= _attesterCache) revert InvalidTrustedAttesterInput(); //i_0, 0x505 <= 0x000 (FALSE) || i_1, 0x202 <= 0x505 (TRUE) **REVERT**
            else _attesterCache = attester; //i_0, _attesterCache = 0x505 ||
            if ($getAttestation(module, attester).checkValid(moduleType)) {
                --threshold;
            }
            if (threshold == 0) return;
        }
        revert InsufficientAttestations();
    }
```

**Outcome:**&#x20;

Temporary inability to create an account via the `RegistryFactory` contract until the owner modifies the attesters to be sorted via `addAttester()` and `removeAttester()`.

---
### Tools Used
Manual Review

---
### Recommendations

Ensure that the `attesters[]` array is sorted whenever it is modified. This can be achieved by implementing sorting logic  in the `addAttester()` and `removeAttester()`  or applyinh the [`solady/LibSort`](https://github.com/Vectorized/solady/blob/main/src/utils/LibSort.sol).

Moreover, applying the sorting logic in the `constructor` ensures that the `attesters`  is sorted upon initialization.

```diff
\\Location: RegistryFactory.sol

+import { LibSort } from "solady/src/utils/LibSort.sol";

contract RegistryFactory is Stakeable, INexusFactory {
+   using LibSort for address[];

// snipped


       function addAttester(address attester) external {
        attesters.push(attester);
+       address[] memory attesters_ = attesters;
+       attesters_.insertionSort();
+       attesters = attesters_;
    }

    function removeAttester(address attester) external {
        for (uint256 i = 0; i < attesters.length; i++) {
            if (attesters[i] == attester) {
                attesters[i] = attesters[attesters.length - 1];
                attesters.pop();
+               address[] memory attesters_ = attesters;
+               attesters_.insertionSort();
+               attesters = attesters_;
                break;
            }
        } 
    }

// snipped

}
```
---