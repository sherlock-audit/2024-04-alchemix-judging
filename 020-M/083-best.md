Lucky Olive Cottonmouth

high

# Incorrect slippage calculation will cause issues with swaps when debt token is alETH

## Summary
`OptimismRewardCollector::getExpectedExchange` calculates the incorrect `expectedExchange` due to decimal mismatch when `debtToken` is `alETH`. The `expectedExchange` value will have a lower amount of decimals than expected, which is used for the `minAmountOut` slippage protection for swapping from OP -> alETH. Since `minAmountOut` decimals are lower than the expected token, there is essentially no slippage protection on this swap, leading to sandwich attacks and causing loss of funds.

## Vulnerability Detail
`OptimismRewardCollector` has two functionalities: 

`claimAndDonateRewards`: Claim reward token, swap for alUSD or alETH, and proceed to donate to alchemist depositors
`getExchangeRate`: Calculate expected exchange rate, used for minimum amount out slippage calculation for the swaps.

When rewards are distributed via `RewardRouter::distributeRewards`:

```javascript
    /// @dev Distributes grant rewards and triggers reward collector to claim and donate
    function distributeRewards(address vault) external returns (uint256) {
        // If vault is set to receive rewards from grants, send amount to reward collector to donate
        if (rewards[vault].rewardAmount > 0) {
            .
            .
            .

            TokenUtils.safeTransfer(IRewardCollector(rewards[vault].rewardCollectorAddress).rewardToken(), rewards[vault].rewardCollectorAddress, amountToSend);
            .      
            .
            .

        return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
    }
```

swaps are performed depending on the address of `debtToken`. If the address of `debtToken` is `alUSD`, then we swap from `OP -> USDC -> alETH`. If the address of `debtToken` is `alETH`, then we swap from `OP -> WETH -> alETH`. 

Prior to swapping, the slippage `minAmountOut` is calculated. This is done via the following function:

`OptimismRewardCollector::getExpectedExchange`
```javascript
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

As we can see in `RewardRouter::distributeRewards`, the `expectedExchange` will be used as the `minimumAmountOut` parameter within the `OptimismRewardCollector::claimAndDonateRewards` function once it's multiplied by `slippageBPS / BPS`, which is hardcoded to 9500 and 10000, respectively.

`OptimismRewardCollector::claimAndDonateRewards`
```javascript
    function claimAndDonateRewards(address token, uint256 minimumAmountOut) external returns (uint256) {
        .
        .
        .
        // @audit `alUsdOptimism` address
        if (debtToken == 0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A) {
            // Velodrome Swap Routes: OP -> USDC -> alUSD
            .
            .
            .
            IVelodromeSwapRouter(swapRouter).swapExactTokensForTokens(amountRewardToken, minimumAmountOut, routes, address(this), block.timestamp);
        // @audit `alEthOptimism` address
        } else if (debtToken == 0x3E29D3A9316dAB217754d13b28646B76607c5f04) {
            // Velodrome Swap Routes: OP -> alETH
            .
            .
            .
            IVelodromeSwapRouter(swapRouter).swapExactTokensForTokens(amountRewardToken, minimumAmountOut, routes, address(this), block.timestamp);
        } else {
            revert IllegalState("Reward collector `debtToken` is not supported");
        }

        // Donate to alchemist depositors
        uint256 debtReturned = IERC20(debtToken).balanceOf(address(this));
        TokenUtils.safeApprove(debtToken, alchemist, debtReturned);
        IAlchemistV2(alchemist).donate(token, debtReturned);
    }
```
Now that we know the flow of how the reward collection and distribution works, let's delve into the problem here.

The problem is in `OptimismRewardCollector::getExpectedExchange`, how `opToUsd` and `ethToUsd` chainlink oracle returns different USD decimals. `opToUsd` returns 8 decimals, while `ethToUsd` returns 11 decimals. 

```javascript
    if (debtToken == alUsdOptimism) {
        expectedExchange = totalToSwap * uint(opToUsd) / 1e8;
    }
```

If `debtToken` is `alUsdOptimism`, we know that we will be performing the swap to get `alUSD`, which has [18 decimals](https://optimistic.etherscan.io/token/0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A). Reward token is in `OP`, which also has 18 decimals. `opToUsd` will have 8 decimals since that is what `opToUsdOracle` returns. Source: 
1. https://data.chain.link/feeds/optimism/mainnet/op-usd
2. https://optimistic.etherscan.io/address/0x0D276FC14719f9292D5C1eA2198673d1f4269246#readContract (check `latestAnswer`). 

We know `expectedExchange` will be used for `minAmountOut` slippage for `alUSD`, therefore it must have 18 decimals, so the division by 1e8 is correct. This calculation is CORRECT.

Now lets look at if `debtToken` is `alEthOptimism`:
```javascript
else if (debtToken == alEthOptimism) {
            expectedExchange = totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd);
        }
```

We already know `totalToSwap` has 18 decimals and `opToUsd` has 8 decimals. In this case, we swap from OP -> alETH, which has [18 decimals](https://optimistic.etherscan.io/token/0x3e29d3a9316dab217754d13b28646b76607c5f04). However, `ethToUsdOracle` returns 11 decimals for the value of `ethToUsd`. Source: 
1. https://data.chain.link/feeds/optimism/mainnet/eth-usd
2. https://optimistic.etherscan.io/address/0x13e3Ee699D1909E989722E753853AE30b17e08c5#readContract (check under `latestAnswer`). 

This will give us `1e18` * `1e8` / `1e11` = `1e15` decimals which is far less than the 18 decimals of `alETH`. 

Slippage is clearly inadequate and is susceptible to sandwich attacks causing loss of funds.


## Proof of Concept
Add the following test file `slippageCalculation.t.sol` to `v2-foundry/src/test`

```javascript
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";

contract slippageCalculation {

    uint256 public constant BPS = 10000;
    uint256 public slippageBPS = 9500;

function testSlippage() public{
    
    // Arbitrary value of reward token to swap (18 decimals)
    uint256 totalToSwap = 5e18;
    
    // This is the latest answer returned by opToUsdOracle (8 decimals)
    uint256 opToUsd = 222698506;
    
    // This is the latest answer returned by ethToUsdOracle (11 decimals)
    uint256 ethToUsd = 306127085166;

    uint256 expectedExchange = totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd);
    uint256 minimumAmountOut = expectedExchange * slippageBPS / BPS;
    console.log("Slippage value", minimumAmountOut);
    
    // alETH returns 18 decimals, but here we can see that minimumAmountOut is less than 16 decimals, far less than the expected decimals
    assert(minimumAmountOut < 1e16);
}

}

```
Run: `forge test --mt testSlippage -vv`

Output:

```solidity
[PASS] testSlippage() (gas: 5773)
Logs:
  Slippage value 3455486151858758

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 387.00µs (161.30µs CPU time)
```

## Impact
Variety of attack vectors due to inadequate slippage, such as sandwich attacks, causing loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L25-L56

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L135-L137

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L65-L78


## Tool used
Manual Review

## Recommendation
One possible solution is to make the following changes to `OptimismRewardCollector::getExpectedExchange`:

```diff
    else if (debtToken == alEthOptimism) {
-       expectedExchange = totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd);
+       expectedExchange = totalToSwap * uint(uint(opToUsd)) * 1e3 / uint(ethToUsd);
    }
```
Or the protocol can utilize other chainlink functions to get the decimals, in case it changes to a different number.