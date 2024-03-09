Special Saffron Moth

medium

# Insufficient checks allow traders to create unbalanced positions by selling a part of one side of a smilee option

## Summary

The protocol implements checks in the mint and burn function of IGs to prevent unbalanced positions. Only bull, bear, or balanced smilee (straddle) positions should be allowed. By opening a smilee (straddle) and selling only a part of one side of the position an unbalanced position can be created. This leads to an incorrect calculation so that a part of user funds are lost.

## Vulnerability Detail

The reason is an insufficient check in the `burn` / `sell` function of the `DVP` and `PositionManager` contract:

```solidity
if ((amount.up > 0 && amount.down > 0) && (amount.up != amount.down)) {
    // reverts
}
```

As we can see the check above does not prevent the following action:

- User buys a smilee (straddle) option
- User sells only a part of one side of the position
- An unbalanced position is created and the user loses funds as calculations do not expect that

This could probably lead to bigger problems in the protocol (for example delta hedging) as the protocol does not expect unbalanced positions.

The following POC can be implemented in the `IG.t.sol` test file:

```solidity
function testUnbalancePosition() public {
    uint256 currEpoch = ig.currentEpoch();
    uint256 strike = ig.currentStrike();

    // buy smilee (straddle like) option
    uint256 buyAmt = 1e18;
    TokenUtils.provideApprovedTokens(address(0x10), baseToken, alice, address(ig), 2*buyAmt, vm);
    (uint256 expectedMarketValue, ) = ig.premium(strike, buyAmt, buyAmt);
    vm.prank(alice);
    ig.mint(alice, strike, buyAmt, buyAmt, expectedMarketValue, 1e18, 0);

    // sell half of one side so 1/4 of the buy amount
    uint256 sellAmt = 5e17;
    vm.prank(alice);
    (expectedMarketValue, ) = ig.payoff(currEpoch, strike, buyAmt, buyAmt);
    vm.prank(alice);
    ig.burn(currEpoch, alice, strike, sellAmt, 0, expectedMarketValue, 1e18);

    bytes32 posId = keccak256(abi.encodePacked(alice, ig.currentStrike()));

    (uint256 amount,,,) = ig.positions(posId);

    // but only half of the buy amount is left in the position therefore the user lost 1/4 of the position
    assertEq(buyAmt / 2, amount);
}
```

## Impact

Unbalanced positions can be created and mess up the protocol calculations.

## Code Snippet

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/DVP.sol#L257-L262

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/periphery/PositionManager.sol#L228-L231

## Tool used

Manual Review

## Recommendation

Add checks that if the position is a smilee (straddle) position, the user should not be able to sell only a part of one side of the position in `DVP.sol` and in the `PositionManager.sol` file.
