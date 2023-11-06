## 1. Low. OmniOracle.sol doesn't work with tokens returning symbol as bytes32
https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniOracle.sol#L48

### Vulnerability details
Some tokens (e.g. MKR) have metadata fields (name / symbol) encoded as bytes32 instead of the string prescribed by the ERC20 specification.
Thus such tokens can't be used in Band Oracle
```solidity
    function getPrice(address _underlying) external view returns (uint256) {
        OracleConfig memory config = oracleConfigs[_underlying];
        if (config.provider == Provider.Band) {
            IStdReference.ReferenceData memory data;
            if (_underlying == WETH) {
                data = IStdReference(config.oracleAddress).getReferenceData("ETH", USD);
            } else {
@>              data = IStdReference(config.oracleAddress).getReferenceData(IERC20Metadata(_underlying).symbol(), USD);
            }
            ...
    }
```
### Recommended Mitigation Steps
Use the BoringCrypto safeSymbol() function code with the returnDataToString() parsing function to handle the case of a bytes32 return value: https://github.com/boringcrypto/BoringSolidity/blob/ccb743d4c3363ca37491b87c6c9b24b1f5fa25dc/contracts/libraries/BoringERC20.sol#L15-L39

## 2. Low. Restrict updating isIsolatedCollateral from true to false and vice-versa
### Vulnerability details
https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniPool.sol#L508-L522

Current implementation allows to set new config. In case normal market will be marked isolated or vice versa, it will break internal accounting of OmniPool.sol. Because code assumes that this variable will never change

### Recommended Mitigation Steps
Explicitly disallow updating `isIsolatedCollateral` on configured markets

## 3. Low. Remove dust deposit amount after socializing loss
### Vulnerability details
https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniPool.sol#L397

There can be dust amount of deposit `socializeLoss` is called
```solidity
    function socializeLoss(address _market, bytes32 _account) external onlyRole(DEFAULT_ADMIN_ROLE) {
        uint8 borrowTier = getAccountBorrowTier(accountInfos[_account]);
        Evaluation memory eval = evaluateAccount(_account);
        uint256 percentDiff = eval.depositTrueValue * 1e18 / eval.borrowTrueValue;
@>      require(percentDiff < 0.00001e18, "OmniPool::socializeLoss: Account not fully liquidated, please call liquidate prior to fully liquidate account.");
        IOmniToken(_market).socializeLoss(_account, borrowTier);
        emit SocializedLoss(_market, borrowTier, _account);
    }
```

However it socializes all debt of user and leaves deposit to the user. But it should firstly reduce user's debt by that deposit, and then to socialize debt.
```solidity
    function socializeLoss(bytes32 _account, uint8 _trancheId) external nonReentrant {
        require(msg.sender == omniPool, "OmniToken::socializeLoss: Bad caller");
        uint256 totalDeposits = 0;
        for (uint8 i = _trancheId; i < trancheCount; ++i) {
            totalDeposits += tranches[i].totalDepositAmount;
        }
        OmniTokenTranche storage tranche = tranches[_trancheId];
        uint256 share = trancheAccountBorrowShares[_trancheId][_account];
@>      //@audit It is debt amount which must be socialized
        uint256 amount = Math.ceilDiv(share * tranche.totalBorrowAmount, tranche.totalBorrowShare); // Represents amount of bad debt there still is (need to ensure user's account is emptied of collateral before this is called)
        uint256 leftoverAmount = amount;
@>      //@audit full amount is socialized
        for (uint8 ti = trancheCount - 1; ti > _trancheId; --ti) {
            OmniTokenTranche storage upperTranche = tranches[ti];
            uint256 amountProp = (amount * upperTranche.totalDepositAmount) / totalDeposits;
            upperTranche.totalDepositAmount -= amountProp;
            leftoverAmount -= amountProp;
        }
        tranche.totalDepositAmount -= leftoverAmount;
        tranche.totalBorrowAmount -= amount;
        tranche.totalBorrowShare -= share;
        trancheAccountBorrowShares[_trancheId][_account] = 0;
        emit SocializedLoss(_account, _trancheId, amount, share);
    }
```
### Recommended Mitigation Steps
Subtract user's deposit before socializing debt
```diff
    function socializeLoss(bytes32 _account, uint8 _trancheId) external nonReentrant {
        require(msg.sender == omniPool, "OmniToken::socializeLoss: Bad caller");
        uint256 totalDeposits = 0;
+       uint256 userDepositShares = 0;
        for (uint8 i = _trancheId; i < trancheCount; ++i) {
            totalDeposits += tranches[i].totalDepositAmount;
+           userDepositShares += trancheAccountDepositShares[i][_account];
        }
        OmniTokenTranche storage tranche = tranches[_trancheId];
        uint256 share = trancheAccountBorrowShares[_trancheId][_account];
        uint256 amount = Math.ceilDiv(share * tranche.totalBorrowAmount, tranche.totalBorrowShare); // Represents amount of bad debt there still is (need to ensure user's account is emptied of collateral before this is called)
+       
+       amount -= userDepositShares * tranche.totalDepositAmount / tranche.totalDepositShare;
+       
        uint256 leftoverAmount = amount;
        for (uint8 ti = trancheCount - 1; ti > _trancheId; --ti) {
            OmniTokenTranche storage upperTranche = tranches[ti];
            uint256 amountProp = (amount * upperTranche.totalDepositAmount) / totalDeposits;
            upperTranche.totalDepositAmount -= amountProp;
            leftoverAmount -= amountProp;
        }
        tranche.totalDepositAmount -= leftoverAmount;
        tranche.totalBorrowAmount -= amount;
        tranche.totalBorrowShare -= share;
        trancheAccountBorrowShares[_trancheId][_account] = 0;
        emit SocializedLoss(_account, _trancheId, amount, share);
    }
```

## 4. R. `toAddress()` logic can be simpler without `ADDRESS_MASK`
https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/SubAccount.sol#L27

### Recommended Mitigation Steps
```diff
    function toAddress(bytes32 _account) internal pure returns (address) {
-       return address(uint160(uint256(_account) & ADDRESS_MASK));
+       return address(uint160(uint256(_account)));
    }
```

## 5. R. Remove unused variable from struct `ModeConfiguration`
Field `modeMarketCount` is never used, consider removing it