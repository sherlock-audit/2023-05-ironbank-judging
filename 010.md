rvierdiiev

medium

# PriceOracle.getPrice doesn't check for stale price

## Summary
PriceOracle.getPrice doesn't check for stale price. As result protocol can make decisions based on not up to date prices, which can cause loses.
## Vulnerability Detail
`PriceOracle.getPrice` function is going to provide asset price using chain link price feeds.
https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L66-L72
```soldiity
    function getPriceFromChainlink(address base, address quote) internal view returns (uint256) {
        (, int256 price,,,) = registry.latestRoundData(base, quote);
        require(price > 0, "invalid price");

        // Extend the decimals to 1e18.
        return uint256(price) * 10 ** (18 - uint256(registry.decimals(base, quote)));
    }
```
This function doesn't check that prices are up to date. Because of that it's possible that price is not outdated which can cause financial loses for protocol.
## Impact
Protocol can face bad debt.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
You need to check that price is not outdated by checking round timestamp.