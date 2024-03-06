Rural Blonde Cougar

medium

# IG options mint premium and burn payoff are unfair and sometimes incorrect when uniswap price is different from oracle

## Summary

When IG options are minted or burned, the premium (payoff) is calculated at the 2 different prices:
- oracle price
- price from actual hedge swap in uniswap

The worse of them (higher for premium, lower for payoff) is then used for the actual premium (payoff).

The problem is that hedge swap price is not always the correct price which should be used for this evaluation. Moreover, this swap price is applied to the whole minted (burned) position size, while the hedge amount might be very different, most of the time much smaller, but sometimes bigger than minted (burned) positions size.

Example when hedge swap price is not the correct:
1. No trades from strike price = 1 (vault: base = 1, side = 1)
2. When current price = 1.01, IG bull of size = 0.1 is bought. Delta hedge amount is to sell 0.116 of side token.
3. Delta hedge at uniswap happens at price:
3.1. Price = 0.99. Vault receives 0.11484 of base token (instead of 0.11716), so the vault is in a loss from the hedge. However, the premium for IG bull calculated at the price = 0.99 is much smaller than premium at the oracle price = 1.01, so the premium is not increased due to vault's loss from the hedge.
3.2. Price = 1.03. Vault receives 0.11948 of base token, so the vault is in a profit from the hedge. However, the premium for IG bull calculated at the price = 1.03 is much higher than premium at the oracle price = 1.01, so the premium charged from the user is several times larger than expected, even though the vault is in a profit from the hedge

This happens because when you buy IG bull, it is expected that delta hedge will be to **buy** side token, thus a higher hedge price = loss for the vault, and premium is higher at this higher price. However, is hedge is to sell, then it's the opposite (a lower hedge price = loss for the vault, but the premium is lower for the user, while a higher price = profit for the vault, but the premium is higher for the user).

Another problem is that most of the time, the hedge amount is much smaller than mint/burn amount, but the increase of premium (decrease of payoff) is applied to the whole mint/burn amount. At times this can be extremely unfair to the user, when the delta hedge amount is tiny, but the impact from the swap price to user position is huge, for example:
1. Strike = 1, Current price = 1.01 (vault: base = 1.133, side = 0.871)
2. Oracle price moves to 1.009
3. User buys IG bear option of size = 1
4. Vault delta hedges by buying 0.013 side token (this happens from price move, not from user position, which doesn't require delta hedge at this price)
5. If uniswap price = 0.99, then vault is in profit (buys at a smaller price), although it's tiny
6. But the user is charge a much higher premium:
6.1. Premium at oracle price = 0.0013
6.2. Actual premium (at uniswap price) = 0.0042 - almost 4 times the premium at oracle

As can be seen from this example, the vault is in a profit, but the user is charged significantly higher price than it should pay.

Now, a smart user might do this:
1. Buy IG bear option of size = 1e-18 (1 wei) which will delta hedge for free
2. Buy IG bear option of size = 1 (which will not delta hedge - delta hedge = 0) ==> no swap, oracle price is used for swap price
3. Premium paid is premium at oracle price = 0.0013

So the user who is not so smart will pay 4x the premium of smart user, which is extremely unfair.

## Vulnerability Detail

The `DVP._mint` function calculates position value at oracle and hedge swap price, choosing the greater of them for the premium:
```solidity
    uint256 swapPrice = _deltaHedgePosition(strike, amount, true);
    uint256 premiumOrac = _getMarketValue(strike, amount, true, IPriceOracle(_getPriceOracle()).getPrice(sideToken, baseToken));
    uint256 premiumSwap = _getMarketValue(strike, amount, true, swapPrice);
    premium_ = premiumSwap > premiumOrac ? premiumSwap : premiumOrac;
```

The `DVP._burn` function similarly calculates the payoff (only chooses the smaller value):
```solidity
    uint256 swapPrice = _deltaHedgePosition(strike, amount, false);
    uint256 payoffOrac = _getMarketValue(strike, amount, false, IPriceOracle(_getPriceOracle()).getPrice(sideToken, baseToken));
    uint256 payoffSwap = _getMarketValue(strike, amount, false, swapPrice);
    paidPayoff = payoffSwap < payoffOrac ? payoffSwap : payoffOrac;
```

The premium is calculated incorrectly (against the protocol) in the following cases:
- for IG Bull - if delta hedge amount is positive (meaning side token is sold), then the swap price which is less than oracle is bad for the vault (side token is sold for a cheaper price) but premium for the user is not higher to compensate for it
- for IG Bear - if delta hedge amount is negative (meaning side token is bought), then the swap price which is greater than oracle is bad for the vault (less side token is bought for a more expensive price) but premium for the user is not higher to compensate for it

The premium is calculated incorrectly (unfair for the user) in the following cases:
- for IG Bull - if delta hedge amount is positive (meaning side token is sold), then the swap price which is greater than oracle is good for the vault (side token is sold for a more expensive price) but premium for the user is much higher
- for IG Bear - if delta hedge amount is negative (meaning side token is bought), then the swap price which is less than oracle is good for the vault (more side token is bought for a cheaper price) but premium for the user is much higher
- for IG Bull - if current price is less than strike price, delta hedge amount is positive, but the hedge amount is tiny, swap price is greater than oracle - then vault is slightly in profit from the hedge, but user's premium is multiples higher
- for IG Bear - if current price is greater than strike price, delta hedge amount is negative, but the hedge amount is tiny, swap price is less than oracle - then vault is slightly in profit from the hedge, but user's premium is multiples higher

## Impact

In some edge cases, the protocol fails to account for the loss from the hedge swap at bad price, without charging additional premium from the user to compensate for it, generating expected loss for the protocol in the long term.

In some other cases, the premium charged from the user is multiples higher than it should be even though there is no negative impact for the protocol/vault.

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

    function testUnfairUserPremium() public {
        vm.prank(admin);
        ap.setDvpPositionManager(address(pm));

        _strike = 1e18;
        VaultUtils.addVaultDeposit(alice, 1e18, admin, address(vault), vm);
        VaultUtils.addVaultDeposit(bob, 1e18, admin, address(vault), vm);

        Utils.skipDay(true, vm);

        vm.prank(admin);
        ig.rollEpoch();

        vm.prank(admin);
        exchange.setSlippage(-0.02e18, 0, 0); // positive for the protocol
        testBuyOption(1.01e18, 0, 1);
        // testBuyOption(1.009e18, 0, 1); // uncomment to see how a smart user can reduce premium paid by 4x simply buying 1 wei option to do free hedging
        testBuyOption(1.009e18, 0, 1e18-2); // premium charges is 4x the normal premium
        vm.prank(admin);
    }

    function testBuyOption(uint price, uint128 optionAmountUp, uint128 optionAmountDown) internal {

        vm.prank(admin);
        priceOracle.setTokenPrice(address(sideToken), price);

        (uint256 premium, uint256 fee) = _assurePremium(charlie, _strike, optionAmountUp, optionAmountDown);
        console.log("premium calc. ", premium);

        vm.startPrank(charlie);
        premium = ig.mint(charlie, _strike, optionAmountUp, optionAmountDown, premium, 10e18, 0);
        vm.stopPrank();

        console.log("premium actual", premium);
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
  premium calc.  0
  premium actual 0
  baseToken balance 1132810056821010091
  sideToken balance 871083229643748698
...
  premium calc.  1285310999122248
  premium actual 4232769803106572
  baseToken balance 1124380613630074973
  sideToken balance 883888606753881676
...
```

Notice:
1. Vault receives +2% of profit when doing delta hedge
2. Yet the user's premium is 4x times the premium it should have

## Code Snippet

`DVP._mint` chooses the higher of 2 valuations at oracle and swap prices:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol#L152-L155

`DVP._burn` chooses the smaller of 2 valuations at oracle and swap prices:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/DVP.sol#L269-L272

## Tool used

Manual Review

## Recommendation

Add actual loss of protocol from hedging to the premium (hedge amount * (oracle price - swap price)), or 0 if protocol is in profit. This will ensure fair and appropriate premium increase regardless of circumstances.