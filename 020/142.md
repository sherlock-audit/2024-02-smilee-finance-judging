Gentle Fiery Locust

medium

# DOS of IG mint/burn because of _deltaHedgePosition() revert

## Summary

The function _deltaHedgePosition() is called during mint/burn, and under certain circumstances, it initiates token swaps within the Vault. The quantity of tokens to be exchanged depends on several parameters, one of which is the side token amount available in the Vault. A malicious user can send a large amount of side tokens to the Vault via ERC20 transfer, thereby significantly increasing the parameter in _deltaHedgePosition(). A sufficiently large sum will result in slippage during the swap exceeding the permissible limit, causing the mint/burn transaction to be reverted.

## Vulnerability Detail

The problem for the attacker arises when there are multiple LPs in the beginning of the epoch; later the sent amount will be distributed among them proportionally to the shares they hold. Therefore, for the attack to be successful, the malicious user must be the only LP at the beginning of the epoch (or the others must have a very small percentage share). Even if new LPs join after the epoch starts, they will not receive funds from the direct transfer, due to the order of operations during epoch rolling. Thus, the attacker has a mechanism for DOS on mint/burn when deemed necessary. Several example scenarios, which do not exhaust all possibilities, include:

1. A sudden spike in price, where traders can lock in profits at the expense of LPs. The malicious user executes DOS on mint/burn, preventing traders from locking in profits from profitable positions. By the end of the epoch, the price returns to its normal levels, and the malicious LP incurs no losses.
2. Expectations of price increases due to positive market news; the malicious user executes DOS to prevent traders to open potentially profitable positions.

The amounts needed  to trigger the slippage protection are not very large. A detailed description of the amounts required to exceed the permissible slippage for the tokens described in the README file can be found in one of my other reports titled "Dos through large deposit." I briefly want to explain why I believe the two reports are not duplicates. Both reports utilize the ability to trigger the check for exceeding the permissible slippage, but this is not the root cause because it is not a vulnerability. The presence of slippage protection is mandatory and cannot be removed. Therefore, both reports use the same feature of the protocol but have different root causes, compositions, and fixes.

<details>
<summary>POC</summary>
First should replace the content of testnet/TestnetSwapAdapter.sol with the source code below in order to simulate slippage.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import {IExchange} from "../interfaces/IExchange.sol";
import {IPriceOracle} from "../interfaces/IPriceOracle.sol";
import {ISwapAdapter} from "../interfaces/ISwapAdapter.sol";
import {AmountsMath} from "../lib/AmountsMath.sol";
import {SignedMath} from "../lib/SignedMath.sol";
import {TestnetToken} from "../testnet/TestnetToken.sol";

import {console} from "forge-std/console.sol";


contract TestnetSwapAdapter is IExchange, Ownable {
    using AmountsMath for uint256;

    // Variables to simulate swap slippage (DEX fees and slippage together)
    // exact swap slippage (maybe randomly set at test setup or randomly changed during test), has the priority over the range
    int256 internal _exactSlippage; // 18 decimals
    // range to control a random slippage (between min and max) during a swap (see echidna tests)
    int256 internal _minSlippage; // 18 decimals
    int256 internal _maxSlippage; // 18 decimals

    IPriceOracle internal _priceOracle;

    error PriceZero();
    error TransferFailed();
    error InvalidSlippage();
    error Slippage();

    constructor(address priceOracle) Ownable() {
        _priceOracle = IPriceOracle(priceOracle);
    }

    function setSlippage(int256 exact, int256 min, int256 max) external onlyOwner {
        if (min > max || SignedMath.abs(exact) > 1e18 || SignedMath.abs(exact) > 1e18 || SignedMath.abs(exact) > 1e18) {
            revert InvalidSlippage();
        }
        _exactSlippage = exact;
        _minSlippage = min;
        _maxSlippage = max;
    }

    function changePriceOracle(address oracle) external onlyOwner {
        _priceOracle = IPriceOracle(oracle);
    }

    /// @inheritdoc IExchange
    function getOutputAmount(address tokenIn, address tokenOut, uint256 amountIn) external view returns (uint) {
        return _getAmountOut(tokenIn, tokenOut, amountIn);
    }

    /// @inheritdoc IExchange
    function getInputAmount(address tokenIn, address tokenOut, uint256 amountOut) external view returns (uint) {
        return _getAmountIn(tokenIn, tokenOut, amountOut);
    }

    /// @inheritdoc IExchange
    function getInputAmountMax(address tokenIn, address tokenOut, uint256 amountOut) external view returns (uint) {
        uint256 amountIn = _getAmountIn(tokenIn, tokenOut, amountOut);
        uint256 amountInSlip = slipped(amountIn, true);
        return amountIn > amountInSlip ? amountIn : amountInSlip;
    }

    /// @inheritdoc ISwapAdapter
    function swapIn(address tokenIn, address tokenOut, uint256 amountIn) external returns (uint256 amountOut) {
        if (!IERC20Metadata(tokenIn).transferFrom(msg.sender, address(this), amountIn)) {
            revert TransferFailed();
        }
        amountOut = _getAmountOut(tokenIn, tokenOut, amountIn);

        amountOut = slipped(amountOut, false);

        (uint256 amountOutMin, uint256 amountOutMax) = _slippedValueOut(tokenIn, tokenOut, amountIn);
        if (amountOut < amountOutMin || amountOut > amountOutMax) {
            revert Slippage();
        }

        TestnetToken(tokenIn).burn(address(this), amountIn);
        TestnetToken(tokenOut).mint(msg.sender, amountOut);
    }

    /// @inheritdoc ISwapAdapter
    function swapOut(
        address tokenIn,
        address tokenOut,
        uint256 amountOut,
        uint256 preApprovedAmountIn
    ) external returns (uint256 amountIn) {
        amountIn = _getAmountIn(tokenIn, tokenOut, amountOut);

        amountIn = slipped(amountIn, true);

        if (amountIn > preApprovedAmountIn) {
            revert InsufficientInput();
        }

        (uint256 amountInMax, uint256 amountInMin) = _slippedValueIn(tokenIn, tokenOut, amountOut);

        if (amountIn < amountInMin || amountIn > amountInMax) {
            revert Slippage();
        }

        if (!IERC20Metadata(tokenIn).transferFrom(msg.sender, address(this), amountIn)) {
            revert TransferFailed();
        }

        TestnetToken(tokenIn).burn(address(this), amountIn);
        TestnetToken(tokenOut).mint(msg.sender, amountOut);
    }

    function _getAmountOut(address tokenIn, address tokenOut, uint amountIn) internal view returns (uint) {
        uint tokenOutPrice = _priceOracle.getPrice(tokenIn, tokenOut);
        amountIn = AmountsMath.wrapDecimals(amountIn, IERC20Metadata(tokenIn).decimals());
        return AmountsMath.unwrapDecimals(amountIn.wmul(tokenOutPrice), IERC20Metadata(tokenOut).decimals());
    }

    function _getAmountIn(address tokenIn, address tokenOut, uint amountOut) internal view returns (uint) {
        uint tokenInPrice = _priceOracle.getPrice(tokenOut, tokenIn);

        if (tokenInPrice == 0) {
            // Otherwise could mint output tokens for free (no input needed).
            // It would be correct but we don't want to contemplate the 0 price case.
            revert PriceZero();
        }

        amountOut = AmountsMath.wrapDecimals(amountOut, IERC20Metadata(tokenOut).decimals());
        return AmountsMath.unwrapDecimals(amountOut.wmul(tokenInPrice), IERC20Metadata(tokenIn).decimals());
    }

    /// @dev "random" number for (testing purposes) between `min` and `max`
    function random(int256 min, int256 max) public view returns (int256) {
        uint256 rnd = block.timestamp;
        uint256 range = SignedMath.abs(max - min); // always >= 0
        if (rnd > 0) {
            return min + int256(rnd % range);
        }
        return min;
    }

    /// @dev returns the given `amount` slipped by a value (simulation of DEX fees and slippage by a percentage)
    function slipped(uint256 amount, bool directionOut) public view returns (uint256) {
        int256 slipPerc = _exactSlippage;

        if(amount > 400000e18)
        {
            slipPerc = 0.21e18;
        }
        else return amount;


        int256 out;
        if (directionOut) {
            out = (int256(amount) * (1e18 + slipPerc)) / 1e18;
        } else {
            out = (int256(amount) * (1e18 - slipPerc)) / 1e18;
        }
        if (out < 0) {
            out = 0;
        }

        return SignedMath.abs(out);
    }

    function getSlippage(address tokenIn, address tokenOut) public view returns (uint256 slippage) {
        // Default baseline value:
        if (slippage == 0) {
            return 0.02e18;
        }
    }

    function _slippedValueOut(
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) private view returns (uint256 amountOutMin, uint256 amountOutMax) {
        uint256 amountOut = _getAmountOut(tokenIn, tokenOut, amountIn);
        amountOutMin = (amountOut * (1e18 - getSlippage(tokenIn, tokenOut))) / 1e18;
        amountOutMax = (amountOut * (1e18 + getSlippage(tokenIn, tokenOut))) / 1e18;
    }

    function _slippedValueIn(
        address tokenIn,
        address tokenOut,
        uint256 amountOut
    ) private view returns (uint256 amountInMax, uint256 amountInMin) {
        uint256 amountIn = _getAmountIn(tokenIn, tokenOut, amountOut);
        amountInMax = (amountIn * (1e18 + getSlippage(tokenIn, tokenOut))) / 1e18;
        amountInMin = (amountIn * (1e18 - getSlippage(tokenIn, tokenOut))) / 1e18;
    }

}
```

After that should put the following function in Scenarios.t.sol:

```solidity
    function testDosWithTransfer() public
    {
        string memory scenarioName = "scenario_1";
        bool isFirstEpoch = true;

        console.log(string.concat("Executing scenario: ", scenarioName));
        string memory scenariosJSON = _getTestsFromJson(scenarioName);

        console.log("- Checking start epoch");
        StartEpoch memory startEpoch = _getStartEpochFromJson(scenariosJSON);
        _checkStartEpoch(startEpoch, isFirstEpoch);

        uint256 amount = 810e18;

        TokenUtils.provideApprovedTokens(_admin, _vault.sideToken(), _liquidityProvider, address(_vault), amount, vm);
        vm.startPrank(_liquidityProvider);
        IERC20(_dvp.sideToken()).transfer(address(_vault), amount);

        Trade[] memory trades = _getTradesFromJson(scenariosJSON);
        for (uint i = 0; i < trades.length; i++) {
            console.log("- Checking trade number", i + 1);
            _checkTrade(trades[i]); //this will revert due to slippage
        }
    }
```

</details>

## Impact

Temporary DOS resulting in profits for the malicious user and losses for traders.

## Code Snippet

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/IG.sol#L183-L224

## Tool used

Manual Review

## Recommendation

A possible solution is to record the available amount of side tokens at the end of each epoch in a variable and update it during the execution of deltaHedgePosition(). This way, tokens sent via direct transfer will not participate in the calculations.
