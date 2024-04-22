Sweet Hazel Weasel

high

# Hardcoded swap path

## Summary

Hardcoded swap path within the reward collector contract would lead to a loss of assets when swapping the reward tokens for debt tokens.

## Vulnerability Detail

Per Linr 61 below, the reward collector will always attempt to swap the entire reward balance (OP tokens) on its contract to the debt assets (alETH or alUSD) via Velodrome on the Optimism chain.

The reward tokens (OP) can potentially be sent to the reward collector by Reward Router or transferred directly into the contract by anyone.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L61

```solidity
File: OptimismRewardCollector.sol
57:     function claimAndDonateRewards(address token, uint256 minimumAmountOut) external returns (uint256) {
58:         require(msg.sender == rewardRouter, "Must be Reward Router");
59: 
60:         // Amount of reward token claimed plus any sent to this contract from grants.
61:         uint256 amountRewardToken = IERC20(rewardToken).balanceOf(address(this));
..SNIP..
```

Per Line 67 and Line 74 below, it was found that the routes for the swap are hardcoded, which will lead to a number of issues:

#### Issue 1 - Stuck reward tokens due to pools with insufficient liquidity

Assume there is a large amount of OP tokens (e.g., 1M OP) residing on the reward collector contract. If one of the pools in the routing path does not have sufficient liquidity to perform the swap for the "in-transit" tokens, then the swap will always fail and revert.

Note that there is no point in waiting for liquidity to be replenished in a pool, as there is no guarantee that there will always be LPs available to provide liquidity to a pool.

The OP tokens will be stuck in the reward collector contract with no way to extract them as there is no sweeping function for the owner to recover the OP tokens.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L61

```solidity
File: OptimismRewardCollector.sol
57:     function claimAndDonateRewards(address token, uint256 minimumAmountOut) external returns (uint256) {
58:         require(msg.sender == rewardRouter, "Must be Reward Router");
59: 
60:         // Amount of reward token claimed plus any sent to this contract from grants.
61:         uint256 amountRewardToken = IERC20(rewardToken).balanceOf(address(this));
62: 
63:         if (amountRewardToken == 0) return 0;
64: 
65:         if (debtToken == 0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A) {
66:             // Velodrome Swap Routes: OP -> USDC -> alUSD
67:             IVelodromeSwapRouter.Route[] memory routes = new IVelodromeSwapRouter.Route[](2);
68:             routes[0] = IVelodromeSwapRouter.Route(0x4200000000000000000000000000000000000042, 0x7F5c764cBc14f9669B88837ca1490cCa17c31607, false, 0xF1046053aa5682b4F9a81b5481394DA16BE5FF5a);
69:             routes[1] = IVelodromeSwapRouter.Route(0x7F5c764cBc14f9669B88837ca1490cCa17c31607, 0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A, true, 0xF1046053aa5682b4F9a81b5481394DA16BE5FF5a);
70:             TokenUtils.safeApprove(rewardToken, swapRouter, amountRewardToken);
71:             IVelodromeSwapRouter(swapRouter).swapExactTokensForTokens(amountRewardToken, minimumAmountOut, routes, address(this), block.timestamp);
72:         } else if (debtToken == 0x3E29D3A9316dAB217754d13b28646B76607c5f04) {
73:             // Velodrome Swap Routes: OP -> alETH
74:             IVelodromeSwapRouter.Route[] memory routes = new IVelodromeSwapRouter.Route[](2);
75:             routes[0] = IVelodromeSwapRouter.Route(0x4200000000000000000000000000000000000042, 0x4200000000000000000000000000000000000006, false, 0xF1046053aa5682b4F9a81b5481394DA16BE5FF5a);
76:             routes[1] = IVelodromeSwapRouter.Route(0x4200000000000000000000000000000000000006, 0x3E29D3A9316dAB217754d13b28646B76607c5f04, true, 0xF1046053aa5682b4F9a81b5481394DA16BE5FF5a);
77:             TokenUtils.safeApprove(rewardToken, swapRouter, amountRewardToken);
78:             IVelodromeSwapRouter(swapRouter).swapExactTokensForTokens(amountRewardToken, minimumAmountOut, routes, address(this), block.timestamp);
79:         } else {
80:             revert IllegalState("Reward collector `debtToken` is not supported");
81:         }
```

#### Issue 2 - Swap with bad slippage

When a pool has insufficient liquidity or imbalanced liquidity (e.g., the pool's reserve_1 is much larger than the pool's reserve_2), the slippage will be very bad. Since the swap's routing path is hardcoded, the reward collector contract is forced to relax its slippage control and perform a swap against these pools, which incurs large slippage, leading to a loss for the protocol/users.

## Impact

Loss of reward tokens.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L61

## Tool used

Manual Review

## Recommendation

Consider allowing the Alchemix DAO to update the swap's routing path so that it can be adjusted according to market conditions. Alternatively, allow the unused reward tokens to be recovered from the contract.