Low Mauve Crab

high

# Incorrect check in Vault::_beforeRollEpoch can lead to user fund loss due to unpayable liabilities

## Summary

The check for insufficient vault liquidity is done incorrectly, leading to user fund loss since liabilities to traders will not be able to be paid off by the vault. 

## Vulnerability Detail

Before rolling the epoch in the vault, it checks to ensure that the liabilities (payoffs) can be met through the vault's funds:

```solidity
if (lockedLiquidity < _state.liquidity.newPendingPayoffs) {
    revert InsufficientLiquidity( 
        bytes4(keccak256("_beforeRollEpoch()::lockedLiquidity <= _state.liquidity.newPendingPayoffs"))
    );
}
```

However, this only considers the new pending payoffs from the latest epoch, and does not consider all the other pending payoffs from previous epochs. This means that the check may pass, but `lockedLiquidity` may still be insufficient to cover the entirity of all the pending payoffs.

## Impact

The vault will not be able to meet the payoffs of all traders, causing loss of user funds.

## Code Snippet

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/Vault.sol#L604-L608

## Tool used

Manual Review

## Recommendation
Fix the check so that it includes the previous pending payoffs as well:

```diff
-if (lockedLiquidity < _state.liquidity.newPendingPayoffs) {
+if (lockedLiquidity < _state.liquidity.newPendingPayoffs + _state.liquidity.pendingPayoffs) {
    revert InsufficientLiquidity( 
        bytes4(keccak256("_beforeRollEpoch()::lockedLiquidity <= _state.liquidity.newPendingPayoffs"))
    );
}
```