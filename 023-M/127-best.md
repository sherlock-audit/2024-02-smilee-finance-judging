Gentle Fiery Locust

high

# Dos through large deposit

## Summary

To maintain the balance between base tokens and side tokens, the Vault performs token swaps on Uniswap when rolling over epochs and during delta hedging (when minting/burning IG positions). The SwapAdapterRouter has a default slippage of 2% calculated on the price obtained from Chainlink. The problem is that when a large amount of tokens needs to be exchanged, the allowed slippage may be exceeded, leading to the impossibility of rolling over epochs and consequently causing a denial of service (DOS) on the protocol.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/providers/SwapAdapterRouter.sol#L152-L205

The implementation of the slippage check can be seen in the functions SwapAdapterRouter.swapIn and SwapAdapterRouter.swapOut. It is important to determine the amount of tokens needed to exceed the maximum slippage. For this purpose, I used the swap tool in the Uniswap V3 app. 
![Screenshot from 2024-03-06 12-14-53](https://github.com/sherlock-audit/2024-02-smilee-finance-gstoyanovbg/assets/57452234/27abe1e5-c93e-4594-9e8d-cdd50a540712)
![Screenshot from 2024-03-06 11-15-57](https://github.com/sherlock-audit/2024-02-smilee-finance-gstoyanovbg/assets/57452234/cadedcc5-5f2e-43b3-a5b9-4bedd042be2b)
![Screenshot from 2024-03-06 11-15-57](https://github.com/sherlock-audit/2024-02-smilee-finance-gstoyanovbg/assets/57452234/2081da68-df27-47b2-9f61-22928e08ca32)
![Screenshot from 2024-03-06 11-15-10](https://github.com/sherlock-audit/2024-02-smilee-finance-gstoyanovbg/assets/57452234/b08e3aec-511d-464e-ad25-8c5e76892090)
![Screenshot from 2024-03-06 11-12-33](https://github.com/sherlock-audit/2024-02-smilee-finance-gstoyanovbg/assets/57452234/9e8686c1-a7be-471e-8138-b6f520c3d8e0)

From the screenshots, it can be seen that $3M is needed for wETH, $1M for ARB, $645K for GMX, and only $30K for JOE. These are not excessively large amounts and it is entirely possible to reach them accidentally with a large deposit or with several smaller deposits in one epoch. On the other hand, a malicious user may deliberately deposit an appropriate amount to cause a DOS on the protocol. Even if the Vault is killed, deposits cannot be withdrawn through rescueShares because the requirement is for the Vault to be dead, which happens after the epoch is rolled over. There are two working solutions left to recover locked deposits and profits:
1) Administrators (or someone else) to deposit a certain amount of the opposite token directly to the Vault. This way, the amount that needs to be exchanged will be reduced so as not to exceed the maximum slippage. The problem is that after the epoch is rolled over, these funds will be distributed among share holders and the sender will lose them.
2) By increasing the maximum slippage through SwapAdapterRouter.setSlippage (the maximum is 10%) and rolling over the epoch at the cost of serious losses for LPs. This may not be enough to solve the problem, as is the case with JOE. It will be necessary to wait for the set delay in TimeLock to expire.

<details>
<summary>POC</summary>
This POC show how someone can make DOS using a single deposit but the same can be achieved with multiple deposits or withdraws. The amounts in this POC are for illustration and don't have any market meaning. 

To run this poc first should replace the content of testnet/TestnetSwapAdapter.sol with the one below. The idea of the changes is to simulate slippage.

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

Then should add the function below to test/Scenarios.t.sol.

```solidity
function testDosWithDeposit() public 
    {
        string memory scenarioName = "scenario_1";
        bool isFirstEpoch = true;

        console.log(string.concat("Executing scenario: ", scenarioName));
        string memory scenariosJSON = _getTestsFromJson(scenarioName);

        console.log("- Checking start epoch");
        StartEpoch memory startEpoch = _getStartEpochFromJson(scenariosJSON);
        _checkStartEpoch(startEpoch, isFirstEpoch);

        address attacker = address(0x4);

        uint256 amount = 810000e18;

        TokenUtils.provideApprovedTokens(_admin, _vault.baseToken(), attacker, address(_vault), amount, vm);

        vm.startPrank(attacker);
        _vault.deposit(amount, attacker, 0);

         Utils.skipWeek(true, vm);
         vm.startPrank(_admin);
         //this call will revert due to slippage
         _dvp.rollEpoch();
    }
```

</details>

## Impact

Dos of the protocol, lock of funds which can be recovered at the cost of significant losses for administrators or LPs.

## Code Snippet

Above

## Tool used

Manual Review

## Recommendation

It is obvious that the slippage check cannot be removed because it protects the capital of LPs. A check for max deposits per epoch could be added, which, if well-calculated, would solve the problem, but it would severely limit the protocol. Another option is to introduce a max deposit per transaction and to balance the vault on every deposit (or when a certain amount is exceeded), not just when rolling over epochs. This is the solution I recommend.
