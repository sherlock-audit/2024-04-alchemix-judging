Agreeable Gunmetal Lark

medium

# Wrong amount of totalMinted[bridgeTokenAddress] is written when a user is not fee exempted.

## Summary
When a user is not fee exempted then the total amount of tokens minted to him are decreased by applying the fess due to this totalMinted[bridgeTokenAddress] are increased by wrong amount.

## Vulnerability Detail
Following is the exchangeOldForCanonical function 
```solidity
function exchangeOldForCanonical(address bridgeTokenAddress, uint256 tokenAmount) external nonReentrant validBridgeToken(bridgeTokenAddress) returns (uint256 canonicalTokensOut) {
        if (exchangesPaused) {
            revert IllegalState();
        }

        if (!bridgeTokenEnabled[bridgeTokenAddress]) {
            revert IllegalState();
        }

        // Check mint caps and adjust mint count
        uint256 total = tokenAmount + totalMinted[bridgeTokenAddress];
        if (total > mintCeiling[bridgeTokenAddress]) {
            revert IllegalState();
        }
        totalMinted[bridgeTokenAddress] = total;

        // Pull in the old tokens
        TokenUtils.safeTransferFrom(bridgeTokenAddress, msg.sender, address(this), tokenAmount);

        // Handle the fee, if applicable
        canonicalTokensOut = tokenAmount;
        if (!_isFeeExempt(msg.sender)) {
            canonicalTokensOut -= ((canonicalTokensOut * swapFees[bridgeTokenAddress][0]) / FEE_PRECISION);
        }

        // Mint canonical tokens and give it to the sender
        super._mint(msg.sender, canonicalTokensOut);
    }
```
When a user is not fee exempted then following loc is also executed
```solidity
 if (!_isFeeExempt(msg.sender)) {
            canonicalTokensOut -= ((canonicalTokensOut * swapFees[bridgeTokenAddress][0]) / FEE_PRECISION);
        }
``` 
and then user are minted canonicalTokenOut amount of tokens . It can be seen canonicalTokensOut are reduced if fee is not exempted but totalMinted[bridgeTokenAddress] are updated as follows
```solidity
 uint256 total = tokenAmount + totalMinted[bridgeTokenAddress];  
        totalMinted[bridgeTokenAddress] = total;
```
It is clear when fee is not exempted total minted tokens are less than tokenAmount + totalMinted[bridgeTokenAddress] becaue canonicalTokenOut are minted which are less than tokenAmount but it updates the totalMinted[bridgeTokenAddress] by the tokenAmount which is wrong because canonicalTokenOut are minted not tokenAmount.


## Impact
It would cause less tokens to be minted even when more tokens can be minted which causes mintCeiling[bridgeTokenAddress] to never be reached.
## Code Snippet
https://github.com/sherlock-audit/2024-04-alchemix/blob/main/v2-foundry/src/CrossChainCanonicalBase.sol#L106
## Tool used

Manual Review

## Recommendation
Make the following changes to the code 
```solidity
function exchangeOldForCanonical(address bridgeTokenAddress, uint256 tokenAmount) external nonReentrant validBridgeToken(bridgeTokenAddress) returns (uint256 canonicalTokensOut) {
        if (exchangesPaused) {
            revert IllegalState();
        }

        if (!bridgeTokenEnabled[bridgeTokenAddress]) {
            revert IllegalState();
        }
        
        // Handle the fee, if applicable
        canonicalTokensOut = tokenAmount;
        if (!_isFeeExempt(msg.sender)) {
            canonicalTokensOut -= ((canonicalTokensOut * swapFees[bridgeTokenAddress][0]) / FEE_PRECISION);
        }


        // Check mint caps and adjust mint count
        uint256 total = canonicalTokensOut + totalMinted[bridgeTokenAddress];
        if (total > mintCeiling[bridgeTokenAddress]) {
            revert IllegalState();
        }
        totalMinted[bridgeTokenAddress] = total;

        // Pull in the old tokens
        TokenUtils.safeTransferFrom(bridgeTokenAddress, msg.sender, address(this), tokenAmount);

       
        // Mint canonical tokens and give it to the sender
        super._mint(msg.sender, canonicalTokensOut);
    }
```