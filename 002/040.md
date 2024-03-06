Rural Blonde Cougar

medium

# PositionManager will revert when trying to return back to user excess of the premium transferred from the user when minting position

## Summary

`PositionManager.mint` calculates preliminary premium to be paid for buying the option and transfers it from the user. The actual premium paid may differ, and if it's smaller, excess is returned back to user. However, it is returned using the `safeTransferFrom`:
```solidity
    if (obtainedPremium > premium) {
        baseToken.safeTransferFrom(address(this), msg.sender, obtainedPremium - premium);
    }
```

The problem is that `PositionManager` doesn't approve itself to transfer baseToken to `msg.sender`, and USDC `transferFrom` implementation requires approval even if address is transferring from its own address. Thus the transfer will revert and user will be unable to open position.

## Vulnerability Detail

Both `transferFrom` implementations in USDC on Arbitrum (USDC and USDC.e) require approval from any address, including when doing transfers from your own address.
https://arbiscan.io/address/0x1efb3f88bc88f03fd1804a5c53b7141bbef5ded8#code
```solidity
    function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
        _transfer(sender, recipient, amount);
        _approve(sender, _msgSender(), _allowances[sender][_msgSender()].sub(amount, "ERC20: transfer amount exceeds allowance"));
        return true;
    }
```

https://arbiscan.io/address/0x86e721b43d4ecfa71119dd38c0f938a75fdb57b3#code
```solidity
    function transferFrom(
        address from,
        address to,
        uint256 value
    )
        external
        override
        whenNotPaused
        notBlacklisted(msg.sender)
        notBlacklisted(from)
        notBlacklisted(to)
        returns (bool)
    {
        require(
            value <= allowed[from][msg.sender],
            "ERC20: transfer amount exceeds allowance"
        );
        _transfer(from, to, value);
        allowed[from][msg.sender] = allowed[from][msg.sender].sub(value);
        return true;
    }
```

`PositionManager` doesn't approve itself to do transfers anywhere, so `baseToken.safeTransferFrom(address(this), msg.sender, obtainedPremium - premium);` will always revert, preventing the user from opening position via `PositionManager`, breaking important protocol function.

## Impact

User is unable to open positions via `PositionManager` in certain situations as all such transactions will revert, breaking important protocol functionality and potentially losing user funds / profit due to failure to open position.

## Code Snippet

`PositionManager.mint` transfers base token back to `msg.sender` via `safeTransferFrom`:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/periphery/PositionManager.sol#L139-L141

## Tool used

Manual Review

## Recommendation

Consider using `safeTransfer` instead of `safeTransferFrom` when transferring token from self.