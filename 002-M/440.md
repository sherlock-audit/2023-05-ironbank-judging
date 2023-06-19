bin2chen

medium

# getPriceFromChainlink() doesn't check If Arbitrum sequencer is down in Chainlink feeds

## Summary
When utilizing Chainlink in L2 chains like Arbitrum, it's important to ensure that the prices provided are not falsely perceived as fresh, even when the sequencer is down. This vulnerability could potentially be exploited by malicious actors to gain an unfair advantage.

## Vulnerability Detail
There is no check:
getPriceFromChainlink
```solidity
    function getPriceFromChainlink(address base, address quote) internal view returns (uint256) {
@>      (, int256 price,,,) = registry.latestRoundData(base, quote);
        require(price > 0, "invalid price");

        // Extend the decimals to 1e18.
        return uint256(price) * 10 ** (18 - uint256(registry.decimals(base, quote)));
    }
```


## Impact

could potentially be exploited by malicious actors to gain an unfair advantage.

## Code Snippet

https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L66-L72

## Tool used

Manual Review

## Recommendation

code example of Chainlink:
https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code
