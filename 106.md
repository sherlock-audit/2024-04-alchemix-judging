Sweet Hazel Weasel

medium

# Protocol vulnerable to donation attacks

## Summary

The protocol will be vulnerable to donation attacks as the access control of the `OptimismRewardCollector.claimAndDonateRewards` function can be bypassed.

## Vulnerability Detail

To prevent a possible donation attack, the core `AlchemistV2.donate` function can only be accessed by a whitelisted address per Line 907 below.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemistV2.sol#L907

```solidity
File: AlchemistV2.sol
905:     /// @inheritdoc IAlchemistV2Actions
906:     function donate(address yieldToken, uint256 amount) external override lock {
907:         _onlyWhitelisted();
908:         _checkArgument(amount > 0);
...SNIP..
```

However, with the new updates introduced in this audit, anyone could technically bypass this restriction and perform a donation even if their accounts are not whitelisted in the system.

The following are described steps to perform a donation attack even if Bob's account is not whitelisted.

1. Bob directly transfers a large number of reward tokens (e.g., OP tokens) to the `OptimismRewardCollector` contract. As per Line 61 in the `OptimismRewardCollector.claimAndDonateRewards` function below, all the reward tokens (including Bob's donated assets) residing on the contract will be swapped to the debt token and donated to the protocol at Line 86 below.

   The issue is that Line 58 will ensure that only the Reward Router can execute this function. However, this is not an issue for him because there is a way to bypass this access control in the next step.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L61

```solidity
File: OptimismRewardCollector.sol
57:     function claimAndDonateRewards(address token, uint256 minimumAmountOut) external returns (uint256) {
58:         require(msg.sender == rewardRouter, "Must be Reward Router");
59: 
60:         // Amount of reward token claimed plus any sent to this contract from grants.
61:         uint256 amountRewardToken = IERC20(rewardToken).balanceOf(address(this));
..SNIP..
83:         // Donate to alchemist depositors
84:         uint256 debtReturned = IERC20(debtToken).balanceOf(address(this));
85:         TokenUtils.safeApprove(debtToken, alchemist, debtReturned);
86:         IAlchemistV2(alchemist).donate(token, debtReturned);
87: 
88:         return amountRewardToken;
89:     }
```

2. Since the `RewardRouter.distributeRewards` function is permissionless, Bob can call it, which will, in turn, call the `OptimismRewardCollector.claimAndDonateRewards` function at Line 55 below on behalf of Bob. In this case, Bob successfully triggered the `OptimismRewardCollector.claimAndDonateRewards` function, and his donated reward tokens will be added to the protocol.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L35

```solidity
File: RewardRouter.sol
34:     /// @dev Distributes grant rewards and triggers reward collector to claim and donate
35:     function distributeRewards(address vault) external returns (uint256) {
..SNIP..
54: 
55:         return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
56:     }
```

## Impact

The protocol will be vulnerable to donation attacks as the access control of the `OptimismRewardCollector.claimAndDonateRewards` function can be bypassed.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L61

## Tool used

Manual Review

## Recommendation

To prevent a potential donation attack, consider swapping and donating only the amount of reward tokens sent directly from the Reward Router instead of swapping and donating the entire balance residing on the contract. This can be achieved by passing in the number of reward tokens to be swapped/donated as a parameter of the `claimAndDonateRewards` function when the Reward Router calls this function when distributing the rewards.