Sweet Hazel Weasel

medium

# `block.timestamp` should be used instead of `block.number`

## Summary

The reward distribution should be a variable of time instead of a block number, as the reward distribution is the ratio of the timeframe to  "time since the last harvest" per the source code's comment. Thus, `block.timestamp` should be used here instead of `block.number` to ensure the amount of rewards streamed or distributed continuously over a period of time

## Vulnerability Detail

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L38

```solidity
File: RewardRouter.sol
34:     /// @dev Distributes grant rewards and triggers reward collector to claim and donate
35:     function distributeRewards(address vault) external returns (uint256) {
36:         // If vault is set to receive rewards from grants, send amount to reward collector to donate
37:         if (rewards[vault].rewardAmount > 0) {
38:             // Calculates ratio of timeframe to time since last harvest
39:             // Uses this ratio to determine partial reward amount or extra reward amount
40:             uint256 blocksSinceLastReward = block.number - rewards[vault].lastRewardBlock;
41:             uint256 maxReward = rewards[vault].rewardAmount - rewards[vault].rewardPaid;
42:             uint256 currentReward = rewards[vault].rewardAmount * blocksSinceLastReward / rewards[vault].rewardTimeframe;
43:             uint256 amountToSend = currentReward > maxReward ? maxReward : currentReward;
```

Currently, the rewards are distributed based on the number of blocks that have been passed since the block of the last reward distribution/harvest. As a result, the `block.number` is used to fetch the current L2 block number.

However, per the comment in Line 38 above:

```solidity
// Calculates ratio of timeframe to time since last harvest
```

As such, the reward distribution should be a variable of time instead of a block number, as the reward distribution is the ratio of the timeframe to "time since the last harvest" per the source code's comment. Thus, `block.timestamp` should be used here instead to ensure the amount of rewards streamed or distributed continuously over a period of time.

## Impact

Using `block.number` could potentially lead to a number of pitfalls or impacts in this case:

1. A block is minted in Optimism every 2 seconds, while a block is minted in Ethereum every 12 seconds. Thus, the admin might erroneously apply the wrong block minting period (12 seconds instead of 2 seconds) when computing the reward timeframe, leading to a shorter timeframe and faster reward distribution.
2. When the sequencer is stopped for a certain reason, the block minting is also paused, and the block number will stop increasing. However, while the sequencer is stopped, the reward router will fail to take into consideration the amount of time that has already passed when computing the number of rewards to be distributed when the sequencer resumes later. Thus, it breaks the requirement of distributing the rewards based on the ratio of timeframe to time since the last harvest, as mentioned earlier.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L38

## Tool used

Manual Review

## Recommendation

Consider using `block.timestamp` instead of `block.number` while computing the number of rewards to distribute.