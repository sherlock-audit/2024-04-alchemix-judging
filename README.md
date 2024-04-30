# Issue H-1: The calculated value for slippage protection in the protocol is inaccurate 

Source: https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/5 

## Found by 
Bauer
## Summary


The protocol calculates the slippage protection value based on the price of OP relative to USD and OP relative to ETH, while the intended exchange is for alUSD and alETH. This results in inaccuracies in the calculated slippage protection value.

## Vulnerability Detail

In the `RewardRouter.distributeRewards()` function, the protocol first sends the OP rewards to the OptimismRewardCollector contract, 
```solidity
         TokenUtils.safeTransfer(IRewardCollector(rewards[vault].rewardCollectorAddress).rewardToken(), rewards[vault].rewardCollectorAddress, amountToSend);
            rewards[vault].lastRewardBlock = block.number;
            rewards[vault].rewardPaid += amountToSend;

```


then calls the `RewardCollector.claimAndDonateRewards()` function to convert OP into alUSD or alETH. 
```solidity
        return IRewardCollector(rewards[vault].rewardCollectorAddress).claimAndDonateRewards(vault, IRewardCollector(rewards[vault].rewardCollectorAddress).getExpectedExchange(vault) * slippageBPS / BPS);


```

During the conversion process, there is a parameter for slippage protection, which is calculated using `OptimismRewardCollector.getExpectedExchange() * slippageBPS / BPS`. Let's take a look at the `getExpectedExchange()` function. In this function, the protocol retrieves the prices of optoUSD and optoETH from Chainlink.

```solidity
        (
            uint80 roundID,
            int256 opToUsd,
            ,
            uint256 updateTime,
            uint80 answeredInRound
        ) = IChainlinkOracle(opToUsdOracle).latestRoundData();

```
```solidity
    // Ensure that round is complete, otherwise price is stale.
        (
            uint80 roundIDEth,
            int256 ethToUsd,
            ,
            uint256 updateTimeEth,
            uint80 answeredInRoundEth
        ) = IChainlinkOracle(ethToUsdOracle).latestRoundData();

```

 If `debtToken == alUsdOptimism`, the expectedExchange for slippage protection is calculated as totalToSwap * uint(opToUsd) / 1e8. 
```solidity
     // Find expected amount out before calling harvest
        if (debtToken == alUsdOptimism) {
            expectedExchange = totalToSwap * uint(opToUsd) / 1e8;

```

If debtToken == alEthOptimism, the expectedExchange for slippage protection is calculated as totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd). 
```solidity
  else if (debtToken == alEthOptimism) {
            expectedExchange = totalToSwap * uint(uint(opToUsd)) / uint(ethToUsd);

```

Here, we observe that the `expectedExchange` is calculated based on the value of OP relative to USD and OP relative to ETH, while the protocol intends to exchange for alUSD and alETH. 
```solidity
 if (debtToken == 0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A) {
            // Velodrome Swap Routes: OP -> USDC -> alUSD
            IVelodromeSwapRouter.route[] memory routes = new IVelodromeSwapRouter.route[](2);
            routes[0] = IVelodromeSwapRouter.route(0x4200000000000000000000000000000000000042, 0x7F5c764cBc14f9669B88837ca1490cCa17c31607, false);
            routes[1] = IVelodromeSwapRouter.route(0x7F5c764cBc14f9669B88837ca1490cCa17c31607, 0xCB8FA9a76b8e203D8C3797bF438d8FB81Ea3326A, true);
            TokenUtils.safeApprove(rewardToken, swapRouter, amountRewardToken);
            IVelodromeSwapRouter(swapRouter).swapExactTokensForTokens(amountRewardToken, minimumAmountOut, routes, address(this), block.timestamp);
        } else if (debtToken == 0x3E29D3A9316dAB217754d13b28646B76607c5f04) {
            // Velodrome Swap Routes: OP -> alETH
            IVelodromeSwapRouter.route[] memory routes = new IVelodromeSwapRouter.route[](1);
            routes[0] = IVelodromeSwapRouter.route(0x4200000000000000000000000000000000000042, 0x3E29D3A9316dAB217754d13b28646B76607c5f04, false);
            TokenUtils.safeApprove(rewardToken, swapRouter, amountRewardToken);
            IVelodromeSwapRouter(swapRouter).swapExactTokensForTokens(amountRewardToken, minimumAmountOut, routes, address(this), block.timestamp);
        } 

```

However, the price of alUSD is not equivalent to USD, and the price of alETH is not equivalent to ETH. This discrepancy leads to inaccuracies in the calculated value for slippage protection, making the protocol vulnerable to sandwich attacks.

## Impact
The protocol is susceptible to sandwich attacks.


## Code Snippet
https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/utils/collectors/OptimismRewardCollector.sol#L120-L126

## Tool used

Manual Review

## Recommendation
Calculate using the correct prices.




# Issue H-2: No check for Arbitrum/Optimisim sequencer down in Chainlink feeds 

Source: https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/14 

## Found by 
0xMaroutis, boredpukar, dimah7, eeshenggoh, nilay27, no, nuthan2x, xiaoming90
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

# Issue M-1: Tokens are burnt from flashloan borrower even if it didn't approve it 

Source: https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/32 

The protocol has acknowledged this issue.

## Found by 
Bigsam, Dudex\_2004, ZanyBonzy, m4ttm, zigtur
## Summary

During flashloans, tokens are burnt from the flashloan borrower without its approval.

That doesn't comply with EIP-3156 and may lead to loss of funds for the receiver.

## Vulnerability Detail

The `AlchemicTokenV2Base` contract allows flash loans through the [EIP-3156 standard](https://eips.ethereum.org/EIPS/eip-3156).

The loan amount and some fees are burnt from the loan borrower (`receiver`).
However, according to EIP-3156:
> For the transaction to not revert, `receiver` MUST approve `amount + fee` of `token` to be taken by `msg.sender` before the end of `onFlashLoan`.

In the current implementation, the funds will be taken from `receiver` and burnt even if the `receiver` didn't approve it.

A flashloan receiver that relies on the approval mechanism to deny flashloans will not
be able to deny `AlchemicTokenV2` flashloans.

As a flashloan receiver may be unable to deny flashloans and fees are taken from it, the receiver's balance could be drained.

## Impact

Loss of funds of specific flashloan receivers.

## Code Snippet

The [`AlchemicTokenV2.flashLoan` function does not check](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L247-L272) the `receiver` allowance once the `onFlashLoan` call returns. It directly burn the funds from the receiver.

```solidity
  function flashLoan(
    IERC3156FlashBorrower receiver,
    address token,
    uint256 amount,
    bytes calldata data
  ) external override nonReentrant returns (bool) {
    if (token != address(this)) {
      revert IllegalArgument();
    }

    if (amount > maxFlashLoan(token)) {
      revert IllegalArgument();
    }

    uint256 fee = flashFee(token, amount);

    _mint(address(receiver), amount);

    if (receiver.onFlashLoan(msg.sender, token, amount, fee, data) != CALLBACK_SUCCESS) {
      revert IllegalState();
    }

    _burn(address(receiver), amount + fee); // Will throw error if not enough to burn

    return true;
  }
```

The `ERC20Upgradeable._burn` function doesn't verify the approval neither:
```solidity
    function _burn(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: burn from the zero address");

        _beforeTokenTransfer(account, address(0), amount);

        uint256 accountBalance = _balances[account];
        require(accountBalance >= amount, "ERC20: burn amount exceeds balance");
        unchecked {
            _balances[account] = accountBalance - amount;
            // Overflow not possible: amount <= accountBalance <= totalSupply.
            _totalSupply -= amount;
        }

        emit Transfer(account, address(0), amount);

        _afterTokenTransfer(account, address(0), amount);
    }
```


## Proof of Concept

The following flashloan receiver will be vulnerable to such attack.

```solidity
import "../../lib/openzeppelin-contracts/contracts/access/Ownable.sol";
import "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC3156FlashBorrower} from "../../lib/openzeppelin-contracts/contracts/interfaces/IERC3156FlashBorrower.sol";

contract VulnerableFlashloanReceiver is IERC3156FlashBorrower, Ownable {

    constructor() Ownable() {}

    // @POC: Only owner can approve interactions with lenders, as a flashloan should revert when approval is 0
    function approveFlashloan(address flashlender, address token, uint256 amount) external onlyOwner() {
        IERC20(token).approve(flashlender, amount);
    }

    function onFlashLoan(address initiator, address token, uint256 amount, uint256 fee, bytes calldata data) external override returns(bytes32) {
        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }
}
```

As shown previously, the `AlchemicTokenV2.flashLoan` will not revert after `VulnerableFlashloanReceiver.onFlashLoan` if the contract owns enough tokens.
It will burn tokens from this contract without its approval.


## Tool used

Manual Review

## Recommendation

Consider adding an approval consumption after the `onFlashLoan` call and before the `_burn` call.

The following patch implements such a fix:

```diff
diff --git a/v2-foundry/src/AlchemicTokenV2Base.sol b/v2-foundry/src/AlchemicTokenV2Base.sol
index 395014a..ba75fad 100644
--- a/v2-foundry/src/AlchemicTokenV2Base.sol
+++ b/v2-foundry/src/AlchemicTokenV2Base.sol
@@ -266,6 +266,9 @@ contract AlchemicTokenV2Base is ERC20Upgradeable, AccessControlUpgradeable, IERC
       revert IllegalState();
     }
 
+    uint256 newAllowance = allowance(msg.sender, address(this)) - (amount + fee); // Will revert if not enough approval
+    _approve(msg.sender, address(this), newAllowance);
+
     _burn(address(receiver), amount + fee); // Will throw error if not enough to burn
 
     return true;
```

*Note: The patch can be applied through `git apply`.*



## Discussion

**Hesnicewithit**

It is the responsibility of the developer of the receiver to implement a check that the loan initiator is trusted.

 From EIP
"Any receiver that keeps an approval for a given lender needs to include in onFlashLoan a mechanism to verify that the initiator is trusted."

Also we do not charge a fee in the first place. If you check the existing contracts you can see it is set to 0 and always will be

**Hash01011122**

Agreed what you are mentioning here is accurate, also in README file of Alchemix it isn't mentioned that protocol doesn't comply to any EIPs.
Still I think this issue should is borderline low/medium issue. 
I am tending to consider it as medium severity, as it clearly shows accurate attack path. Also argument of developer error can't be used in this case as user didn't approve for any transaction in the first place.

# Issue M-2: User's alAssets might be burned by malicious users 

Source: https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/95 

The protocol has acknowledged this issue.

## Found by 
xiaoming90
## Summary

A malicious user could trigger a flash loan on behalf of another user, resulting in the victim's alAssets being burned as a fee.

## Vulnerability Detail

Assume Bob's account (smart contract) implements the `onFlashLoan` function and owns 100 alETH tokens. Alice could call `AlchemicTokenV2Base.flashLoan` function on behalf of Bob by setting the `receiver` as Bob. This would cause a `fee` amount of alETH tokens to be burnaed from Bob's account at Line 269, even if Bob did not trigger the flash loan. Alice could repeat this until all of the alETH tokens in Bob's account are burned.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L269

```solidity
File: AlchemicTokenV2Base.sol
247:   function flashLoan(
248:     IERC3156FlashBorrower receiver,
249:     address token,
250:     uint256 amount,
251:     bytes calldata data
252:   ) external override nonReentrant returns (bool) {
253:     if (token != address(this)) {
254:       revert IllegalArgument();
255:     }
256: 
257:     if (amount > maxFlashLoan(token)) {
258:       revert IllegalArgument();
259:     }
260: 
261:     uint256 fee = flashFee(token, amount);
262: 
263:     _mint(address(receiver), amount);
264: 
265:     if (receiver.onFlashLoan(msg.sender, token, amount, fee, data) != CALLBACK_SUCCESS) {
266:       revert IllegalState();
267:     }
268: 
269:     _burn(address(receiver), amount + fee); // Will throw error if not enough to burn
270: 
271:     return true;
272:   }
```

This issue will happen as long as one of the following conditions is true:

1. Bob's `onFlashloan` function does not have access control (does not verify the `msg.sender`); OR
2. Bob's `onFlashLoan` function has access control but always returns `CALLBACK_SUCCESS` at the end of the function call, regardless of the operation's status.

## Impact

Victim's alAssets might be burned by malicious users.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L269

## Tool used

Manual Review

## Recommendation

Consider not allowing the user to trigger a flash loan on behalf of another user to mitigate this issue unless there is a specific use case for this.



## Discussion

**Hesnicewithit**

It is up to the developer of the receiver to check the initiator address before doing anything

**Hash01011122**

I think this issue suffice medium severity as mentioned in the #32 issue developer error can't be used as argument to invalidate issue as Watson's were not aware of it at the time of audit.

# Issue M-3: `block.timestamp` should be used instead of `block.number` 

Source: https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/100 

## Found by 
0xMaroutis, Hajime, SBSecurity, T1MOH, boredpukar, cryptic, xiaoming90
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



## Discussion

**Hesnicewithit**

This is a good improvement for when we deploy on new L2s. Will implement this change 

# Issue M-4: Burn limit of a bridge can be bypassed 

Source: https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/102 

The protocol has acknowledged this issue.

## Found by 
0xAadi, AhmedAdam, Kirkeelee, blackhole, no, xiaoming90
## Summary

The burn limit of a bridge can be bypassed.

## Vulnerability Detail

The purpose of having the burn limit (`xBridges[msg.sender].burnerParams.maxLimit`) for the bridge is to restrict the number of tokens it can burn over a fixed period of time. If a bridge calls the following `burn` function, the code from Line 197 to Line 200 will ensure that the burn limit is enforced so that the bridge cannot burn an unlimited amount of tokens.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L197

```solidity
File: AlchemicTokenV2Base.sol
190:   function burn(address account, uint256 amount) external {
191:     if (msg.sender != account) {
192:       uint256 newAllowance = allowance(account, msg.sender) - amount;
193:       _approve(account, msg.sender, newAllowance);
194:     }
195: 
196:     // If bridge is registered check limits and update accordingly.
197:     if (xBridges[msg.sender].burnerParams.maxLimit > 0) {
198:       uint256 currentLimit = burningCurrentLimitOf(msg.sender);
199:       if (amount > currentLimit) revert IXERC20.IXERC20_NotHighEnoughLimits();
200:       _useBurnerLimits(msg.sender, amount);
201:     }
202: 
203:     _burn(account, amount);
204:   }
```

However, this burn limit is ineffective or useless due to how the `burn` function is implemented.

Assume that the bridge holds 1M xalETH tokens and the bridge's max burn limit is only 10,000 xalETH per day. When the bridge attempts to burn all 1M xalETH tokens in one go over a short period of time by calling `AlchemicTokenV2Base.burn` function, it will not be able to do so, as the validation check within the `burn` function will revert. However, the bridge can simply bypass this restriction by transferring the 1M xalETH tokens to another address that is not marked as a bridge and using this address to burn the 1M xalETH to achieve the same outcome.

The same issue applied for the [`AlchemicTokenV2Base.burnSelf`](https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L177) function.

## Impact

The intention of the max burn limit is to restrict the number of assets that a bridge can burn so as to contain the negative impact in the event the bridge is compromised. However, as shown in the previous section, this limit does not work and can be bypassed.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L177

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L197

## Tool used

Manual Review

## Recommendation

Consider only allowing whitelisted addresses to call the `burn` function. If this is not possible due to the technical requirement of the Alchemix core, note the limitation of the xERC20's burn limit within Alchemix and do not rely on it as a security control for risk management, as this can be bypassed easily.



## Discussion

**Hesnicewithit**

Incorrect assumption about the bridge. Bridge does not hold these tokens. The user holds them and the bridge burns on their behalf based on an allowance system.

# Issue M-5: Protocol vulnerable to donation attacks 

Source: https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/106 

The protocol has acknowledged this issue.

## Found by 
Varun\_05, xiaoming90
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



## Discussion

**Hesnicewithit**

How is this an attack? If someone wants to give away free tokens to the users this is fine. If you look at the whitelist only function you will see that this is simply a way to stop smart contracts from interacting, unless they are specific contracts we approve. Any EOA address can call donate on the alchemist contract. 

