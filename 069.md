Bent Crimson Goat

medium

# `PositionManager` NFTs can be maliciously modified prior to a transfer.

## Summary

The [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol#L91) represents underlying [`DVP`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol) positions as tradeable NFTs, allowing them to accrue second-order network effects via speculation and trade on secondary marketplaces. However, malicious sellers can burn a significant proportion of the tokens without completely burning the token, resulting in a reduction in trade safety.

## Vulnerability Detail

[`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol#L91) positions control access to significant underlying value. These positions are represented using [ERC-721](https://eips.ethereum.org/EIPS/eip-721)s, allowing them to be traded on secondary marketplaces or via p2p transfers.

However, there is no native restriction in the [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol#L91) that would prevent a malicious user from creating a  valuable position, putting it up for sale on a secondary marketplace (i.e. [LooksRare](https://looksrare.org/)) and maliciously draining the value from the token in the moments before completion of a sale.

It should be noted the entire value of the token cannot be drained as this would [`_burn`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/periphery/PositionManager.sol#L235) the token:

```solidity
 if (position.notionalUp == 0 && position.notionalDown == 0) {
  delete _positions[tokenId];
  _burn(tokenId);
}
```

This means a malicious seller can withdraw all but the entirety of the underlying assets prior to completion of the sale, without invalidating the liveness of the sale.

This yields a reduction in trade safety due to the limited trust that can be placed in [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol#L91) transfers between game-theoretically adverse counterparties.

## Impact

The reliance on trusting the seller limits the trade safety of [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol#L91) NFTs on marketplaces, resulting in loss for unsuspecting users and market stagnation that limits the potential viable accrual of secondary marketplace fees.

## Code Snippet

```solidity
function _sell(
    uint256 tokenId,
    uint256 notionalUp,
    uint256 notionalDown,
    uint256 expectedMarketValue,
    uint256 maxSlippage
) internal returns (uint256 payoff_) {
    ManagedPosition storage position = _positions[tokenId];
    // NOTE: as the positions within the DVP are all of the PositionManager, we must replicate this check here.
    if (notionalUp > position.notionalUp || notionalDown > position.notionalDown) {
        revert CantBurnMoreThanMinted();
    }

    if ((notionalUp > 0 && notionalDown > 0) && (notionalUp != notionalDown)) {
        // If amount is a smile, it must be balanced:
        revert AsymmetricAmount();
    }

    // NOTE: the DVP already checks that the burned notional is lesser or equal to the position notional.
    // NOTE: the payoff is transferred directly from the DVP
    payoff_ = IDVP(position.dvpAddr).burn(
        position.expiry,
        msg.sender,
        position.strike,
        notionalUp,
        notionalDown,
        expectedMarketValue,
        maxSlippage
    );

    // NOTE: premium fix for the leverage issue annotated in the mint flow.
    // notional : position.notional = fix : position.premium
    uint256 premiumFix = ((notionalUp + notionalDown) * position.premium) /
        (position.notionalUp + position.notionalDown);
    position.premium -= premiumFix;
    position.cumulatedPayoff += payoff_;
    position.notionalUp -= notionalUp;
    position.notionalDown -= notionalDown;

    if (position.notionalUp == 0 && position.notionalDown == 0) {
        delete _positions[tokenId];
        _burn(tokenId);
    }

    emit SellDVP(tokenId, (notionalUp + notionalDown), payoff_);
    emit Sell(position.dvpAddr, position.expiry, payoff_);
}
```

## Tool used

Foundry

## Recommendation

There are a couple of possible recommendations:
1. Modifying a token should place a timeout on trade.
2. The [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol#L91) should only allow callers to burn positions in their entirety.


