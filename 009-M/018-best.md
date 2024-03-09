Abundant Cornflower Seagull

medium

# Precision loss in `SwapAdapterRouter::_valueIn` due to tokens with different decimals

## Summary
The `SwapAdapterRouter::_valueIn` function has different logic depending if `tokenIn` has more decimals than `tokenOut`, but it still returns incorrect values.

## Vulnerability Detail
To begin with, let me introduce where and why this function get's called. When we want to start the new epoch, the `EPOCH_ROLLER` calls [`EpochControls::rollEpoch`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/EpochControls.sol#L24), it on [L27](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/EpochControls.sol#L27) calls [`Vault::_beforeRollEpoch`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/Vault.sol#L588). At some point on [L667](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/Vault.sol#L667), this function internally calls [Vault::_adjustBalances`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/Vault.sol#L685). Why? Cause we might need to swap side tokens into base tokens and vice versa.

In the scenario when we need to swap side tokens into base tokens cause we don't have enough of them to cover all the liquidity, then we on [L698](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/Vault.sol#L698) we call [`SwapAdapterRouter:;getInputAmount`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/providers/SwapAdapterRouter.sol#L130). It [returns](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/providers/SwapAdapterRouter.sol#L137) the value we get from [`SwapAdapterRouter::_valueIn`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/providers/SwapAdapterRouter.sol#L251). Let's look at it a bit closer.

<details>
<summary>`SwapAdapterRouter::_valueIn`</summary>

```javascript
    function _valueIn(address tokenIn, address tokenOut, uint256 amountOut) private view returns (uint256 amountIn) {
        IPriceOracle po = IPriceOracle(_ap.priceOracle());
        uint256 price = po.getPrice(tokenOut, tokenIn);
        uint8 dIn = IERC20Metadata(tokenIn).decimals();
        uint8 dOut = IERC20Metadata(tokenOut).decimals();
        amountIn = price * amountOut;
        amountIn = dIn > dOut ? amountIn * 10 ** (dIn - dOut) : amountIn / 10 ** (dOut - dIn);
        amountIn = amountIn / 10 ** 18;
    }
```

</details>

The purpose of this function is we input how much `tokenIn` we need to sell to get `amountOut` of `tokenOut`. Since we call it if we need to sell side tokens to base tokens, then `tokenIn == side token` and `tokenOut == base token`.
First we get the price oracle from address provider, then we get the price of tokenOut in tokenIn by calling [`ChainlinkPriceOracle::getPrice`](https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/providers/chainlink/ChainlinkPriceOracle.sol#L128). Let's take a closer look at it as well.

<details>
<summary>`ChainlinkPriceOracle::getPrice`</summary>

```javascript
    function getPrice(address token0, address token1) external view returns (uint256) {
        // NOTE: both prices are expected to be in the same reference currency (e.g. USD)
        uint256 token0Price = getTokenPrice(token0);
        uint256 token1Price = getTokenPrice(token1);

        if (token1Price == 0) {
            revert PriceZero();
        }

        return token0Price.wdiv(token1Price);
    }
```

</details>

It just takes prices from price feeds and then does `token0.wdiv(token1)`. In our case `token0` is tokenOut (base token) and `token1` is tokenIn (side token). The value from `wdiv` function is 1e18, therefore, if WETH price is $1000, then 1 USDC = 0.001 WETH = 1e15.

Back to `_valueIn`. Then we get the `amountIn = price * amountOut` and start to adjust it according to decimals of tokenIn and tokenOut.

Now, I've got 4 of examples for you with a POC:

*First Example:* Base token is USDC (6 decimals), side token is WETH (18 decimals) and we need to get 10K USDC. The output we've got: 10000000000000 (1e13).
*Second Example:* Base token is DAI (18 decimals), side token is WETH (18 decimals) and we need to get 10K DAI (I know that in READ.ME starting base token is USDC, but it says in future it will be any stable coin, and you can replace DAI with any stablecoin with 18 decimals). The output: 10.
*Third Example:* Base token is USDC (6 decimals), side token is WBTC (8 decimals) and we need to get 10K USDC. The output we've got: 1.
*Fourth Example:* Base token is DAI (6 decimals), side token is WBTC (8 decimals) and we need to get 10K DAI. The output we've got: 0.

<details>
<summary>POC</summary>
Just insert it a new file in test folder:

```javascript
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.15;

import {Test, console} from "forge-std/Test.sol";
import {AmountsMath} from "../src/lib/AmountsMath.sol";

contract AuditPrecisionTest is Test {
    // Used by Chanilink
    using AmountsMath for uint256;

    uint256 public WETHPrice = 1000e8; //Chainlink WETH price = 1000.00000000 USD
    uint256 public USDCPrice = 1e8; // Chainlink USDC price = 1.00000000 USD
    uint256 public WBTCPrice = 10000e8; // Chainlink WBTC price = 10,000.00000000 USD
    uint256 public DAIPrice = 1e8; // Chainlink DAI price = 1.00000000 USD

    uint8 public USDCDecimals = 6;
    uint8 public WETHDecimals = 18;
    uint8 public DAIDecimals = 18;
    uint8 public WBTCDecimals = 6;

    uint256 public amountOutUSDC10K = 10000; // We need 10k USDC
    uint256 public amountOutUSDC1K = 1000; // We need 10k USDC
    uint256 public amoutnOutUSDC1 = 1; // We need only 1 USDC

    uint256 public amountOutDAI10K = 10000; // We need 10k DAI
    uint256 public amountOutDAI1K = 1000; // We need 10k DAI
    uint256 public amoutnOutDAI1 = 1; // We need only 1 DAI

    function setUp() public {}

    /////////////// ETH PRICES ///////////////

    function testUSDCAmountInForWETH() public view {
        // Get the price of USDC in WETH (price of tokenOut in tokenIn).
        // This is what's returned from ChanlinkPriceOracle::getPrice
        uint256 usdcToWethPrice = USDCPrice.wdiv(WETHPrice);
        console.log("1 USDC = ",usdcToWethPrice, "WETH");
        // 1,000,000,000,000,000 -> 1e15

        // Get how much WETH we need to pay to get 10K USDC (amountOut)
        uint256 amountInWETH = usdcToWethPrice * amountOutUSDC10K;
        console.log("To get 10K USDC, we need to pay", amountInWETH, "WETH");
        // 10,000,000,000,000,000,000 -> 1e19 -> 10e18 -> 10 WETH
        

        // Continue calculating amountIn, since WETH (tokenIn) has 18 decimals and USDC (tokenOut) has only 6,
        // Then we use formula amountIn * 10 ** (dIn - dOut)
        amountInWETH = amountInWETH * 10 ** (WETHDecimals - USDCDecimals);
        console.log("amountInWETH for USDC =", amountInWETH);
        // 10,000,000,000,000,000,000,000,000,000,000 -> 1e31

        amountInWETH = amountInWETH / 10 ** 18;
        console.log("final amountIn for USDC =", amountInWETH);
        // 10,000,000,000,000 -> 1e13
    }

    function testDAIAmountInForWETH() public view {
        // Get the price of DAI in WETH (price of tokenOut in tokenIn).
        // This is what's returned from ChanlinkPriceOracle::getPrice
        uint256 daiToWethPrice = DAIPrice.wdiv(WETHPrice);
        console.log("1 DAI = ",daiToWethPrice, "WETH");
        // 1,000,000,000,000,000 -> 1e15

        // Get how much WETH we need to pay to get 10K DAI (amountOut)
        uint256 amountInWETH = daiToWethPrice * amountOutDAI10K;
        console.log("To get 10K DAI, we need to pay", amountInWETH, "WETH");
        // 10,000,000,000,000,000,000 -> 1e19 -> 10e18 -> 10 WETH
        
        // Continue calculating amountIn, since WETH (tokenIn) has 18 decimals and DAI (tokenOut) has 18,
        // Then dIn > dOut is not true (decimals for tokenIn and tokenOut respectively), then we use formula amountIn / 10 ** (dOut - dIn)
        amountInWETH = amountInWETH / 10 ** (DAIDecimals - WETHDecimals);
        console.log("amountInWETH for DAI =", amountInWETH);
        // 10,000,000,000,000,000,000 -> 1e19 -> 10e18 -> 10 WETH
        
        amountInWETH = amountInWETH / 10 ** 18;
        console.log("final amountIn for DAI =", amountInWETH);
        // 10 WETH
    }

    /////////////// WBTC PRICES ///////////////

    function testUSDCAmountInForWBTC() public view {
        // Get the price of USDC in WBTC (price of tokenOut in tokenIn).
        // This is what's returned from ChanlinkPriceOracle::getPrice
        uint256 usdcToWBTCPrice = USDCPrice.wdiv(WBTCPrice);
        console.log("1 USDC = ",usdcToWBTCPrice, "WBTC");
        // 100,000,000,000,000 -> 1e14 (0.0001 WBTC = 1 USDC)

        // Get how much WBTC we need to pay to get 10K USDC (amountOut)
        uint256 amountInWBTC = usdcToWBTCPrice * amountOutUSDC10K;
        console.log("To get 10K USDC, we need to pay", amountInWBTC, "WBTC");
        // 1,000,000,000,000,000,000 -> 1e18
        

        // Continue calculating amountIn, since WBTC (tokenIn) has 8 decimals and USDC (tokenOut) has only 6,
        // Then we use formula amountIn * 10 ** (dIn - dOut)
        amountInWBTC = amountInWBTC * 10 ** (WBTCDecimals - USDCDecimals);
        console.log("amountInWBTC for USDC =", amountInWBTC);
        // 1,000,000,000,000,000,000 -> 1e18

        amountInWBTC = amountInWBTC / 10 ** 18;
        console.log("final amountIn for USDC =", amountInWBTC);
        // 1 WBTC
    }

    function testDAIAmountInForWBTC() public view {
        // Get the price of DAI in WBTC (price of tokenOut in tokenIn).
        // This is what's returned from ChanlinkPriceOracle::getPrice
        uint256 daiToWBTCPrice = DAIPrice.wdiv(WBTCPrice);
        console.log("1 DAI = ",daiToWBTCPrice, "WBTC");
        // 100,000,000,000,000 -> 1e14 (0.0001 WBTC for 1 DAI)

        // Get how much WBTC we need to pay to get 10K DAI (amountOut)
        uint256 amountInWBTC = daiToWBTCPrice * amountOutDAI10K;
        console.log("To get 10K DAI, we need to pay", amountInWBTC, "WBTC");
        // 1,000,000,000,000,000,000 -> 1e18
        
        // Continue calculating amountIn, since WBTC (tokenIn) has 8 decimals and DAI (tokenOut) has 18,
        // Then dIn > dOut is not true (decimals for tokenIn and tokenOut respectively), then we use formula amountIn / 10 ** (dOut - dIn)
        amountInWBTC = amountInWBTC / 10 ** (DAIDecimals - WBTCDecimals);
        console.log("amountInWBTC for DAI =", amountInWBTC);
        // 1,000,000

        amountInWBTC = amountInWBTC / 10 ** 18;
        console.log("final amountIn for DAI =", amountInWBTC);
        // 0
    }
```

</details>

## Impact
Even if the example with DAI are not correct, cause it might be never accepted as a stable coin, I want to put out that for example to get 10K USDC we need to transfer 1e13 WETH or 1 WBTC, which is incorrect, cause, for example, for 10K USDC it still gives out 1 WBTC and 1.6 * 10 ** 12, when WETH has 18 decimals.

It has high probability of happening, even though we actually sell 1.5x of side tokens than we need, it can easily lead to reverts and no ability to roll epochm until users complete their withdrawals and DVPs complete their payoffs, but of course issue needs certain conditions, therefore, a medium.

## Code Snippet
Link to every function mentioned above in inside the text, therefore, you can easily click it if you want ;)

## Tool used

Manual Review, Foundry

## Recommendation
Looks like there's a very little need in those adjustments and when we calculate `amountIn = price * amountOut`, we get what we need. For example, in the instance of USDC and WETH, we've got 1e19 -> 10e18 -> 10 WETH = 10K USDC at price of $1000 (used in the POC). Therefore, there should be a mititgation for tokens tokenIn which don't have 18 decimals:

```diff
        function _valueIn(address tokenIn, address tokenOut, uint256 amountOut) private view returns (uint256 amountIn) {
        IPriceOracle po = IPriceOracle(_ap.priceOracle());
        uint256 price = po.getPrice(tokenOut, tokenIn);
        uint8 dIn = IERC20Metadata(tokenIn).decimals();
        uint8 dOut = IERC20Metadata(tokenOut).decimals();
        amountIn = price * amountOut;
+       if (dIn < 18) {
+           uint8 excessDecimals = 18 - dIn;
+           amountIn = amountIn / 10 ** excessDecimals; 
+       } else if (dIn > 18) {
+           uint8 lackDecimals = dIn - 18;
+           amountIn = amountIn * 10 ** lackDecimals;
+       }
-       amountIn = dIn > dOut ? amountIn * 10 ** (dIn - dOut) : amountIn / 10 ** (dOut - dIn);
-       amountIn = amountIn / 10 ** 18;
    }
```
