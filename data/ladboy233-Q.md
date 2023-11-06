| Issue  | Description |
|--------|-------------|
| Admin validation | should validate if duplicate market exists when admin set mode |
| Account Count Limitation | In `OmniTokenNoBorrow.sol` and `OmniToken.sol`, only the first 25 accounts are counted toward the user balance. |
| Airdrop Token Handling | `OmniTokenNoBorrow.sol` and `OmniToken.sol` are not equipped to handle airdrop tokens effectively. |
| Borrow Cap Validation | `OmniPool.sol`'s `enterMarkets` does not check if the borrow cap is reached before entering the market. |
| OpenZeppelin Version | The protocol uses OpenZeppelin version 4.8.0; it is recommended to update to a more recent version (4.9+). |
| Hardcoded WETH Address | The WETH address is hardcoded for ETH mainnet, which could cause issues in a multichain environment. |
| Oracle Validation | Additional validation is needed for the oracle when `Provider.Other` is used to ensure price reliability. |
| Token Oracle Validation by Symbol | The protocol should not validate the token oracle by symbol due to the potential for symbol changes. |
| Token Decimals Caching | Token decimals should be cached in the oracle to avoid fetching them every time, which is a costly operation. |
| Price Feed Deprecation Handling | Operations such as liquidation, borrowing, and withdrawal could be blocked if the oracle price becomes stale or the price feed is deprecated. |


# Should validate if duplicate market exists when admin set mode

when admin [set the mode](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L537)

```solidity
   function setModeConfiguration(ModeConfiguration memory _modeConfiguration)
        external
        onlyRole(MARKET_CONFIGURATOR_ROLE)
    {
        if (_modeConfiguration.expirationTimestamp <= block.timestamp) { revert("OmniPool::setModeConfiguration: Bad expiration timestamp."); }
        modeCount++;
        modeConfigurations[modeCount] = _modeConfiguration;
        emit SetModeConfiguration(modeCount, _modeConfiguration);
    }
```

the code should validate if duplicate markets exists in the mode configuration to avoid double evaluting and counting user collateral

# In OmniTokenNoBorrow.sol and OmniToken.sol, only first 25 account counts toward the user balance

balanceOf in OmniToken only iteravte over [first 25 account](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniToken.sol#L457) to count the user balance

but user can create sub account in uint96 space

then when external api replies on the balanceOf to query the user balance, the returned result undercounts under balance if the user's sub account number exceed 25

if the protocol has the need to expose the user balance

the protocol should update the user balance when user deposit
the protocol should reduce the user balance when user withdraw or transfer

and then return the user balance in O(1) time instead of in O(n) time when running the for loop

# OmniTokenNoBorrow.sol and OmniToken.sol is not capable of handling airdrop token

OmniTokenNoBorrow.sol and OmniToken.sol is likely to hold a large amount of asset

in case when there is a project issue airdrop based on the token balance, 

the token pool is not capable of claim airdrop and cannot transfer the airdrop out from the token pool

then the airdrop is lost

# In OmniPool.sol, enterMarkets does not validate if borrow cap is reached

when enterMarket, the protocol only validate when the [borrow cap is not 0](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L109)

```solidity
require(IOmniToken(market).getBorrowCap(0) > 0, "OmniPool::enterMarkets: Market has no borrow cap for 0 tranche.");
```

but does not validate the BorrowCap of the market is reachced

in case when BorrowCap of the market is reachced, user cannot really borrow but still paying the gas to call accure every time for that added market

# Use a more recent version of openzeppelin

the protocol openzepplein used is 4.8.0, but the newly version is 4.9+

recommend using a more recent version of openzeppelin to avoid bugs

# WETH address should not be hardcoded in multichain setting

the protocol wants to deploy the contract in both eth mainnet and BSC

but the WETH address is [hardcoded](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniOracle.sol#L24) to ETH address in mainnet

```solidity
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2; // Need to hardcode WETH address per network deployed for Band
```

as the comment suggest,

the protocol needs to hardcode WETH address per network deployed for Band

# should perform additional validation for oracle when Provider.Other is used

in this [line of code](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniOracle.sol#L58)

```solidity
} else if (config.provider == Provider.Other) {
	return IOmniOracle(config.oracleAddress).getPrice(_underlying) * (PRICE_SCALE / 1e18) / (10 ** IERC20Metadata(_underlying).decimals());
} 
```

in case when token oracle is not provided by band or chainlink, Provider.Other is used

if the provider.OTHER is used, the protocol should validate if the price is stale or return 0 as well instead of use the oracle price directly

because the oracle price deposit collateral value and incorrect value can result in loss of fund (false liquidation / overborrowing)

# should not validate the token oracle by symbol

When band oracle is used, if the token is not WETH, the type of the oracle query is constructed [based on the token symbol](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniOracle.sol#L48)

```
data = IStdReference(config.oracleAddress).getReferenceData(IERC20Metadata(_underlying).symbol(), USD);
```

however, the underlying address returned symbol can change, so it is recommend to cache the token symbol and validate by token address instead of using the returned token symbol to construct query data

# should cache the token decimals in oracle

when oracle is used, every time the [token decimal](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniOracle.sol#L56) is fetched when return the oracle price

```solidity
return uint256(answer) * (PRICE_SCALE / (10 ** IChainlinkAggregator(config.oracleAddress).decimals())) / (10 ** IERC20Metadata(_underlying).decimals());
```

but the underlying token returned decimal can change

so it is recommend the cache the decimal for token

# liquidation / borrow / withdraw revert and blocked in the oracle price becomes stale or price feed is deprecated

chainlink reguarly deprecate the feed

https://docs.chain.link/data-feeds/deprecating-feeds?network=deprecated&page=1

in case when the feed is deprecated,

calling IChainlinkAggregator(config.oracleAddress).latestRoundData() wilil result in revert

or when the price is stale, both chainlink oracle or band oracle revert

when it is correct to not use the incorrect or misreported price

the liquidation flow / borrow / withdraw flow that [replies on the oracle price](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L251) is [blocked and revert](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniOracle.sol#L55) until the feed is updated by admin or the price not become stale





