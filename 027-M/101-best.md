Low Mauve Crab

medium

# Incorrect logic to check for financial data approximation, leading to incorrect IG payoff and pricing.

## Summary
When financial data (`kA, kB`) are approximated, the protocol intends to pause the DVP contract after rolling epoch. However, the check for approximation is not done correctly, so the protocol will not pause at times when it should.

## Vulnerability Detail

Approximation of `kA` would occur within `FinanceIGPrice.liquidityRange` as shown here:
```solidity
function liquidityRange(LiquidityRangeParams calldata params) public pure returns (uint256 kA, uint256 kB) {
        uint256 mSigmaT = ud(params.sigma)
            .mul(ud(params.sigmaMultiplier))
            .mul(ud(params.yearsToMaturity).sqrt())
            .unwrap();

        kA = ud(params.k).mul(sd(SignedMath.neg(mSigmaT)).exp().intoUD60x18()).unwrap(); // k*e^(-mSigmaT)
        kB = ud(params.k).mul(ud(mSigmaT).exp()).unwrap(); //

        if (kA == 0) { //@note: this is the approximation of kA, when it is calculated as zero.
            kA = 1;
        }
    }
    
```
Now, the following logic is used to check approximation and then pause:
```solidity
if (FinanceIG.checkFinanceApprox(financeParameters)) {
    _pause();
    emit PausedForFinanceApproximation();
}
```
```solidity
function checkFinanceApprox(FinanceParameters storage params) public view returns (bool isFinanceApproximated) {
    uint256 kkartd = FinanceIGPrice._kkrtd(params.currentStrike, params.kA);
    uint256 kkbrtd = FinanceIGPrice._kkrtd(params.currentStrike, params.kB);
    return kkartd == 1 || kkbrtd == 1;
  }
```
```solidity
function _kkrtd(uint256 k, uint256 krange) public pure returns (uint256) {
        UD60x18 res = (ud(k).mul(ud(krange))).sqrt();
        uint256 resUnwrapped = res.unwrap();
        if (resUnwrapped == 0) {
            return 1;
        }
        return resUnwrapped;
    }
```

For this to return true, it requires $\sqrt{k*k_A}$ (or the same with $k_B$) to equal $0$. However, since approximated $k_A$ set to $1$, this will not be true. Thus, even though approximation of $k_A$ occured, it won't be detected by the check, and the protocol will not pause. 

## Impact
Trading will be enabled in the DVP contract even though incorrect liquidity range values have been set, leading to incorrect calculation of price and payoff for IG options, disrupting the intended functionality of the protocol. 

## Code Snippet
**FinanceIG.sol**
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/lib/FinanceIG.sol#L151-L155

## Tool used

Manual Review

## Recommendation
In `checkFinanceApprox`, check whether $k_A$ is equal to 1, not whether $\sqrt{k*k_A}$ is equal to zero.
