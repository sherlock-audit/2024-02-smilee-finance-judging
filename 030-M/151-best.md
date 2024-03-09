Gentle Fiery Locust

medium

# rescueShares() will not allow LPs to withdraw part of the fees

## Summary

The _burn function in DVP can be executed even when the Vault is dead. The reason is that traders should be able to close their positions even if the vault is dead. The problem arises because the rescueShares function uses the share price calculated in the last epoch. Fees from unclosed positions will not be included because at that moment, they will be pending payoff. Consequently, fees from closing positions after the price has been calculated will not be paid out to LPs. They will remain locked in the Vault.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/Vault.sol#L557-L581

## Impact

Lock of funds and losses for the LPs.

## Code Snippet

Above.

## Tool used

Manual Review

## Recommendation

A possible solution is to recalculate the price on every burn if the Vault is dead.
