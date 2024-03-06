Bent Crimson Goat

high

# The `PositionManager` is unable to sell off `IG` positions when minted using a different `strike` than `financeParameters.currentStrike`.

## Summary

The [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol) stores [`ManagedPosition`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/periphery/PositionManager.sol#L17)s against the user-defined `strike` price, instead of their realised strike price as defined by the [`IG`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/IG.sol).

This introduces miscommunication between the two modules, resulting in the failure to allow users to properly close out their positions.

## Vulnerability Detail

When minting a position via the [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol), the caller may request a `strike` price via [`IPositionManager.MintParams`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/interfaces/IPositionManager.sol#L18).

Once a `PositionManager` NFT has been successfully minted, the caller-defined `strike` provided in [`IPositionManager.MintParams`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/interfaces/IPositionManager.sol#L18) is recorded against the newly instantiated [`ManagedPosition`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/periphery/PositionManager.sol#L17):

```solidity
if (params.tokenId == 0) {
  // Mint token:
  tokenId = _nextId++;
  _mint(params.recipient, tokenId);

  Epoch memory epoch = dvp.getEpoch();

  // Save position:
  _positions[tokenId] = ManagedPosition({
    dvpAddr: params.dvpAddr,
@>  strike: params.strike,
    expiry: epoch.current,
    premium: premium,
    leverage: (params.notionalUp + params.notionalDown) / premium,
    notionalUp: params.notionalUp,
    notionalDown: params.notionalDown,
    cumulatedPayoff: 0
  });
}
```

However, this is not the `strike` price the position is guaranteed to be minted at.

Looking at [`IG`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/IG.sol), a concrete implementation of the [`DVP`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol), the caller's requested `strike` is explicitly ignored, instead favouring to apply `financeParameters.currentStrike` instead:

```solidity
/**
 * @inheritdoc IDVP
 * @audit The strike parameter is unused, but that affects how the NFT is minted in the PositionManager.
 */
function mint(
    address recipient,
    uint256 strike,
    uint256 amountUp,
    uint256 amountDown,
    uint256 expectedPremium,
    uint256 maxSlippage,
    uint256 nftAccessTokenId
) external override returns (uint256 premium_) {
@>  strike;
    _checkNFTAccess(nftAccessTokenId, recipient, amountUp + amountDown);
    Amount memory amount_ = Amount({up: amountUp, down: amountDown});

@>  premium_ = _mint(recipient, financeParameters.currentStrike, amount_, expectedPremium, maxSlippage);
}
```

Continuing on, let's assume that the caller to [`mint(IPositionManager.MintParams)`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/periphery/PositionManager.sol#L91) provided a value of `strike` which was different to `financeParameters.currentStrike`.

When it comes to selling the position via the [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol), the `tokenId` of the position to be burned is used to look up the equivalent strike price:

```solidity
payoff_ = IDVP(position.dvpAddr).burn(
    position.expiry,
    msg.sender,
@>  position.strike,
    notionalUp,
    notionalDown,
    expectedMarketValue,
    maxSlippage
);
```

Which in turn is passed into the [`DVP`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol), which expects to discover a position made at a **different** strike price:

```solidity
function _burn(
    uint256 expiry,
    address recipient,
    uint256 strike,
    Amount memory amount,
    uint256 expectedMarketValue,
    uint256 maxSlippage
) internal returns (uint256 paidPayoff) {
    _requireNotPaused();
@>  Position.Info storage position = _getPosition(expiry, msg.sender, strike); // @audit The position's strike is different than realized.
```

This results in the position controlled by the [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol) to fail to be sold off, as it is incorrectly determined to not exist.

Subsequently, the user who controls the underlying position (only by proxy via the [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol)) being unable to close out their position.

## Impact

Stuck funds (high).

## Code Snippet

```solidity
/**
 * @inheritdoc IDVP
 * @audit The strike parameter is unused, but that affects how the NFT is minted in the PositionManager.
 */
function mint(
    address recipient,
    uint256 strike,
    uint256 amountUp,
    uint256 amountDown,
    uint256 expectedPremium,
    uint256 maxSlippage,
    uint256 nftAccessTokenId
) external override returns (uint256 premium_) {
    strike;
    _checkNFTAccess(nftAccessTokenId, recipient, amountUp + amountDown);
    Amount memory amount_ = Amount({up: amountUp, down: amountDown});

    premium_ = _mint(recipient, financeParameters.currentStrike, amount_, expectedPremium, maxSlippage);
}
```

## Tool used

Foundry

## Recommendation

There are two possible recommendations:

1. Force a `revert` when minting a new position via the [`PositionManager`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol) if the provided `strike` is not equal to the signalling [`currentStrike()`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/IG.sol#L57).
2. Alternatively, store the resolved `strike` price at which the [`DVP`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol) decides to mint the position at in the [`ManagedPosition`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/periphery/PositionManager.sol#L17), opposed to the requested strike.
