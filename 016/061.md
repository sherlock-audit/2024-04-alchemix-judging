Acrobatic Slate Seagull

medium

# There is a mistake in the logic of the `_calculateNewCurrentLimit` function

## Summary
When the admin updates the limits for the bridge and the new `mintingMaxLimit` is lower than the previous `mintingMaxLimit` for this bridge, there is an issue in the calculation of the new current minting limit.

## Vulnerability Detail
The `mintingCurrentLimit` and `burningCurrentLimit` are calculated according to the following formula, provided that `currentLimit != maxLimit` and `_timestamp + _DURATION > block.timestamp`:
```solidity
      uint256 _timePassed = block.timestamp - _timestamp;
      uint256 _calculatedLimit = _limit + (_timePassed * _ratePerSecond);
      _limit = _calculatedLimit > _maxLimit ? _maxLimit : _calculatedLimit;
```
The problem lies in the calculation of `_newCurrentLimit`, which decreases by the difference between `oldMaxLimit and newMaxlimit` when `_oldMaxLimit > _maxLimit`. This calculation is incorrect under a certain condition.
Let's take a closer look at this issue:

1) The admin first calls `setLimits` for the bridge with the parameters `setLimits(_bridge, 1000e18, 1000e18)`.
After this call, `mintingCurrentLimit, burningCurrentLimit, mintingMaxLimit, and burningMaxLimit` are all set to 1000e18.
2) Some time passes, and users interact with the token through the bridge, with `mintingCurrentLimit` assumed to be 400e18 at the moment (the number of tokens the bridge can currently mint).
3) The admin wants to change the `maxMintingLimit` and `maxBurningLimit` to 800e18, but an incorrect calculation of `currentMintingLimit` occurs. The `newCurrentLimit` will be 200e18 instead of the previous 400e18:
```solidity
function _calculateNewCurrentLimit(
    uint256 _limit,  // 800e18
    uint256 _oldLimit, //1000e18
    uint256 _currentLimit // 400e18
  ) internal pure returns (uint256 _newCurrentLimit) {
    uint256 _difference;

    if (_oldLimit > _limit) { // 1000e18 > 800e18
      _difference = _oldLimit - _limit; // 1000e18 > 800e18 = 200e18
      _newCurrentLimit = _currentLimit > _difference ? _currentLimit - _difference : 0; // 400e18 - 200e18 = 200e18
    } else {
      _difference = _limit - _oldLimit;
      _newCurrentLimit = _currentLimit + _difference;
    }
  }
  
  function _changeMinterLimit(address _bridge, uint256 _limit) internal {
 xBridges[_bridge].minterParams.currentLimit = _calculateNewCurrentLimit(_limit, _oldLimit, _currentLimit);
    xBridges[_bridge].minterParams.ratePerSecond = _limit / _DURATION;
    xBridges[_bridge].minterParams.timestamp = block.timestamp; 
}
```
The `_calculateNewCurrentLimit` function correctly calculates `_newCurrentLimit` if `_currentLimit == _oldMaxLimit`, but otherwise, `currentMintingLimit` is reduced by `(oldMaxMintingLimit - newMaxMintingLimit)`, which is incorrect logic.

----

```code
_difference = _oldLimit - _limit;
_newCurrentLimit = _currentLimit > _difference ? _currentLimit - _difference : 0;

Invalid case:
     oldMaxLimit = 1000e18, newMaxLimit = 800e18, currentLimit = 805e18
    _difference = 1000e18 - 800e18 = 200e18
    _newCurrentLimit  = 805e18 - 200e18 = 605e18
    
Valid case:
     oldMaxLimit = 1000e18, newMaxLimit = 800e18, currentLimit = 805e18
     _newCurrentLimit = 800e18

```



## Impact
The `_currentLimit` decreases when `_currentLimit < _limit`, which is incorrect.

## Code Snippet
[src/AlchemicTokenV2Base.sol#L418-L420](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L418-L420)

## Tool used

Manual Review

## Recommendation
Consider changing the `_calculateNewCurrentLimit` function, which calculates `mintingCurrentLimit and burningCurrentLimit`:
```diff
function _calculateNewCurrentLimit(
    uint256 _limit,
    uint256 _oldLimit,
    uint256 _currentLimit
  ) internal pure returns (uint256 _newCurrentLimit) {
    uint256 _difference;

    if (_oldLimit > _limit) {
+    _newCurrentLimit = _currentLimit > _limit ? _limit : _currentLimit;
-      _difference = _oldLimit - _limit;
-      _newCurrentLimit = _currentLimit > _difference ? _currentLimit - _difference : 0;
    } else {
      _difference = _limit - _oldLimit;
      _newCurrentLimit = _currentLimit + _difference;
    }
  }
```
