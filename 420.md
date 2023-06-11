devScrooge

medium

# DOS for supplyPToken due to not approving 0 amount first

## Summary 
There are some tokens like, USDT, which will revert when approving or executing `safeIncreaseAllowance` is the allowance was not previosuly set to 0

## Vulnerability Detail
The `TxBuilderExtension.sol` contract implements a function called `supplyPToken` which is used for the users to supply tokens for a market on the `IronBank` contract:

```solidity
 function supplyPToken(address user, address pToken, uint256 amount) internal nonReentrant {
        address underlying = PTokenInterface(pToken).getUnderlying();
        IERC20(underlying).safeTransferFrom(user, pToken, amount);
        PTokenInterface(pToken).absorb(address(this));
        IERC20(pToken).safeIncreaseAllowance(address(ironBank), amount);
        ironBank.supply(address(this), user, pToken, amount);
    }
```

There is a specific line which is increase the allowance of the `pToken` for the IronBank to be able to execute the supply function:

```solidity
 IERC20(pToken).safeIncreaseAllowance(address(ironBank), amount);
```

The problem is that if `pToken` is one of the tokens that reverts if the allowance was not first set to 0 before increasing it to other amount, such as USDT, the transaction will revert and a denial of service will take place.

## Impact
Denial of service for the users that execute the `supplyPToken` function in on the `TxBuilderExtension`contract by calling `execute` for supplying tokens to a market which has a token that reverts, such as USDT, as underlying token.


## Code Snippet
https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/extensions/TxBuilderExtension.sol#L380

## Tool used

Manual Review

## Recommendation
Approve 0 amount first.
```solidity
IERC20(pToken).safeApprove(address(ironBank), 0); 
IERC20(pToken).safeIncreaseAllowance(address(ironBank), amount);
```

