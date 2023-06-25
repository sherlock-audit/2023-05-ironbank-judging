# Issue H-1: supplyNativeToken will strand ETH in contract if called after ACTION_DEFER_LIQUIDITY_CHECK 

Source: https://github.com/sherlock-audit/2023-05-ironbank-judging/issues/361 

## Found by 
0x52, evilakela
## Summary

supplyNativeToken deposits msg.value to the WETH contract. This is very problematic if it is called after ACTION_DEFER_LIQUIDITY_CHECK. Since onDeferredLiqudityCheck creates a new context msg.value will be 0 and no ETH will actually be deposited for the user, causing funds to be stranded in the contract. 

## Vulnerability Detail

[TxBuilderExtension.sol#L252-L256](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/extensions/TxBuilderExtension.sol#L252-L256)

    function supplyNativeToken(address user) internal nonReentrant {
        WethInterface(weth).deposit{value: msg.value}();
        IERC20(weth).safeIncreaseAllowance(address(ironBank), msg.value);
        ironBank.supply(address(this), user, weth, msg.value);
    }

supplyNativeToken uses the context sensitive msg.value to determine how much ETH to send to convert to WETH. After ACTION_DEFER_LIQUIDITY_CHECK is called, it enters a new context in which msg.value is always 0. We can outline the execution path to see where this happens:

`execute > executeInteral > deferLiquidityCheck > ironBank.deferLiquidityCheck > onDeferredLiquidityCheck (new context) > executeInternal > supplyNativeToken`

When IronBank makes it's callback to TxBuilderExtension it creates a new context. Since the ETH is not sent along to this new context, msg.value will always be 0. Which will result in no ETH being deposited and the sent ether is left in the contract.

Although these funds can be recovered by the admin, it may can easily cause the user to be unfairly liquidated in the meantime since a (potentially significant) portion of their collateral hasn't been deposited. Additionally in conjunction with my other submission on ownable not being initialized correctly, the funds would be completely unrecoverable due to lack of owner.

## Impact

User funds are indefinitely (potentially permanently) stuck in the contract. Users may be unfairly liquidated due to their collateral not depositing.

## Code Snippet

[TxBuilderExtension.sol#L252-L256](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/extensions/TxBuilderExtension.sol#L252-L256)

## Tool used

Manual Review

## Recommendation

msg.value should be cached at the beginning of the function to preserve it across contexts



## Discussion

**ibsunhub**

Confirm the issue!

However, we believe the correct modification is to pass `msg.value` through the whole external call and make `deferLiquidityCheck` function payable.

**0xffff11**

Valid high. I also think the fix from sponsor is most accurate.

**IAm0x52**

Passing through `msg.value` will result in it being nonfunctional in the event that `supplyNativeToken` is called before `ACTION_DEFER_LIQUIDITY_CHECK` since it will deposit msg.value into WETH. Then when it calls `deferLiquidityCheck` it would again attempt to send `msg.value` which would cause it to revert due to lack of funds. 

My suggestion would be to cache msg.value as an internal storage variable (i.e. _msgValue) at the beginning of [execute](https://github.com/sherlock-audit/2023-05-ironbank/blob/9ebf1702b2163b55479624794ab7999392367d2a/ib-v2/src/extensions/TxBuilderExtension.sol#L101). Adjust [supplyNativeToken](https://github.com/sherlock-audit/2023-05-ironbank/blob/9ebf1702b2163b55479624794ab7999392367d2a/ib-v2/src/extensions/TxBuilderExtension.sol#L253-L255) to use that storage variable rather than `msg.value`. After the supply to ironBank reset this variable to 0. This allows you to preserve the msg.value across all contexts

**ibsunhub**

The situation mentioned is same with #192.
The solution sounds viable and better. Will work on a fix according to the recommendation.

**0xffff11**

@ibsunhub  You mean that #192 should be a dup of this one?

**ibsunhub**

No, just think they are related.

# Issue M-1: Liquidations will be frozen, when the oracle go offline or a token's price dropping to zero 

Source: https://github.com/sherlock-audit/2023-05-ironbank-judging/issues/433 

## Found by 
3agle, Angry\_Mustache\_Man, BenRai, MohammedRizwan, Ocean\_Sky, R-Nemes, Schpiel, berlin-101, bitsurfer, josephdara, shaka
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



## Discussion

**ibsunhub**

It doesn't make any sense to liquidate an asset whose price is 0. It won't add value to debt nor collateral. It just can't have a zero-price asset on a lending protocol.

We thought about creating a backing oracle to be ready to replace ChainLink if ChainLink's price feed goes wrong. However, it also creates a trust issue since we control the price of the backing oracle and we could switch to it at will. For now, we stick with ChainLink. If the price from ChainLink is inaccurate (including being 0), we pause the borrow / repay / liquidate on this token.

**0xffff11**

I do believe this is an actual issue. Oracle failure should not pause liquidations, Another fix I can recommend (having a fallback oracle is the best), it is that for every fetch, the last price is stored on a variable. If the oracle fails, the last price on the variable will be the one used instead. 

**ibsunhub**

There are different failure for ChainLink price feed. If the price is not updated, retrieving the stale price from ChainLink is basically the same with the solution that stores a variable. Another concern is that we believe this fix (store the previous price on a variable) will not solve the root cause and will increase a significant amount of gas consumption for user operations (additional read / write storage during every price retrieval).

I think the best solution here is still a backing oracle. However, that's another topic to create a sub-system to deal with the trust issue and the availability issue.

**0xffff11**

Yes, I agree with your point, backing oracle should be the way to go. I think tellor oracles are very common as the fallback oracle. Might be worth checking those out.  Therefore, I believe that you confirm that the issue is valid, right? @ibsunhub  

**ib-tycho**

We maintain our position that this issue is not valid. It should be noted that prominent lending protocols like Compound v3 Comet, AAVE, and Euler do not implement fallback oracles or address Chainlink's potential downtime. While having fallback oracles can be beneficial, we consider them to be optional rather than essential for protocol functionality.

**0xffff11**

Issue will be kept as a medium at the moment because there is a chance of the price going to 0 and the scenario would be given.

# Issue M-2: getPriceFromChainlink() doesn't check If Arbitrum sequencer is down in Chainlink feeds 

Source: https://github.com/sherlock-audit/2023-05-ironbank-judging/issues/440 

## Found by 
0x52, 0xMAKEOUTHILL, Angry\_Mustache\_Man, Arabadzhiev, Arz, Aymen0909, BenRai, Breeje, Brenzee, BugBusters, Delvir0, HexHackers, Ignite, Jaraxxus, Kodyvim, Madalad, MohammedRizwan, Ocean\_Sky, Proxy, R-Nemes, SovaSlava, ast3ros, berlin-101, bin2chen, bitsurfer, branch\_indigo, deadrxsezzz, devScrooge, josephdara, kutugu, n1punp, n33k, ni8mare, p0wd3r, plainshift-2, rvierdiiev, santipu\_, sashik\_eth, shaka, simon135, sl1, sufi, tnquanghuy0512, toshii, tsvetanovv, turvec, vagrant
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



## Discussion

**0xffff11**

Valid medium

**ib-tycho**

Regarding the mistake in the contest details mentioned in the `README`, we apologize for any confusion caused. When we stated that we would deploy on Arbitrum and Optimism, we meant that we would make the necessary modifications before deployment. This is our standard practice of maintaining contracts with different branches, same as what we did in v1: https://github.com/ibdotxyz/compound-protocol/branches

We are aware of the absence of a registry on OP and Arb, as pointed out by some individuals. We would like to inquire if it is possible to offer the minimum reward for an oracle issue on L2. Thank you.

**ib-tycho**

We'll fix this when deploying on L2, but we disagree with Severity. I would consider this as Low

**0xffff11**

According to past reports and sponsor confirmed that they will fix the issue. The issue will remain as a medium.

