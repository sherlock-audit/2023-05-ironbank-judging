0x52

medium

# Flashloan#maxFlashloan and flashFee are not EIP-3156 compatible

## Summary

Flashloan#maxFlashloan and flashFee will return the incorrect values when a market has been borrow restricted such as when it has been soft delisted. In this scenario taking a flashloan in the borrow restricted is now impossible but Flashloan#maxFlashloan and flashFee continue to function as if it is possible, violating the EIP-3156 standard.

NOTE: Flashloan.sol has been explicitly labeled as fully EIP-3156 compatible in the readme. Since it is an explicit violation of the requirements laid out, I believe this should be considered a valid medium due to missing/broken functionality.

## Vulnerability Detail

From EIP-3156 standard:

`The maxFlashLoan function MUST return the maximum loan possible for token. If a token is not currently supported maxFlashLoan MUST return 0, instead of reverting`

When a market is borrow restricted the ironBank.borrow subcall in _loan will revert, meaning that the token is not currently supported and maxFlashloan should return 0.

[FlashLoan.sol#L28-L46](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/flashLoan/FlashLoan.sol#L28-L46)

    function maxFlashLoan(address token) external view override returns (uint256) {
        if (!IronBankInterface(ironBank).isMarketListed(token)) { <- @audit-issue borrow restricted markets will still be "listed"
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

The current implementation of maxFlashLoan only checks that the market has been listed but never checks that it is still possible to borrow. Since _loan requires borrowing from the market, calls to borrow restricted markets will revert. Following the EIP standard maxFlashLoan should return 0 in this scenario but the above code will return the value as if the market is fully functional.

`The flashFee function MUST return the fee charged for a loan of amount token. If the token is not supported flashFee MUST revert.`

According to the EIP-3156 standard flashFee must revert in this same scenario.

[FlashLoan.sol#L49-L55](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/flashLoan/FlashLoan.sol#L49-L55)

    function flashFee(address token, uint256 amount) external view override returns (uint256) {
        amount;

        require(IronBankInterface(ironBank).isMarketListed(token), "token not listed");

        return 0;
    }

We can see in the code above that this is not the case and borrow restricted markets, although incompatible, won't revert as expected.

## Impact

Operation risk for anyone integrating with flashloan, due to incompatibility with the EIP-3156 standard

## Code Snippet

[FlashLoan.sol#L28-L55](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/flashLoan/FlashLoan.sol#L28-L55)

## Tool used

Manual Review

## Recommendation

Consider borrow restricted markets and adjust functions to behave correctly when interacting with them