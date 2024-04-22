Decent Mulberry Poodle

high

# Inconsistency in Exchange Rate Due to Oracle Update Timing Can Lead to Read-Only Reentrancy

## Summary

In the RewardRouter and OptimismRewardCollector, the getExpectedExchange function retrieves data from a Chainlink oracle to determine the exchange rate for tokens. This function itself does not modify any state but fetches the current market rate which is crucial for deciding how many tokens to swap and at what rate.

If the oracle data changes between the fetching of data in getExpectedExchange and its use in distributeRewards (which calls claimAndDonateRewards), the operations may proceed with outdated or incorrect data. This can happen if, for example, there are rapid price fluctuations in the market that aren't immediately reflected in the oracle update received by the contract.

The outdated or incorrect data can lead to suboptimal financial decisions. This result from the dependency on external, mutable data sources (the oracles) which can lead to inconsistencies within a single transaction due to the timing of data updates.

## Vulnerability Detail

The [getExpectedExchange](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L91) function in [OptimismRewardCollector](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol) uses price data from Chainlink oracles to calculate the expected amount of tokens that should be received in exchange.

This function is critical in the calculation performed in the [distributeRewards](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L35) function of the [RewardRouter](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol).

If there is a significant delay or rapid update in the oracle price between successive calls to getExpectedExchange during a transaction, it could result in decisions being made on stale data.

## Impact

The use of potentially stale data due to rapid price fluctuations and oracle update frequencies can lead to incorrect token distributions and financial losses.

Suppose RewardRouter initiates a reward distribution.
- getExpectedExchange is called, retrieving the current price of OP to USD at $10 per OP.
- A rapid market shift occurs, and the price updates to $12 per OP, but this update happens right after the call.
- The contract then proceeds to swap tokens based on the outdated $10 pricing, leading to less USD received than expected, affecting the amount finally donated.

## Code Snippet

OptimismRewardCollector
https://github.com/sherlock-audit/2024-04-alchemix/blob/32e4902f77b05ea856cf52617c55c3450507281c/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L91
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

RewardRouter
https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/RewardRouter.sol#L35
```solidity
/// @dev Distributes grant rewards and triggers reward collector to claim and donate
    function distributeRewards(address vault) external returns (uint256) {
        // If vault is set to receive rewards from grants, send amount to reward collector to donate
        if (rewards[vault].rewardAmount > 0) {
            // Calculates ratio of timeframe to time since last harvest
            // Uses this ratio to determine partial reward amount or extra reward amount
            uint256 blocksSinceLastReward = block.number - rewards[vault].lastRewardBlock;
            uint256 maxReward = rewards[vault].rewardAmount - rewards[vault].rewardPaid;
            uint256 currentReward = rewards[vault].rewardAmount * blocksSinceLastReward / rewards[vault].rewardTimeframe;
            uint256 amountToSend = currentReward > maxReward ? maxReward : currentReward;

            TokenUtils.safeTransfer(IRewardCollector(rewards[vault].rewardCollectorAddress).rewardToken(), rewards[vault].rewardCollectorAddress, amountToSend);
            rewards[vault].lastRewardBlock = block.number;
            rewards[vault].rewardPaid += amountToSend;

            if (rewards[vault].rewardPaid == rewards[vault].rewardAmount) {
                rewards[vault].rewardAmount = 0;
                rewards[vault].rewardPaid = 0;
            }
        }

        return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);
    }
```

## Tool used

Manual Review

## Recommendation

Implement reentrancy guards or non-reentrant modifiers for these functions that interact with external contracts and depend on external data. This would prevent any form of reentrancy – direct or indirect – during critical operations.
