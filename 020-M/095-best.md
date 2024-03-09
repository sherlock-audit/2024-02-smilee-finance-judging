Low Mauve Crab

high

# When rolling epoch, the pendingPayoff is incremented with the incorrect value, leading to loss for LPs

## Summary
See Detail.

## Vulnerability Detail
In `Vault._beforeRollEpoch()`, the following block of code attempts to account the new pending payoffs (new vault liabilities):
```solidity
if (vaultCt.v0() > 0) {
            _accountResidualPayoffs(IPriceOracle(_getPriceOracle()).getPrice(sideToken, baseToken));

            uint256 payoffToReserve = _residualPayoff();

            vaultCt.reservePayoff(payoffToReserve);
        }
```
The first function `_accountResidualPayoffs` effectively calculates payoff via the formula `payoff = usedLiquidity * payoffPercentage`.
Then it updates the payoff for this latest epoch using the calculated payoff.

Then, the second function obtains the payoff for the latest epoch (which was just set by the first function)

Then the third function effectively sets the vault's `_state.liquidity.newPendingPayoffs` to the new payoff.

The issue is that a portion of the payoff would have already been reserved from previous epochs, since `usedLiquidity` carries over across epochs and is only decreased when traders burn their position to collect payoff. However `newPendingPayoffs` is still being set equal to the **entire** payoff. This severely and incorrectly inflates the value of `state.liquidity.pendingPayoffs` when the vault's epoch rolls.

## Impact
As long as the payoff inflation doesn't exceed the vault's TVL (in that case the epoch rolling would revert which is a critical DoS), the traders receive more payoff than deserved, leading to loss for LPs. This breaks the core property that LPs risk is to always be about the same as what a uniswap v3 LP undertakes.

## Code Snippet
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol#L342-L348

## Tool used

Manual Review

## Recommendation
Call `vaultCt.reservePayoff()` with only the new payoff due from this epoch, not `_residualPayoff();` as that represents the entire pending payoff.

For example, a solution would be to calculate _residualPayoff(); (call this payoffBefore), then call `_accountResidualPayoffs()`, and then again re-calculate _residualPayoff(); (call this payoffAfter). Then 
Example fix:

```diff
if (vaultCt.v0() > 0) {
+            uint256 payoffBefore = _residualPayoff();
            _accountResidualPayoffs(IPriceOracle(_getPriceOracle()).getPrice(sideToken, baseToken));
+          uint256 payoffAfter = _residualPayoff();
-           uint256 payoffToReserve = _residualPayoff();
+           uint256 payoffToReserve = payoffAfter - payoffBefore; // can't underflow since minimum increase in payoff is 0

            vaultCt.reservePayoff(payoffToReserve);
        }
```