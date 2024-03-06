Low Mauve Crab

high

# Incorrect payoff calculation leading to loss for LPs

## Summary
Payoff calculated is double what its meant to be.

## Vulnerability Detail
According to page 3 of the [IG financial engineering docs](https://drive.google.com/file/d/1d80IHxHfFUA_poH5J1hmkRov9yK9Wusm/view), the payoff for a bull DVP is calculated via $V_0\frac{1 + \frac{S_t}{K} - 2\sqrt{\frac{S_t}{K}}}{ \theta}$ . (this is assuming that the price is within liquidity range).

$V_0$ represents the residualAmountUp and $\frac{1 + \frac{S_t}{K} - 2\sqrt{\frac{S_t}{K}}}{ \theta}$ represents the percentage payoff.

However when calculating the payoffs within `Finance.computeResidualPayoffs`, it calculates it via:
```solidity
payoffUp_ = ud(residualAmountUp).mul(ud(percentageUp).mul(convert(2))).unwrap();
```
This is incorrect since it multiplies the result by 2 with `.mul(convert(2))` but this is not specified in the formula within the financial engineering docs.

## Impact
Higher residual payoffs than what is correct. This can lead to 2 impacts. If the inflated payoff is high enough, it can lead to not enough baseTokens in the vault to pay the liabilities, making it impossible to roll epoch. Otherwise if the inflated payoff doesnt exceed vault TVL, the LPs lose more money than what is correct, so LPs take on way more risk than normal.

## Code Snippet
**Finance.sol**
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/Finance.sol#L26

## Tool used
Manual Review

## Recommendation
Don't multiply by 2 in `Finance.computeResidualPayoffs` as that leads to incorrect payoff calculation.