Sweet Hazel Weasel

medium

# Griefing attack against `distributeRewards` function

## Summary

A malicious user could perform a griefing attack by intentionally triggering the `distributeRewards` function on every block to prevent the other users from receiving the rewards.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L42

```solidity
File: RewardRouter.sol
35:     function distributeRewards(address vault) external returns (uint256) {
36:         // If vault is set to receive rewards from grants, send amount to reward collector to donate
37:         if (rewards[vault].rewardAmount > 0) {
38:             // Calculates ratio of timeframe to time since last harvest
39:             // Uses this ratio to determine partial reward amount or extra reward amount
40:             uint256 blocksSinceLastReward = block.number - rewards[vault].lastRewardBlock;
41:             uint256 maxReward = rewards[vault].rewardAmount - rewards[vault].rewardPaid;
42:             uint256 currentReward = rewards[vault].rewardAmount * blocksSinceLastReward / rewards[vault].rewardTimeframe;
43:             uint256 amountToSend = currentReward > maxReward ? maxReward : currentReward;
44: 
45:             TokenUtils.safeTransfer(IRewardCollector(rewards[vault].rewardCollectorAddress).rewardToken(), rewards[vault].rewardCollectorAddress, amountToSend);
46:             rewards[vault].lastRewardBlock = block.number;
47:             rewards[vault].rewardPaid += amountToSend;
48: 
49:             if (rewards[vault].rewardPaid == rewards[vault].rewardAmount) {
50:                 rewards[vault].rewardAmount = 0;
51:                 rewards[vault].rewardPaid = 0;
52:             }
53:         }
54: 
55:         return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
56:     }
```

Line 42 above uses the following formula to compute the current reward:

```solidity
blocksSinceLastReward = block.number - rewards[vault].lastRewardBlock;
currentReward = rewards[vault].rewardAmount * blocksSinceLastReward / rewards[vault].rewardTimeframe;
```

However, the issue is that the above formula will round down to zero under the following certain conditions:

1. A malicious user intentionally triggers the `distributeRewards` function on every block (the gas fee is cheap on L2). As a result, the `blocksSinceLastReward` will always be one. If the `rewardAmount` is less than the `rewards[vault].rewardTimeframe`, it will round down to zero.
2. The reward amount is small compared to the reward timeframe. In this case, even if there are no malicious activities, the `rewardAmount` multiplied by `blocksSinceLastReward` could still be less than the denominator (`rewardTimeframe`), resulting in rounding down to zero.

The first condition is of greater concern as a malicious user could perform a griefing attack by intentionally triggering the `distributeRewards` function on every block to prevent the other users from receiving the rewards.

## Impact

Users are unable to receive the rewards.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L42

## Tool used

Manual Review

## Recommendation

If the `currentReward` is zero or less than a certain threshold, consider exiting the function immediately and bypassing the rest of the operations (e.g., updating of state) to avoid malicious users from executing a griefing attack.

```diff
function distributeRewards(address vault) external returns (uint256) {
    // If vault is set to receive rewards from grants, send amount to reward collector to donate
    if (rewards[vault].rewardAmount > 0) {
        // Calculates ratio of timeframe to time since last harvest
        // Uses this ratio to determine partial reward amount or extra reward amount
        uint256 blocksSinceLastReward = block.number - rewards[vault].lastRewardBlock;
        uint256 maxReward = rewards[vault].rewardAmount - rewards[vault].rewardPaid;
        uint256 currentReward = rewards[vault].rewardAmount * blocksSinceLastReward / rewards[vault].rewardTimeframe;
        uint256 amountToSend = currentReward > maxReward ? maxReward : currentReward;
        
+       if (amountToSend < minAmount) return 0
```