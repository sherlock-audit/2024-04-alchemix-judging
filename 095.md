Sweet Hazel Weasel

medium

# User's alAssets might be burned by malicious users

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