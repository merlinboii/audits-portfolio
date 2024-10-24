# Jala Swap Report
The first DEX on Chiliz to swap, wrap, provide liquidity, and stake fan tokens.

* **nSLOC**: 896
* **Contest details**: https://audits.sherlock.xyz/contests/233

## Disclaimer
This report contains the valid findings discovered by me (merlinboii). 

Severity Criteria: [Criteria for Issue Validity | Sherlock V2](https://docs.sherlock.xyz/audits/real-time-judging/judging)

---

## Findings

| ID | Description | Severity |
| :-: | - | :-: |
|[M-01](#m-01)| Potential swapping more than the given exact input amount |🍋|

---

## <a name="m-01">[M-01]</a> Potential swapping more than the given exact input amount

* **Severity**: Medium
* Source: [merlinboii - Potential swapping more than the given exact input amount](https://github.com/sherlock-audit/2024-02-jala-swap-judging/issues/88)
---
### Summary
The [JalaMasterRouter.swapExactTokensForTokens()](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L191-L215) allows callers to swap more than their exact input amount `amountIn`.

---
### Vulnerability Detail
The [JalaMasterRouter.swapExactTokensForTokens()](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L191-L215) passes the entire swapped token balance of the `JalaMasterRouter` as an input amount instead of accepting the exact input amount specified by the caller.

---
### Impact
Users can swap more than their expected input amount.

---
### Code Snippet
```solidity
File: JalaMasterRouter.sol

  function swapExactTokensForTokens(
          address originTokenAddress,
          uint256 amountIn,
          uint256 amountOutMin,
          address[] calldata path,
          address to,
          uint256 deadline
      ) external virtual override returns (uint256[] memory amounts, address reminderTokenAddress, uint256 reminder) {
          address wrappedTokenIn = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(originTokenAddress);
  
          require(path[0] == wrappedTokenIn, "MS: !path");
  
          TransferHelper.safeTransferFrom(originTokenAddress, msg.sender, address(this), amountIn);
          _approveAndWrap(originTokenAddress, amountIn);
          IERC20(wrappedTokenIn).approve(router, IERC20(wrappedTokenIn).balanceOf(address(this)));
  
          amounts = IJalaRouter02(router).swapExactTokensForTokens(
-->         IERC20(wrappedTokenIn).balanceOf(address(this)),
              amountOutMin,
              path,
              address(this),
              deadline
          );
          (reminderTokenAddress, reminder) = _unwrapAndTransfer(path[path.length - 1], to);
      }
```
---
### Tool used
Manual Review

---
### Recommendation
Updating the `JalaMasterRouter.swapExactTokensForTokens()` ensure it passes the exact input amount from users.

```diff
File: JalaMasterRouter.sol

  function swapExactTokensForTokens(
          address originTokenAddress,
          uint256 amountIn,
          uint256 amountOutMin,
          address[] calldata path,
          address to,
          uint256 deadline
      ) external virtual override returns (uint256[] memory amounts, address reminderTokenAddress, uint256 reminder) {
          address wrappedTokenIn = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(originTokenAddress);
  
          require(path[0] == wrappedTokenIn, "MS: !path");
  
          TransferHelper.safeTransferFrom(originTokenAddress, msg.sender, address(this), amountIn);
          _approveAndWrap(originTokenAddress, amountIn);
          IERC20(wrappedTokenIn).approve(router, IERC20(wrappedTokenIn).balanceOf(address(this)));

+       uint256 tokenInOffset = IChilizWrappedERC20(wrappedTokenIn).getDecimalsOffset();
+       uint256 exactAmountIn = amountIn * tokenInOffset
          amounts = IJalaRouter02(router).swapExactTokensForTokens(
-            IERC20(wrappedTokenIn).balanceOf(address(this)),
+            exactAmountIn,
              amountOutMin,
              path,
              address(this),
              deadline
          );
          (reminderTokenAddress, reminder) = _unwrapAndTransfer(path[path.length - 1], to);
      }
```
---