## [L-01] When initializing the variables of an OmniToken, ot verifying if the borrow cap for the tranch #0 is > 0 will cause the OmniToken not to be borrowable until the admin sets again the borrowCap for tranch# > 0

When initializing the parameters of an OmniToken, [the argument `_borrowCaps` is not checked to validate if the tranch# 0 will be assigned with a borrowCap > 0.](https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniToken.sol#L63) If the tranch #0 borrow cap is 0, it means that it can't access any of the liquidity from the higher tranches!

  - Taking as referencethe setTrancheBorrowCaps() function, the tranch# 0 must have a borrowCap > 0
```solidity
    function initialize(address _omniPool, address _underlying, address _irm, uint256[] memory _borrowCaps) external initializer {
        ...
        //@audit-issue => If the tranch #0 borrow cap is 0, it means that it can't access any of the liquidity from the higher tranches!
        trancheBorrowCaps = _borrowCaps;
        ...
    }
```

**Fix:**
- Add a check to validate that tranch# 0 borrowCap is > 0
```solidity
    function initialize(address _omniPool, address _underlying, address _irm, uint256[] memory _borrowCaps) external initializer {
        ...
+       require(_borrowCaps[0] > 0, "Invalid borrow caps, must always allow 0 to borrow.");
        trancheBorrowCaps = _borrowCaps;
        ...
    }
```
---
## [L-02] Not fetching all the user subaccounts when using the balanceOf()
- The [existing implementation only pulls the first 25 subaccounts](https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniToken.sol#L455-L461), if a main account has more than 25 subaccs, the main account owner can't fetch the balances of the rest of his accounts
```solidity
    function balanceOf(address _owner) external view returns (uint256) {
        uint256 totalDeposit = 0;
        for (uint96 i = 0; i < MAX_VIEW_ACCOUNTS; ++i) {
          //@audit-issue => The totalDeposit of the owner will be only the sum of the first 25 subaccounts!
            totalDeposit += balanceOfAccount[_owner.toAccount(i)];
        }
        return totalDeposit;
    }
```

**Fix:**
- Allow the user to input the index from where it would like to start iterating, example, if user inputs index as 10, iterate from the subbaccs 10 to 35.
  - In this way, the account owner can get the balance of all his subaccounts, it might require a couple of calls, but it can get it done.
```solidity
+   function balanceOf(address _owner, uint8 index) external view returns (uint256) {
        uint256 totalDeposit = 0;
+       for (uint96 i = index; i < MAX_VIEW_ACCOUNTS; ++i) {
            totalDeposit += balanceOfAccount[_owner.toAccount(i)];
        }
        return totalDeposit;
    }
```

----

## [L-03] No need to use the outflowtokens, the on-fee transfer is deducted from the amount being transferred and taken away from the receiver, not from the sender
- The [outflowTokens](https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/WithUnderlying.sol#L48-L53) does an extra of unnecesary steps, the intention is to account for any transfer fees that could've been charged, but, the transfer fees are applied to the receiver.
  - Example, User A sends 10 Tokens and the fee is 10%:
    - User A will get deducted the 10 Tokens that is sending from his balance
    - The contract will charge 1 token as fee (the 10%)
    - User B will only receive 9 tokens (because of the transfer fees)
  
- So, the sender will only get deducted the amount that is sending, nothing more, nothing less.

**Fix:**
- Remove the lines of code to take a balance snapshot and compute the difference after the transfer is completed
```solidity
function _outflowTokens(address _to, uint256 _amount) internal returns (uint256) {
-   uint256 balanceBefore = IERC20(underlying).balanceOf(address(this));
    IERC20(underlying).safeTransfer(_to, _amount);
+   return _amount;    
-   uint256 balanceAfter = IERC20(underlying).balanceOf(address(this));
-   return balanceBefore - balanceAfter;
}
```

