bitsurfer

medium

# Liquidations will be frozen, when the oracle go offline or a token's price dropping to zero

## Summary

Liquidations will be frozen, when the oracle go offline or a token's price dropping to zero

## Vulnerability Detail

In certain exceptional scenarios, such as when oracles go offline/paused or token prices plummet to zero, liquidations will be temporarily suspended.

When liquidator want to `liquidate()` an account because of bad debt, it will check the `_isLiquidatable()` and loop over through the `userEnteredMarkets` to check if `debtValue > liquidationCollateralValue` for an account. To calculate this oracle is called to get the asset price.

If this call to oracle is corrupted due to offline/paused, the `liquidate()` call function will reverted. More over if the price is returning `0` it will be reverted because `require(assetPrice > 0, "invalid price");`

## Impact

During critical periods, there is a risk that liquidations may not be feasible when they are most needed by the protocol. This can lead to a situation where the value of users' assets declines below their outstanding debts, effectively disabling any motivation for liquidation. Consequently, this can potentially push the protocol into insolvency, posing significant challenges and potential financial instability.

## Code Snippet

https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L1065-L1092

```js
File: IronBank.sol
499:         require(_isLiquidatable(borrower), "borrower not liquidatable");

File: IronBank.sol
1065:     function _isLiquidatable(address user) internal view returns (bool) {
...
1069:         address[] memory userEnteredMarkets = allEnteredMarkets[user];
1070:         for (uint256 i = 0; i < userEnteredMarkets.length; i++) {
...
1079:             uint256 assetPrice = PriceOracleInterface(priceOracle).getPrice(userEnteredMarkets[i]);
1080:             require(assetPrice > 0, "invalid price");
...
1090:         }
1091:         return debtValue > liquidationCollateralValue;
1092:     }

File: PriceOracle.sol
66:     function getPriceFromChainlink(address base, address quote) internal view returns (uint256) {
67:         (, int256 price,,,) = registry.latestRoundData(base, quote);
68:         require(price > 0, "invalid price");
```

## Tool used

Manual Review

## Recommendation

Implement a protective measure to mitigate this potential risk. For example, enclose any oracle get price function within a try-catch block, and incorporate fallback logic within the catch block to handle scenarios where access to the Chainlink oracle data feed is denied. Or use second oracle for backup situation.
