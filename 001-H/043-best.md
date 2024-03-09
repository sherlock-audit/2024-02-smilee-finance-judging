Rural Blonde Cougar

high

# FeeManager `receiveFee` and `trackVaultFee` functions allow anyone to call it with user-provided dvp/vault address and add any arbitrary feeAmount to any address, breaking fees accounting and temporarily bricking DVP smart contract

## Summary

`FeeManager` uses `trackVaultFee` function to account vault fees. The problem is that this function can be called by any smart contract implementing `vault()` function (there are no address or role authentication), thus malicious user can break all vault fees accounting by randomly inflating existing vault's fees, making it hard/impossible for admins to determine the real split of fees between vaults. Moreover, malicious user can provide such `feeAmount` to `trackVaultFee` function, which will increase any vault's fee to `uint256.max` value, meaning all following calls to `trackVaultFee` will revert due to fee addition overflow, temporarily bricking DVP smart contract, which calls `trackVaultFee` on all mints and burns, which will always revert until `FeeManager` smart contract is updated to a new address in `AddressProvider`.

Similarly, `receiveFee` function is used to account fee amounts received by different addresses (dvp), which can later be withdrawn by admin via `withdrawFee` function. The problem is that any smart contract implementing `baseToken()` function can call it, thus any malicious user can break all accounting by adding arbitrary amounts to their addresses without actually paying anything. Once some addresses fees are inflated, it will be difficult for admins to track fee amounts which are real, and which are from fake `dvp`s and fake tokens.

## Vulnerability Detail

`FeeManager.trackVaultFee` function has no role/address check:
```solidity
    function trackVaultFee(address vault, uint256 feeAmount) external {
        // Check sender:
        IDVP dvp = IDVP(msg.sender);
        if (vault != dvp.vault()) {
            revert WrongVault();
        }

        vaultFeeAmounts[vault] += feeAmount;

        emit TransferVaultFee(vault, feeAmount);
    }
```

Any smart contract implementing `vault()` function can call it. The vault address returned can be any address, thus user can inflate vault fees both for existing real vaults, and for any addresses user chooses. This totally breaks all vault fees accounting.

`FeeManager.receiveFee` function has no role/address check either:
```solidity
    function receiveFee(uint256 feeAmount) external {
        _getBaseTokenInfo(msg.sender).safeTransferFrom(msg.sender, address(this), feeAmount);
        senders[msg.sender] += feeAmount;

        emit ReceiveFee(msg.sender, feeAmount);
    }
...
    function _getBaseTokenInfo(address sender) internal view returns (IERC20Metadata token) {
        token = IERC20Metadata(IVaultParams(sender).baseToken());
    }
```

Any smart contract crafted by malicious user can call it. It just has to return base token, which can also be token created by the user. After transfering this fake base token, the `receiveFee` function will increase user's fee balance as if it was real token transferred.

## Impact

Malicious users can break all fee and vault fee accounting by inflating existing vaults or user addresses fees earned without actually paying these fees, making it hard/impossible for admins to determine the actual fees earned from each vault or dvp. Moreover, malicious user can temporarily brick DVP smart contract by inflating vault's accounted fees to `uint256.max`, thus making all DVP mints and burns (which call `trackVaultFee`) revert.

## Code Snippet

`FeeManager.trackVaultFee`:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/FeeManager.sol#L218-L228

`FeeManager.receiveFee`:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/FeeManager.sol#L210-L215

## Tool used

Manual Review

## Recommendation

Consider adding a whitelist of addresses which can call these functions.