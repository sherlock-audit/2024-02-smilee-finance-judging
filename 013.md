Special Saffron Moth

high

# Risk-free deposits when vaults are being killed

## Summary

Users can abuse the flow of killing a vault and gain a share of the last premiums with nearly no risk and therefore steal this share from the other LPs who took the risk.

## Vulnerability Detail

The `rescueShares` function is used to withdraw funds and also take profits/losses from a dead vault. This function takes the `epochPricePerShare` (if the LP made profit or loss) from the previous epoch instead of the current one as it is done in the normal flow:

```solidity
uint256 pricePerShare = epochPricePerShare[epoch.previous];
```

The flow to earn premiums as LP under normal conditions looks like this:

- Epoch_1: User calls `deposit` (deposit happens next epoch)
- Epoch_2: User calls `initiateWithdraw` (withdraw is claimable next epoch)
- Epoch_3: User calls `completeWithdraw`

=> Users receive an amount of `baseTokens` depending on the `epochPricePerShare` of Epoch_2. The user does not know if this trade will lead to profits or losses, as it is a bet on the stability of the `sideToken`. Therefore the user takes a risk by providing liquidity to option traders.

The flow to earn premiums by abusing the edge case of a dying vault looks like this:

- Epoch_1: Admin calls `killVault` (vaults status is set to killed)
- Epoch_1: User sees that at the end of the epoch, LPs make profit and call `deposit` (still possible as the vault modifier checks if the vaults status is dead, not killed)
- After_Epoch_1: Vaults status is now dead and the user calls `rescueShares` to receive premiums from Epoch_1

=> As we can see the user receives an amount of `baseTokens` depending on the `epochPricePerShare` of Epoch_1. The same epoch the user deposited in. Therefore the user can earn premiums without the risk of a full epoch and by depositing at the end of the epoch the risk for the user is minimal, only a black swan event could lead to a loss. This steals funds from the other LPs who deposited the round before and took the risk of a full epoch.

The following POC can be implemented in the `VaultDeathTest.t.sol` test file:

```solidity
function testRiskFreeDepositsOnDyingVaults() public {
    assertEq(0, baseToken.balanceOf(alice));

    // admin kills vault
    vm.prank(tokenAdmin);
    vault.killVault();

    // alice is able to deposit into the vault after it has been killed
    VaultUtils.addVaultDeposit(alice, 100e18, tokenAdmin, address(vault), vm);

    // epoch ends
    Utils.skipDay(true, vm);
    vm.prank(tokenAdmin);
    vault.rollEpoch();
    Utils.skipDay(true, vm);

    // alice is able to rescueShares
    vm.prank(alice);
    vault.rescueShares();
    assertEq(100e18, baseToken.balanceOf(alice));
}
```

## Impact

Users can abuse the situation of a dying vault to earn a share of the premiums without the risks of providing liquidity and therefore steal a share from the other honest LPs.

## Code Snippet

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/Vault.sol#L573

## Tool used

Manual Review

## Recommendation

Do not allow deposits on killed vaults and only allow to kill vaults when no epoch is running (as if killed at the end of the epoch front running the call would also be possible).
