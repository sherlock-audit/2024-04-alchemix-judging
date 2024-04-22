Brief Vinyl Rat

high

# user can mint infinite number of token while `_DURATION` time

## Summary
user can mint infinite number of token while `_DURATION` time

## Vulnerability Detail
In the `AlchemicTokenV2Base.sol` contract, `mint` function 
```solidity
function mint(address recipient, uint256 amount) external onlyWhitelisted {
      ...
    // If bridge is registered check limits and adjust them accordingly.
    if (xBridges[msg.sender].minterParams.maxLimit > 0) {
      uint256 currentLimit = mintingCurrentLimitOf(msg.sender);
      if (amount > currentLimit) revert IXERC20.IXERC20_NotHighEnoughLimits();
      _useMinterLimits(msg.sender, amount);
    }

    _mint(recipient, amount);
  }
```
`mintingCurrentLimitOf` function calls `_getCurrentLimit` function below 
```solidity
function _getCurrentLimit(
    uint256 _currentLimit,
    uint256 _maxLimit,
    uint256 _timestamp,
    uint256 _ratePerSecond
  ) internal view returns (uint256 _limit) {
    _limit = _currentLimit;
    if (_limit == _maxLimit) {
      return _limit;
    } else if (_timestamp + _DURATION <= block.timestamp) {
      _limit = _maxLimit;
    } else if (_timestamp + _DURATION > block.timestamp) {
      uint256 _timePassed = block.timestamp - _timestamp;
      uint256 _calculatedLimit = _limit + (_timePassed * _ratePerSecond);
      _limit = _calculatedLimit > _maxLimit ? _maxLimit : _calculatedLimit;
    }
  }
```

As we see,  `_limit ` will return as ` _maxLimit` if `_timestamp + _DURATION <= block.timestamp`.

This means that even while `_DURATION ` period of time , this function will always return `_maxLimit`.

Which then will be taken as a `currentLimit` in `mint` function, and compared to `amount`
```solidity
if (amount > currentLimit) revert IXERC20.IXERC20_NotHighEnoughLimits();
```

After that `_useMinterLimits` will also call `_getCurrentLimit` and receive `_maxLimit` as a current limit

## Impact
User is able to infinitely mint erc20 token while `_DURATION ` period of time

## Code Snippet
https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L127-L140

## Tool used

Manual Review

## Recommendation
Fix it