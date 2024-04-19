Proper Taffy Boar

medium

# Tokens are burnt from flashloan borrower even if it didn't approve it

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