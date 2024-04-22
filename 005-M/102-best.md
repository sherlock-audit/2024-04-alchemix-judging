Sweet Hazel Weasel

medium

# Burn limit of a bridge can be bypassed

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