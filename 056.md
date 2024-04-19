Acrobatic Slate Seagull

medium

# AlchemicTokenV2Base allows burning tokens via the bridge even when it is paused

## Summary
The bridge can be paused at any time, but not all functions it can invoke will be suspended in `AlchemicTokenV2Base` smart contract.

## Vulnerability Detail
The caller with the sentinel role can pause the bridge from minting tokens:
```solidity
function pauseMinter(address minter, bool state) external onlySentinel {
    paused[minter] = state;
    emit Paused(minter, state);
  }
```
However, there is no restriction on burning xalAsset when the bridge is paused, which can lead to the following scenarios:

Scenario 1:
Prerequisite: xalUSD exists on both Optimism and Arbitrum chains.

1) A caller with the sentinel role pauses the bridge on both chains.
2) Any user bridges xalUSD from Optimism to Arbitrum (resulting in xalUSD being burned on Optimism).
3) However, the minting of xalUSD on Arbitrum fails because `AlchemicTokenV2Base` is paused.

Scenario 2:
Prerequisite: xalUSD exists on both Optimism and Arbitrum chains.

1) A caller with the sentinel role pauses the bridge on the Optimism chain.
2) Any user bridges xalUSD from Optimism to Arbitrum (resulting in xalUSD being burned on Optimism).
3) The user successfully mints xalUSD on the Arbitrum chain.

## Impact
Any user can burn their tokens but may not receive them on the destination chain.

## Code Snippet
[src/AlchemicTokenV2Base.sol#L190](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L190)

## Tool used

Manual Review

## Recommendation
Consider pausing the burn function for bridges as well:
```diff
function burnSelf(uint256 amount) external {
    // If bridge is registered check limits and update accordingly.
    if (xBridges[msg.sender].burnerParams.maxLimit > 0) {
+  if (paused[msg.sender]) {
+      revert IllegalState();
+   }
      uint256 currentLimit = burningCurrentLimitOf(msg.sender);
      if (amount > currentLimit) revert IXERC20.IXERC20_NotHighEnoughLimits();
      _useBurnerLimits(msg.sender, amount);
    }

    _burn(msg.sender, amount);
  }
  
  function burn(address account, uint256 amount) external {
    if (msg.sender != account) {
      uint256 newAllowance = allowance(account, msg.sender) - amount;
      _approve(account, msg.sender, newAllowance);
    }

    // If bridge is registered check limits and update accordingly.
    if (xBridges[msg.sender].burnerParams.maxLimit > 0) {
+  if (paused[msg.sender]) {
+      revert IllegalState();
+   }
      uint256 currentLimit = burningCurrentLimitOf(msg.sender);
      if (amount > currentLimit) revert IXERC20.IXERC20_NotHighEnoughLimits();
      _useBurnerLimits(msg.sender, amount);
    }

    _burn(account, amount);
  }
```


