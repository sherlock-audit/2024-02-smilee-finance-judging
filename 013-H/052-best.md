Bent Crimson Goat

high

# Incorrect liquidity calculations when burning expired positions whose notional token does not use 18 decimals of precision.

## Summary

The function [`shareOfPayoff`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L120), used to compute the residual payoff for an expired position, mutates the input parameter [`Amount memory amount_`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L123C23-L123C30) resulting in miscalculations back up in the caller context.

## Vulnerability Detail

When a position is expired, the [`DVP`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/DVP.sol) relies upon the function [`shareOfPayoff`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L120) when burning a position:

```solidity
// Compute the payoff to be paid:
Amount memory payoff_ = _liquidity[expiry].shareOfPayoff(strike, amount, _baseTokenDecimals);
paidPayoff = payoff_.getTotal();

// Account transfer of setted aside payoff:
_liquidity[expiry].decreasePayoff(strike, payoff_);
```

This results in the `amount` parameter passed into [`shareOfPayoff`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L120) running the risk of being mutated when handling non-wad decimals and possessing a non-zero `up` or `down` attribute, since the transformation is not unset.

For simplicity, let's consider the non-zero `up` attribute case:

```solidity
if (amount_.up > 0) {
  amount_.up = AmountsMath.wrapDecimals(amount_.up, decimals);
  used_.up = AmountsMath.wrapDecimals(used_.up, decimals);
  accountedPayoff_.up = AmountsMath.wrapDecimals(accountedPayoff_.up, decimals);

  // amount : used = share : payoff
  payoff_.up = (amount_.up*accountedPayoff_.up)/used_.up;
  payoff_.up = AmountsMath.unwrapDecimals(payoff_.up, decimals);
}
```

Notice here that the child property `up` of `amount_` is written to using `AmountsMath.wrapDecimals(amount_.up, decimals)`, meaning any value assignment here will additionally take affect in the caller's context as well as the local sequence since they are both referencing the same area of memory.

Normally when `decimals` are `18`, this sequence has no effect and returns the same value.

Conversely, for lesser token decimal precisions (i.e. USDC), [the value is scaled up](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/AmountsMath.sol#L55):

```solidity
function wrapDecimals(uint256 amount, uint8 decimals) external pure returns (uint256) {
  if (decimals == _DECIMALS) {
    return amount;
  }
  if (decimals > _DECIMALS) {
    revert TooManyDecimals();
  }
  return amount * (10 ** (_DECIMALS - decimals));
}
```

Continuing with USDC as an example, an `amount_.up` value of `1e6` will be amplified into `1e6 * 1e12 = 1e18` not only within the scope of [`shareOfPayoff`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L120), but also in the scope of [`_burn`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/DVP.sol#L231).

The result is that all subsequent reads of `amount.up` after the call to [`shareOfPayoff`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L120) are erroneously interacting with the scaled version, resulting in incorrect calculations for all subsequent references, such as calculation of the premium, sales fees and the application of position modifications.

## Impact

Due to this incorrect accounting, there are a wide range of affected values and resulting failures throughout the burn flow, including:

1. A significant amplification to `entryPremiumProp`, due to the numerator now being amplified but not the denominator.
2. Excessive fees paid to the `FeeManager`.
3. An exaggerated decrease of `Position` attributes.
  3.1 For smaller positions, this could result in `revert` through underflow.
  3.2 Though for whale positions, numerical underflow may be avoided and would instead convert into heavy loss for the owner.

## Code Snippet

```solidity
/**
 * @notice Get the share of residual payoff set aside for the given expired position
 * @param strike The position strike
 * @param amount_ The position notional
 * @param decimals The notional's token number of decimals
 * @return payoff_ The owed payoff
 * @dev It relies on the calls of decreaseUsage and decreasePayoff after each position is decreased
 */
function shareOfPayoff(
    Info storage self,
    uint256 strike,
    Amount memory amount_,
    uint8 decimals
) external view returns (Amount memory payoff_) {
    Amount memory used_ = self.used[strike];
    Amount memory accountedPayoff_ = self.payoff[strike];

    if (amount_.up > 0) {
        amount_.up = AmountsMath.wrapDecimals(amount_.up, decimals);
        used_.up = AmountsMath.wrapDecimals(used_.up, decimals);
        accountedPayoff_.up = AmountsMath.wrapDecimals(accountedPayoff_.up, decimals);

        // amount : used = share : payoff
        payoff_.up = (amount_.up * accountedPayoff_.up) / used_.up;
        payoff_.up = AmountsMath.unwrapDecimals(payoff_.up, decimals);
    }

    if (amount_.down > 0) {
        amount_.down = AmountsMath.wrapDecimals(amount_.down, decimals);
        used_.down = AmountsMath.wrapDecimals(used_.down, decimals);
        accountedPayoff_.down = AmountsMath.wrapDecimals(accountedPayoff_.down, decimals);

        payoff_.down = (amount_.down * accountedPayoff_.down) / used_.down;
        payoff_.down = AmountsMath.unwrapDecimals(payoff_.down, decimals);
    }
}
```

## Tool used

Foundry

## Recommendation

Ideally, a function call should avoid mutable writes to parameters unless the function accepts exclusive ownership of the parameter being passed, which indeed does not hold true for this scenario. In this regard, it is recommended that [`shareOfPayoff`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L120) should deep-clone the input `amount_` for the modifications necessary only to local calculations.

Failing this, it is advised that [`shareOfPayoff`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Notional.sol#L120) should [`unwrapDecimals`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/AmountsMath.sol#L65) after the precision-sensitive calculations have been performed.
