shealtielanz

medium

# Use `WBTC/USD` price feed rather than falling back to `WBTC/BTC` and `BTC/USD` feeds to calculate the price of `WBTC` to `USD`.

[Price Oracle](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L30)
## Summary
The developers said in the documentation that they couldn't find the `WBTC/USD` price feed on `Chainlink` so they decided to use an alternative method to get the price, which may cause discrepancies in the actual price and could be exploited, so I did some research and found a way for the developers to get the `WBTC/USD` price feeds for accurate prices.
## Vulnerability Detail
The `IronBank` Docs [Link Here](https://docs.ib.xyz/lending-market/price-oracle)

It states as follows:

> There are some tokens that we cannot find the corresponding price feeds on Chainlink. Therefore, we use price feed alternative instead.
For WBTC, we use the combination of two price feeds WBTC/BTC and BTC/USD, to get the WBTC price in USD quote as the source.
> - WBTC/BTC: 0xfdFD9C85aD200c506Cf9e21F1FD8dd01932FBB23
> - BTC/USD: 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c

But this is not so, after my research, I found that you can get the price feed of any token to USD by making a request to `Chainlink` and they'll provide it and also make it available for use.
Just do as follows:
- Click this link https://data.chain.link/
- You can always request a new price feed by clicking on the `Request price feed` at the Top-Right, Link here ~ [Request Data Feed](https://chain.link/contact?ref_id=datafeeds&_ga=2.225795794.953922268.1685899202-1136617684.1679934424)

**This way you can get any price feed that isn’t on Chainlink right now and they'll create and send it to you.**
## Impact
Using getting the price of `WBTC/USD` via calculations with `WBT/BTC` and `BTC/USD` can cause deep discrepancies between the calculated price and the actual price if there is any issue or attack on any of the price feeds, and overall using the actual price feed improves the Protocol and mitigates any issue that could arise due to mistake in wrong calculation. 
## Code Snippet
- https://docs.ib.xyz/lending-market/price-oracle.
- https://chain.link/contact?ref_id=datafeeds&_ga=2.225795794.953922268.1685899202-1136617684.1679934424
## Tool used
`Manual Review` & `A Little Research `

## Recommendation
Used the actual price feed of `WBTC/USD` rather than calculating its price from `WBTC/BTC` and `BTC/USD` price feeds