Sweet Hazel Weasel

medium

# Slippage can be disabled due to rounding error

## Summary

Under certain conditions, rounding errors could disable the slippage control, leading to trade incurs slippage losses or vulnerability to sandwich attacks.

## Vulnerability Detail

The value returned from the `getExpectedExchange` function is used as slippage control during a swap. If the returned value is zero, the slippage control will be disabled, and the trade might suffer from slippage loss or be vulnerable to sandwich attacks.

At Line 136, the `expectedExchange` might round down to zero under certain conditions.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L136

```solidity
File: OptimismRewardCollector.sol
91:     function getExpectedExchange(address yieldToken) external view returns (uint256) {
..SNIP..
132:         // Find expected amount out before calling harvest
133:         if (debtToken == alUsdOptimism) {
134:             expectedExchange = totalToSwap * uint(opToUsd) / 1e8;
135:         } else if (debtToken == alEthOptimism) {
136:             expectedExchange = totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd);
137:         } else {
138:             revert IllegalState("Invalid debt token");
139:         }
```

Currently, the price returned from `opToUsd` is `221047305` ($2.21), while the price returned from `ethToUsd` is `308596000000` (\$3085.96). If the `totalToSwap` at Line 136 is less than or equal to 1396, the `expectedExchange` will be rounded to zero.

The price of ETH is generally much higher than the price of OP token. Thus, if the price difference between ETH and OP tokens widens, this issue will be aggravated.

Since the `distributeRewards` function (see below) is permissionless, anyone can execute it. A malicious user can attempt to trigger this function every block. As such, the `blocksSinceLastReward` will be set to a small value (value=1). Depending on the reward amount and reward timeframe configured for a specific vault, it might result in the `amountToSend` being less than or equal to 1396, resulting in the slippage control being disabled (set to zero).

When the reward collector executes a swap with slippage disabled on Velodrome pools, malicious users could proceed to perform sandwich attacks to siphon the tokens. The gas fee is extremely cheap on L2, so it might be profitable to carry out such attacks.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L35

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

## Impact

Loss of reward tokens.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L136

## Tool used

Manual Review

## Recommendation

To prevent potential rounding errors when handling the rewards, consider only distributing them to reward collectors once they exceed a certain minimum threshold.