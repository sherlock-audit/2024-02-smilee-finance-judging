Howling Blonde Barracuda

high

# Vault can become stuck in certain scenarios where the remaining side token is only a few amount

## Summary

In certain scenarios where the Vault has to trigger `_sellSideTokens` when `rollEpoch` or `deltaHedge` is called, operations could revert due to the swap resulting in a 0 amount of the base token.

## Vulnerability Detail

There are multiple main scenarios where `_sellSideTokens` is triggered:

1. When `rollEpoch` is triggered, and the vault is killed, requiring to sell all side tokens to accommodate withdrawals.
2. When `rollEpoch` is triggered, and the vault is active, `_adjustBalances` is called. If the current base token amount cannot accommodate pending withdrawals and pending payoffs, it sells side tokens to fulfill the pending requests and rebalance the base token amount.
3. When `deltaHedge` is triggered from the corresponding DVP, when traders mint or burn from DVP.

inside `_sellSideTokens`, if amount is non 0 and not exceeding `sideTokens` balance, it will process the swap by calling registered exchange adapter : 

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/Vault.sol#L797-L814

```solidity
    function _sellSideTokens(uint256 amount) internal returns (uint256 baseTokens) {
        if (amount == 0) {
            return 0;
        }
        (, uint256 sideTokens) = _tokenBalances();
        if (amount > sideTokens) {
            revert InsufficientLiquidity(bytes4(keccak256("_sellSideTokens()")));
        }

        address exchangeAddress = _addressProvider.exchangeAdapter();
        if (exchangeAddress == address(0)) {
            revert AddressZero();
        }
        IExchange exchange = IExchange(exchangeAddress);

        IERC20(sideToken).safeApprove(exchangeAddress, amount);
==>     baseTokens = exchange.swapIn(sideToken, baseToken, amount);
    }
```

Inside Exchange's router `swapIn`, if  swapped `amountOut` is 0, the call will revert : 

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/providers/SwapAdapterRouter.sol#L152-L173

```solidity
    function swapIn(address tokenIn, address tokenOut, uint256 amountIn) external returns (uint256 amountOut) {
        _zeroAddressCheck(tokenIn);
        _zeroAddressCheck(tokenOut);
        address adapter = _adapters[_encodePath(tokenIn, tokenOut)].get();
        _zeroAddressCheck(adapter);

        (uint256 amountOutMin, uint256 amountOutMax) = _slippedValueOut(tokenIn, tokenOut, amountIn);

        IERC20Metadata(tokenIn).safeTransferFrom(msg.sender, address(this), amountIn);
        IERC20Metadata(tokenIn).safeApprove(adapter, amountIn);
        amountOut = ISwapAdapter(adapter).swapIn(tokenIn, tokenOut, amountIn);

==>     if (amountOut == 0) {
            revert SwapZero();
        }

        if (amountOut < amountOutMin || amountOut > amountOutMax) {
            revert Slippage();
        }

        IERC20Metadata(tokenOut).safeTransfer(msg.sender, amountOut);
    }
```

## PoC 

Add the following test to `UniswapAdapterTest` test contract : 

```solidity
    function testSwapInWethUsdcDust() public {
        _swapInTest(_WETH, _USDC, _WETH_HOLDER, 1e8);
    }
```

Modify swap in test to log the expected swap amount and result : 

```solidity
    function _swapInTest(address tokenInAddr, address tokenOutAddr, address holder, uint256 amountIn) private {
        IERC20 tokenIn = IERC20(tokenInAddr);
        IERC20 tokenOut = IERC20(tokenOutAddr);

        uint256 tokenInBalanceBefore = tokenIn.balanceOf(holder);
        uint256 tokenOutBalanceBefore = tokenOut.balanceOf(holder);

        uint256 expectedAmountOut = _quoteInput(tokenInAddr, tokenOutAddr, amountIn);
        // uint256 expectedAmountOut2 = _uniswap.getOutputAmount(tokenInAddr, tokenOutAddr, amountIn);
        console.log("expected amount out : ");
        console.log(expectedAmountOut);

        vm.startPrank(holder);
        tokenIn.approve(address(_uniswap), tokenIn.balanceOf(holder));
        _uniswap.swapIn(tokenInAddr, tokenOutAddr, amountIn);
        vm.stopPrank();

        uint256 tokenInBalanceAfter = tokenIn.balanceOf(holder);
        uint256 tokenOutBalanceAfter = tokenOut.balanceOf(holder);
        console.log("token out balance before : ");
        console.log(tokenOutBalanceBefore);
        console.log("token out balance after : ");
        console.log(tokenOutBalanceAfter);
        assertEq(tokenInBalanceAfter, tokenInBalanceBefore - amountIn);
        assertEq(tokenOutBalanceAfter, tokenOutBalanceBefore + expectedAmountOut);
    }
```

Run the test : 

```sh
forge test --match-contract UniswapAdapterTest --match-test testSwapInWethUsdcDust -vvv
```

Log output : 

```shell
Logs:
  expected amount out : 
  0
  token out balance before : 
  717262109074
  token out balance after : 
  717262109074
```

## Impact

The swap resulting in a 0 amount and reverting the calls is not uncommon, considering these scenarios : 

1. When the vault is killed, and the previous epoch has already swapped almost all side tokens to base tokens due to high withdrawal demand, leaving a small amount of side tokens. Admins can fix this by donating and triggering `emergencyRebalance`, but griefers can keep donating a dust amount (non-zero) of side tokens to keep reverting the `rollEpoch`.
2. When the price is relatively stable between each epoch, and user's deposit/withdrawal requests remain the same, resulting in a rebalance that only requires a small amount of side tokens to be swapped to base tokens.
3. Delta hedge triggered from DVP resulting in a small amount to be swapped.

Besides that, side tokens (WBTC, WETH) also usually have much higher value compared to stable coin (USDC), so a small amount of side tokens often results in a 0 swap result.

## Code Snippet

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/Vault.sol#L797-L814
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/providers/SwapAdapterRouter.sol#L164-L166

## Tool used

Manual Review

## Recommendation

Consider adding an additional check, if the amount that needs to be swapped is small, early return the calls instead.
