Low Mauve Crab

high

# The epoch rolling mechanism is incorrect, leading to shorter epochs than intended

## Summary
The epoch rolling mechanism calculates the next epoch's expiry as `REF_TS + N*frequency` where N is some integer. However this means that it is possible for the next epoch's expiry to be shorter than 7 days from the time of rolling.

## Vulnerability Detail
The following 3 functions are used to determine the new epoch's expiry:

```solidity
function _getNextExpiry(uint256 from, uint256 timespan) private view returns (uint256 nextExpiry) {
    nextExpiry = EpochFrequency.nextExpiry(from, timespan);
    // If next epoch expiry is in the past go to next of the next
    while (block.timestamp > nextExpiry) {
        nextExpiry = EpochFrequency.nextExpiry(nextExpiry, timespan); 
    }
}
```
```solidity
function nextExpiry(uint256 ts, uint256 frequency) public pure returns (uint256 expiry) {
        validityCheck(frequency);

        expiry = _nextTimeSpanExpiry(ts, frequency);
    }
```
```solidity
function _nextTimeSpanExpiry(uint256 ts, uint256 timeSpan) private pure returns (uint256 nextExpiry_) {
        return REF_TS + _upDiv(ts - REF_TS, timeSpan) * timeSpan; //@audit check this out deeply
    }
```
It adds `frequency` to the `REF_TS` until it is greater than or equal to `block.timestamp`, and then sets that value as `nextExpiry`. However this means that `nextExpiry` can be set to be anything between the range of `[block.timestamp, block.timestamp + 7 days]`
However `nextExpiry` should be 7 days from the timestamp of rolling the epoch.

## Impact
Epochs are shorter than expected (its even possible for the epoch expiry to be set to `block.timestamp` of rolling into the epoch). However the theta value used in finance calculations is still calculated via `epoch.frequency` being the expected epoch length. This inconsistency will lead to incorrect payoff calculation, and incorrect pricing of IG options.

## Code Snippet
https://github.com/sherlock-audit/2024-02-smilee-finance/blob/3241f1bf0c8e951a41dd2e51997f64ef3ec017bd/smilee-v2-contracts/src/lib/EpochController.sol#L35-L48

## Proof of Concept
Add the following foundry test to `test/unit/EpochControls.t.sol`:

```solidity
function test_wrongEpochDurationCalculation() public {
        vm.warp(REF_TS);
        MockedEpochControls ec = new MockedEpochControls(7 days, 1 days);

        uint256 today = 1709563369;
        vm.warp(today);
        ec.rollEpoch();
        Epoch memory epoch = ec.getEpoch();
        uint256 expiry = epoch.current;
        assert(expiry >= today + 7 days); // this assertion fails!
    }
```

## Tool used
Manual Review

## Recommendation
Ensure that the time until `nextExpiry` is greater than or equal to `epoch.frequency`.

Example mitigation:
```diff
function _getNextExpiry(uint256 from, uint256 timespan) private view returns (uint256 nextExpiry) {
        
        nextExpiry = EpochFrequency.nextExpiry(from, timespan); // @audit I think this aint calculated properly.
        // If next epoch expiry is in the past go to next of the next

-        while (block.timestamp > nextExpiry) {
+       while(nextExpiry  < block.timestamp + timespan)
            nextExpiry = EpochFrequency.nextExpiry(nextExpiry, timespan); 
        }
    }
```