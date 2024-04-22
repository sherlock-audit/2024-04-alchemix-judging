Breezy Gingham Puppy

medium

# Anyone can donate the reward tokens of OptimismRewardCollector to a vault with no active rewards

## Summary

The contract `RewardRouter` calls `OptimismRewardCollector.claimAndDonateRewards` even if vault's `rewardAmount` is set to zero. This enables an arbitrary caller to swap the reward token `OP` to `alUSD` or `alETH` to a previously active vault, even if this vault's reward has been depleted.

## Vulnerability Detail

Consider the function `distributeRewards` of `RewardRouter`:

```js
    function distributeRewards(address vault) external returns (uint256) {
        // If vault is set to receive rewards from grants, send amount to reward collector to donate
        if (rewards[vault].rewardAmount > 0) {
          // ...
        }

>>      return IRewardCollector(rewards[vault].rewardCollectorAddress)
          .claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
    }
```

Notice that the `return` statement is executed even if `rewards[vault].rewardAmount == 0`. The only requirement is that `rewards[vault].rewardCollectorAddress` is set to a correct reward collector, e.g., `OptimismRewardCollector`. This is the case for any vault that was activated via `addVault` in the past, even if the reward has been paid for it in full.

Further, even though `vault` is passed as an argument to `getExpectedExchange`, the contract `OptimismRewardCollector` does not use this argument. It only computes the exchange rate for the reward token, that is, `OP`.

When we look at the code of `OptimismRewardCollector.claimAndDonateRewards`, it only uses `token` (in our case, the `vault` address) in the call to `donate`:

```js
    function claimAndDonateRewards(address token, uint256 minimumAmountOut) external returns (uint256) {
        require(msg.sender == rewardRouter, "Must be Reward Router"); 
        // Amount of reward token claimed plus any sent to this contract from grants.
>>      uint256 amountRewardToken = IERC20(rewardToken).balanceOf(address(this));
        if (amountRewardToken == 0) return 0;

        if (debtToken == 0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A) {
            // Velodrome Swap Routes: OP -> USDC -> alUSD
            // ...
        } else if (debtToken == 0x3E29D3A9316dAB217754d13b28646B76607c5f04) {
            // Velodrome Swap Routes: OP -> alETH
            // ...
        } else {
            revert IllegalState("Reward collector `debtToken` is not supported");
        }

        // Donate to alchemist depositors
        uint256 debtReturned = IERC20(debtToken).balanceOf(address(this));
        TokenUtils.safeApprove(debtToken, alchemist, debtReturned);
>>      IAlchemistV2(alchemist).donate(token, debtReturned);

        return amountRewardToken;
    }
```

The only requirement for `claimAndDonateRewards` to go through is to have some `rewardToken` on the balance of `OptimismRewardCollector`.

## Impact

Some debt may be returned to a vault that does not have active rewards anymore, under the condition of `OptimismRewardCollector` having reward tokens on its balance.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L35-L56
https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L57-L89
https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L91-L143

## Tool used

Manual Review

## Recommendation

Call `claimAndDonateRewards` only if `amountToSend` is positive:

```js
    function distributeRewards(address vault) external returns (uint256) {
        // If vault is set to receive rewards from grants, send amount to reward collector to donate
        if (rewards[vault].rewardAmount > 0) {
            // ...
            uint256 amountToSend = currentReward > maxReward ? maxReward : currentReward;
            // ...
+           if (amountToSend > 0) {
+               return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
+           }
        }

+       return 0;
-       return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
    }
```