## Findings Summary

| Label | Description |
| - | - |
| [L-01](#l-01--entermarketserror-check-0-tranche-borrow-cap) | `enterMarkets()`error check `0 tranche` borrow cap|
| [L-02](#l-02-entermarketslack-of-limitation-on-the-length-of-the-market-risk-of-out_gas) | `enterMarkets()`Lack of limitation on the length of the market, risk of `OUT_GAS` |
| [L-03](#l-03-omnitokennoborrow-supplycap-dos-risk) | OmniTokenNoBorrow `supplyCap` DOS risk |
| [L-04](#l-04-_evaluateaccountinternal-markets-with-no-deposits-will-also-expire-and-be-liquidated) | _evaluateAccountInternal() Markets with no deposits will also expire and be liquidated |
| [L-05](#l-05-omnioraclegetprice--when-providerproviderother-its-very-confusing) |OmniOracle.getPrice()  when provider==Provider.Other It's very confusing|

## [L-01]  `enterMarkets()`error check `0 tranche` borrow cap
in `enterMarkets()` will check `IOmniToken(market).getBorrowCap(0) > 0`
But the user's `BorrowTier` is not always `0`, it could be `IsolatedMarket.riskTranche`
suggest check `getAccountBorrowTier(account)` borrow cap

```diff
    function enterMarkets(uint96 _subId, address[] calldata _markets) external {
...
-           require(IOmniToken(market).getBorrowCap(0) > 0, "OmniPool::enterMarkets: Market has no borrow cap for 0 tranche.");
+           require(IOmniToken(market).getBorrowCap(getAccountBorrowTier(_account)) > 0, "OmniPool::enterMarkets: Market has no borrow cap for 0 tranche.");
            newMarkets[i + existingMarkets.length] = market;
        }
        accountMarkets[accountId] = newMarkets;
        emit EnteredMarkets(accountId, _markets);
    }
```

## [L-02] `enterMarkets()`Lack of limitation on the length of the market, risk of `OUT_GAS`.
Currently there is no limit to the maximum number of markets that can enter
In `_evaluateAccountInternal()` we loop through all markets and calculate accordingly.
Since there is no limit on the length, there is a risk of `OUT_GAS`.
Suggest adding a limit
```diff
    function enterMarkets(uint96 _subId, address[] calldata _markets) external {
...
+      require(newMarkets.length <=25);
        accountMarkets[accountId] = newMarkets;
        emit EnteredMarkets(accountId, _markets);
    }
```

## L-03: OmniTokenNoBorrow `supplyCap` DOS risk
`OmniTokenNoBorrow.sol` will currently limit the global maximum supply `supplyCap`
Since both `deposit()` and `withdraw()` currently charge no fees

This maliciously deposits a large amount, resulting in close to `supplyCap`
thus preventing users from depositing more collateral to avoid liquidation
It is recommended to limit the maximum supply per user instead of a global supply limit

## L-04: _evaluateAccountInternal() Markets with no deposits will also expire and be liquidated
The current method `_evaluateAccountInternal()` will liquidate the market as soon as it expires, regardless of whether the user has made a deposit in the market or not.
It doesn't make sense. It should only affect the calculation of the liquidation result if there is a deposit.
Suggest to calculate expiration only if there is a deposit 
```diff
    function _evaluateAccountInternal(bytes32 _accountId, address[] memory _poolMarkets, AccountInfo memory _account)
        internal
        returns (Evaluation memory eval)
    {
...
            if (i < _poolMarkets.length) {
                market = _poolMarkets[i];
            } else {
                market = _account.isolatedCollateralMarket;
            }
            MarketConfiguration memory marketConfiguration_ = marketConfigurations[market];
-           if (marketConfiguration_.expirationTimestamp <= block.timestamp) {
-               eval.isExpired = true; // Must repay all debts and exit market to get rid of unhealthy account status if expired
-           }
            address underlying = IWithUnderlying(market).underlying();
            uint256 price = IOmniOracle(oracle).getPrice(underlying); // Returns price in 1e18
            uint256 depositAmount = IOmniTokenBase(market).getAccountDepositInUnderlying(_accountId);
            if (depositAmount != 0) {
+              if (marketConfiguration_.expirationTimestamp <= block.timestamp) {
+                   eval.isExpired = true; // Must repay all debts and exit market to get rid of unhealthy account status if expired
+              }

                ++eval.numDeposit;
                uint256 depositValue = (depositAmount * price) / PRICE_SCALE; // Rounds down
                eval.depositTrueValue += depositValue;
                uint256 collateralFactor = marketCount == 1
                    ? SELF_COLLATERALIZATION_FACTOR
                    : _account.modeId == 0 ? uint256(marketConfiguration_.collateralFactor) : uint256(mode.collateralFactor);
                eval.depositAdjValue += (depositValue * collateralFactor) / FACTOR_PRECISION_SCALE; // Rounds down
            }
            if (i >= _poolMarkets.length) {
                // Isolated collateral market. No borrow.
                continue;
            }
```

## [L-05] OmniOracle.getPrice()  when provider==Provider.Other It's very confusing.

in `OmniOracle.getPrice()` 
if config.provider == Provider.Other will return 
```
return IOmniOracle(config.oracleAddress).getPrice(_underlying) * (PRICE_SCALE / 1e18) / (10 ** IERC20Metadata(_underlying).decimals());
```

It is recommended not to use this interface `IOmniOracle.getPrice()`.
Because the current `getPrice()` of `OmniOracle.sol` is already a converted quantity based on `_underlying.decimals()`.
Example: chainlink USD decimals = 6
 when `_underlying.decimals() == 4`, price = ORACLE_USD * (10 ** (18-6)) * (10** (18-4))

Suggestion: add new interface`ICustomOracle.getOraclePrice()`

```diff
    function getPrice(address _underlying) external view returns (uint256) {
...
        } else if (config.provider == Provider.Other) {
-           return IOmniOracle(config.oracleAddress).getPrice(_underlying) * (PRICE_SCALE / 1e18) / (10 ** IERC20Metadata(_underlying).decimals());
+           return ICustomOracle(config.oracleAddress).getOraclePrice(_underlying) * (PRICE_SCALE / 1e18) / (10 ** IERC20Metadata(_underlying).decimals());
        } else {
            revert("OmniOracle::getPrice: Invalid provider.");
        }
    }
```

or  use new define of `getPrice()`
```diff
    function getPrice(address _underlying) external view returns (uint256) {
...
        } else if (config.provider == Provider.Other) {
-           return IOmniOracle(config.oracleAddress).getPrice(_underlying) * (PRICE_SCALE / 1e18) / (10 ** IERC20Metadata(_underlying).decimals());
+           return IOmniOracle(config.oracleAddress).getPrice(_underlying); // with _underlying.decimals() to 18
        } else {
            revert("OmniOracle::getPrice: Invalid provider.");
        }
    }
```
