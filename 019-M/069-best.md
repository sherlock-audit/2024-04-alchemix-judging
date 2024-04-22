Petite Golden Crocodile

medium

# `OptimismRewardCollector` uses `ETH` oracle but swap to `WETH` assuming the price is 1:1

## Summary

OptimismRewardCollector assumes that WETH price vs ETH price is 1:1

## Vulnerability Detail

When rewards are distributed through `RewardRouter.distributeRewards()`, at the end it calls `OptimismRewardCollector.claimAndDonateRewards()`, and pass the result from `OptimismRewardCollector.getExpectedExchange` as `minimumAmountOut` to `claimAndDonateRewards()`. 

```solidity
function distributeRewards(address vault) external returns (uint256) {
    ...

    return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
}

// function claimAndDonateRewards(address token, uint256 minimumAmountOut)
```

`OptimismRewardCollector.getExpectedExchange` converts from a `rewardToken`(Optimism) amount to a corresponding `USD` or `ETH` value based on the `debtToken` (can be `alUSD` or `alETH`).

```solidity
function getExpectedExchange(address yieldToken) external view returns (uint256) {
    uint256 expectedExchange;
    address[] memory token = new address[](1);
    uint256 totalToSwap = TokenUtils.safeBalanceOf(rewardToken, address(this));

    // Ensure that round is complete, otherwise price is stale.
    (
        uint80 roundID,
        int256 opToUsd,
        ,
        uint256 updateTime,
        uint80 answeredInRound
    ) = IChainlinkOracle(opToUsdOracle).latestRoundData();
    
    require(
        opToUsd > 0, 
        "Chainlink Malfunction"
    );

    if( updateTime < block.timestamp - 1200 seconds ) {
        revert("Chainlink Malfunction");
    }

    // Ensure that round is complete, otherwise price is stale.
    (
        uint80 roundIDEth,
        int256 ethToUsd,
        ,
        uint256 updateTimeEth,
        uint80 answeredInRoundEth
    ) = IChainlinkOracle(ethToUsdOracle).latestRoundData();
    
    require(
        ethToUsd > 0, 
        "Chainlink Malfunction"
    );

    if( updateTimeEth < block.timestamp - 1200 seconds ) {
        revert("Chainlink Malfunction");
    }

    // Find expected amount out before calling harvest
    if (debtToken == alUsdOptimism) {
        expectedExchange = totalToSwap * uint(opToUsd) / 1e8;
    } else if (debtToken == alEthOptimism) {
        expectedExchange = totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd);
    } else {
        revert IllegalState("Invalid debt token");
    }

    return expectedExchange;
}
```

We will be looking at the `alEthOptimism` case. It will first convert `rewardToken`(Optimism) to `USD` and then divide by `ETH`, thus finding how much `alETH` to get for passed `totalToSwap`.

`ethToUsd` price is the actual price of ETH taken from here address constant ethToUsdOracle = [`0x13e3Ee699D1909E989722E753853AE30b17e08c5`](https://optimistic.etherscan.io/address/0x13e3Ee699D1909E989722E753853AE30b17e08c5);

Then in `claimAndDonateRewards()` when `debtToken` is `alETH` (already deployed at [`0x3E29D3A9316dAB217754d13b28646B76607c5f04`](https://optimistic.etherscan.io/address/0x3E29D3A9316dAB217754d13b28646B76607c5f04). It will swap from **Optimism** [`0x4200000000000000000000000000000000000042`](https://optimistic.etherscan.io/address/0x4200000000000000000000000000000000000042) to **`WETH`** on Optimism [`0x4200000000000000000000000000000000000006`](https://optimistic.etherscan.io/token/0x4200000000000000000000000000000000000006) and then to `alETH` [`0x3E29D3A9316dAB217754d13b28646B76607c5f04`](https://optimistic.etherscan.io/address/0x3E29D3A9316dAB217754d13b28646B76607c5f04), but as we showed above, it converts the `minimumAmountOut` that is passed to an `ETH` value.

```solidity
function claimAndDonateRewards(address token, uint256 minimumAmountOut) external returns (uint256) {
    require(msg.sender == rewardRouter, "Must be Reward Router"); 

    // Amount of reward token claimed plus any sent to this contract from grants.
    uint256 amountRewardToken = IERC20(rewardToken).balanceOf(address(this));

    if (amountRewardToken == 0) return 0;

    if (debtToken == 0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A) {
    ... USD Case
    } else if (debtToken == 0x3E29D3A9316dAB217754d13b28646B76607c5f04) {
        // Velodrome Swap Routes: OP -> alETH
        IVelodromeSwapRouter.Route[] memory routes = new IVelodromeSwapRouter.Route[](2);
        routes[0] = IVelodromeSwapRouter.Route(0x4200000000000000000000000000000000000042, 0x4200000000000000000000000000000000000006, false, 0xF1046053aa5682b4F9a81b5481394DA16BE5FF5a);
        routes[1] = IVelodromeSwapRouter.Route(0x4200000000000000000000000000000000000006, 0x3E29D3A9316dAB217754d13b28646B76607c5f04, true, 0xF1046053aa5682b4F9a81b5481394DA16BE5FF5a);
        TokenUtils.safeApprove(rewardToken, swapRouter, amountRewardToken);
        IVelodromeSwapRouter(swapRouter).swapExactTokensForTokens(amountRewardToken, minimumAmountOut, routes, address(this), block.timestamp);
    } else {
        revert IllegalState("Reward collector `debtToken` is not supported");
    }

    // Donate to alchemist depositors
    uint256 debtReturned = IERC20(debtToken).balanceOf(address(this));
    TokenUtils.safeApprove(debtToken, alchemist, debtReturned);
    IAlchemistV2(alchemist).donate(token, debtReturned);

    return amountRewardToken;
}
```

Here you can see the price chart of `ETH` vs `WETH` for `April 18, 2024` and as I indicated the maximum difference was `$41`.

> If search a bit harder can find an example for almost $100 diff
> 

![Untitled](https://i.imgur.com/fO5wvTL.png)

That `$41` in `getExpectedExchange()` will result in `4100000000` added to the number, but then will swap to `WETH` which was actually at `$3019` compared to `ETH` at `$3060`.

Example params for `18 April 2024 2:30 PM`

- totalToSwap (rewardToken in `OptimismRewardCollector`) = **1000e18**
- Optimism price = **$2.24 (223989858 from Oracle)**
- ETH price = **$3,060.00 (306000000000 from Oracle)**
- WETH price = **$3,019,00 (301900000000 if use Oracle)**

```solidity
// Find expected amount out before calling harvest
if (debtToken == alUsdOptimism) {
    expectedExchange = totalToSwap * uint(opToUsd) / 1e8;
} else if (debtToken == alEthOptimism) {
    expectedExchange = totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd);
} else {
    revert IllegalState("Invalid debt token");
}
```

The result will be as follow:

- If use ETH price
    - 1000e18 * **223989858 / 306000000000 = 731993000000000000 (0.731993 alETH)**
- If use WETH price
    - 1000e18 * **223989858 / 301900000000 = 741933945014905597.88………….. (0.741933945014905597880092…….. alETH)**

This number will be passed as `minimumAmountOut`, when perform the swap. In this case above (`ETH > WETH`), the `minimumAmountOut` will be lower that the actual, causing the user to lose out on their rewards when the swap takes place. In the other case (`WETH > ETH`), the `minimumAmountOut` will be higher that the actual, which may revert (because of slippage) in cases where it is close to it, but need to be lower.

## Impact

Using the wrong Oracle to compute the slippage may block the swap or lose user rewards.

## Code Snippet
https://github.com/sherlock-audit/2024-04-alchemix/blob/32e4902f77b05ea856cf52617c55c3450507281c/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L136

https://github.com/sherlock-audit/2024-04-alchemix/blob/32e4902f77b05ea856cf52617c55c3450507281c/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L76-L78

## Tool used

Manual Review

## Recommendation

Consider using a WETH oracle to be as accurate as possible.