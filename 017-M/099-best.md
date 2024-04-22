Sweet Hazel Weasel

medium

# Discrepancies between the interface files and the actual implementation

## Summary

There are discrepancies between the interface files and the actual implementation. Integrators referencing the interface file will expect the success status (boolean) to be returned. Since no success status is returned, their contracts will always assume that the burning of tokens has failed, leading to various issues.

## Vulnerability Detail

Per the interface file, the `burnSelf` and `burn` functions are expected to return a boolean indicating whether the burning of tokens was successful.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/interfaces/IERC20Burnable.sol#L7

```solidity
File: IERC20Burnable.sol
05: /// @title  IERC20Burnable
06: /// @author Alchemix Finance
07: interface IERC20Burnable is IERC20 {
08:     /// @notice Burns `amount` tokens from the balance of `msg.sender`.
09:     ///
10:     /// @param amount The amount of tokens to burn.
11:     ///
12:     /// @return If burning the tokens was successful.
13:     function burnSelf(uint256 amount) external returns (bool);
14: 
15:     /// @notice Burns `amount` tokens from `owner`'s balance.
16:     ///
17:     /// @param owner  The address to burn tokens from.
18:     /// @param amount The amount of tokens to burn.
19:     ///
20:     /// @return If burning the tokens was successful.
21:     function burn(address owner, uint256 amount) external returns (bool);
22: }
```

The protocol expects that the alAsset's `burn` and `burnSelf` functions comply with the `IERC20Burnable` interface. Thus, in Lines 135 and 152 below, the low-level calls within the `TokenUtils` contract are made using the function selectors of the `IERC20Burnable` interface.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/libraries/TokenUtils.sol#L133

```solidity
File: TokenUtils.sol
127:     /// @dev Burns tokens.
128:     ///
129:     /// Reverts with a `CallFailed` error if execution of the burn fails or returns an unexpected value.
130:     ///
131:     /// @param token  The token to burn.
132:     /// @param amount The amount of tokens to burn.
133:     function safeBurn(address token, uint256 amount) internal {
134:         (bool success, bytes memory data) = token.call(
135:             abi.encodeWithSelector(IERC20Burnable.burnSelf.selector, amount)
136:         );
137: 
138:         if (token.code.length == 0 || !success || (data.length != 0 && !abi.decode(data, (bool)))) {
139:             revert ERC20CallFailed(token, success, data);
140:         }
141:     }
142: 
143:     /// @dev Burns tokens from its total supply.
144:     ///
145:     /// @dev Reverts with a {CallFailed} error if execution of the burn fails or returns an unexpected value.
146:     ///
147:     /// @param token  The token to burn.
148:     /// @param owner  The owner of the tokens.
149:     /// @param amount The amount of tokens to burn.
150:     function safeBurnFrom(address token, address owner, uint256 amount) internal {
151:         (bool success, bytes memory data) = token.call(
152:             abi.encodeWithSelector(IERC20Burnable.burn.selector, owner, amount)
153:         );
154: 
155:         if (token.code.length == 0 || !success || (data.length != 0 && !abi.decode(data, (bool)))) {
156:             revert ERC20CallFailed(token, success, data);
157:         }
158:     }

```

However, it was found that the function declaration of the `AlchemicTokenV2Base.burnSelf` and `AlchemicTokenV2Base.burn` functions do not return a boolean.

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L175

```solidity
File: AlchemicTokenV2Base.sol
175:   function burnSelf(uint256 amount) external {
..SNIP..
190:   function burn(address account, uint256 amount) external {
```

## Impact

There are discrepancies between the interface files and the actual implementation. Integrators referencing the interface file will expect the success status (boolean) to be returned. Since no success status is returned, their contracts will always assume that the burning of tokens has failed, leading to various issues.

## Code Snippet

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/libraries/TokenUtils.sol#L133

https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/AlchemicTokenV2Base.sol#L175

## Tool used

Manual Review

## Recommendation

Update the `IERC20Burnable` interface file to be aligned with the actual implementation.