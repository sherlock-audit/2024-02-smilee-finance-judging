Massive Alabaster Dalmatian

high

# Non-functional mint in PositionManager if nftAccessFlag is true.

## Summary
Traders cannot mint position through PositionManager::mint() if nftAccessFlag is true.

## Vulnerability Detail
In PositionManger contract, we can use mint() function to mint one position.
```solidity
    function mint(
        IPositionManager.MintParams calldata params
    ) external override returns (uint256 tokenId, uint256 premium) {
        ...
        // Premium already include fee
        baseToken.safeApprove(params.dvpAddr, obtainedPremium);
        premium = dvp.mint(
            address(this),
            params.strike,
            params.notionalUp,
            params.notionalDown,
            params.expectedPremium,
            params.maxSlippage,
            params.nftAccessTokenId
        );
        ...
    }
```
In DVP::mint(), there is one nft_access check. If we want to mint one position in dvp when nftAccessFlag is true, **recipient** must be the owner of **nftAccessTokenId**. However, when trader mint position through PositionManager::mint(), **recipient** is PositionManger, not the trader. So even if traders own nftAccessToken, positionManger::mint() will be blocked.

eg.
- **nftAccessFlag** is true in dvp.
- Alice owns one nftAccessToken, token id is 1.
- Alice call PositionManger::mint by nftAccessToken(1).
- DVP::mint() will be revert because dvp::mint() think **recipient**(PositionManger) does not own this nftAccessToken(1)

```solidity
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
    function _checkNFTAccess(uint256 accessTokenId, address receiver, uint256 notionalAmount) internal {
        if (nftAccessFlag) {
            IDVPAccessNFT nft = IDVPAccessNFT(_addressProvider.dvpAccessNFT());
            if (accessTokenId == 0 || nft.ownerOf(accessTokenId) != receiver) {
                revert NFTAccessDenied();
            }
            nft.checkCap(accessTokenId, notionalAmount);
        }
    }
```
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/IG.sol#L61-L76
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/IG.sol#L307-L316
## Tool used

Manual Review

## Recommendation
DVP::mint need to know the actual trader to verify the nft_access_token.