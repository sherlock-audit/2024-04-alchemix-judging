Mean Mercurial Gibbon

medium

# exchange fee is locked in ````CrossChainCanonicalAlchemicTokenV2```` contract

## Summary
````CrossChainCanonicalAlchemicTokenV2```` contract charges fee while ````exchangeCanonicalForOld()```` and ````exchangeOldForCanonical()````, but there is no interface to withdraw those fees.

## Vulnerability Detail
As shown on L114 and L137, exchange fee is charged and kept in contract, but we can't see any interface to withdraw these fund.
```solidity
File: src\CrossChainCanonicalBase.sol
092:     function exchangeOldForCanonical(address bridgeTokenAddress, uint256 tokenAmount) external nonReentrant validBridgeToken(bridgeTokenAddress) returns (uint256 canonicalTokensOut) {
...
113:         if (!_isFeeExempt(msg.sender)) {
114:             canonicalTokensOut -= ((canonicalTokensOut * swapFees[bridgeTokenAddress][0]) / FEE_PRECISION);
115:         }
...
119:     }

File: src\CrossChainCanonicalBase.sol
122:     function exchangeCanonicalForOld(address bridgeTokenAddress, uint256 tokenAmount) external nonReentrant validBridgeToken(bridgeTokenAddress) returns (uint256 bridgeTokensOut) {
...
136:         if (!_isFeeExempt(msg.sender)) {
137:             bridgeTokensOut -= ((bridgeTokensOut * swapFees[bridgeTokenAddress][1]) / FEE_PRECISION);
138:         }
...
145:     }

```
## Impact
Fees are locked in contract

## Code Snippet
https://github.com/sherlock-audit/2024-04-alchemix/blob/32e4902f77b05ea856cf52617c55c3450507281c/v2-foundry/src/CrossChainCanonicalBase.sol#L114
https://github.com/sherlock-audit/2024-04-alchemix/blob/32e4902f77b05ea856cf52617c55c3450507281c/v2-foundry/src/CrossChainCanonicalBase.sol#L137

## Tool used

Manual Review

## Recommendation
Adding an interface for withdrawing fee
