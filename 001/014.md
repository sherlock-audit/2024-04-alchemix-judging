Slow Brown Coyote

high

# No check for Arbitrum/Optimisim sequencer down in Chainlink feeds

## Summary
When utilizing Chainlink in L2 chains like Arbitrum, it's important to ensure that the prices provided are not falsely perceived as fresh, even when the sequencer is down. This vulnerability could potentially be exploited by malicious actors to gain an unfair advantage.

## Vulnerability Detail
If it updates again within the update threshold. The feeds typically can update several times within a threshold period if the price is moving a lot when the sequencer is down, the new price won't be reported to the chain. the feed on the L2 will return the value it had when it went down
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
```
This causes the `RewardRouter::distributeRewards` to donate undervalued value of rewards.
## Impact
Potentially be exploited by malicious actors to gain an unfair advantage,k causing protocol to donate the undervalued rewards.
## Code Snippet
https://github.com/sherlock-audit/2024-04-alchemix/blob/32e4902f77b05ea856cf52617c55c3450507281c/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L97C1-L130C10

## Tool used

Manual Review

## Recommendation
code example of Chainlink: https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code