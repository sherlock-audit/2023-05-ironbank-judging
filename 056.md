rvierdiiev

medium

# FlashLoan.maxFlashLoan may return incorrect result

## Summary
FlashLoan.maxFlashLoan may return incorrect result, because interests are not accrued.
## Vulnerability Detail
`FlashLoan.maxFlashLoan` function should return maximum amount then can be borrowed, according to [ERC3156](https://eips.ethereum.org/EIPS/eip-3156#lender-specification):
>The maxFlashLoan function MUST return the maximum loan possible for token

https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/flashLoan/FlashLoan.sol#L28-L46
```solidity
    function maxFlashLoan(address token) external view override returns (uint256) {
        if (!IronBankInterface(ironBank).isMarketListed(token)) {
            return 0;
        }

        DataTypes.MarketConfig memory config = IronBankInterface(ironBank).getMarketConfiguration(token);
        uint256 totalCash = IronBankInterface(ironBank).getTotalCash(token);
        uint256 totalBorrow = IronBankInterface(ironBank).getTotalBorrow(token);

        uint256 maxBorrowAmount;
        if (config.borrowCap == 0) {
            maxBorrowAmount = totalCash;
        } else if (config.borrowCap > totalBorrow) {
            uint256 gap = config.borrowCap - totalBorrow;
            maxBorrowAmount = gap < totalCash ? gap : totalCash;
        }

        return maxBorrowAmount;
    }
```

Because interest are not accrued in this function, that means that `totalBorrow` amount can be not up to date. As result bigger amount will be returned to user and when he will try to borrow it through the flashloan, then interests will be accrued and borrow will fail, because borrow cap will be reached.
## Impact
FlashLoan.maxFlashLoan doesn't correspond to ERC3156 and returns wrong amount.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Accrue interests before calculating max amount.