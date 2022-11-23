
# Summary
This report presents the results of the audit with Collaborator Foundation conducted between 21/11/2022 and 23/11/2022.

## Scope

- `contracts/Collaborator.sol`
- `contracts/ICollaborator.sol`
- `contracts/Distributor.sol`

This project repository is available on [Github](https://github.com/Codys12/CollabPublic), and all contracts were audited on the following commit hash: `fe38dd0a59f107ccf4641a2183fa49d8b241b52a`.

# Audit Results

## [C] Critical

### [C1] IERC20 transfer result is ignored

Depending on the implementation, `IERC20.transfer` is not guaranteed to revert if the operation fails. In case this happens, users won't receive any tokens, and their slot will be marked as claimed, even if the transfer has failed.

**contracts/Collaborator.sol:L20-L26**
```
function claim(uint256 slot, address recipient) public{
    
    require(!hasBeenClaimed(slot), "slot has already been claimed");
    require(msg.sender == deployer);
    claimed[slot] = true;
    IERC20(tokenAddress).transfer(recipient, 1000 ether);
}
```

**Recommendation**

Check `IERC20.transfer` result and revert the transaction if the result is not `true`.

## [H] High Risk

### [H1] `newTargetAddress` could be set to zero address

**contracts/Collaborator.sol:L141-L145**
```
function _setTargetAddress(address newTargetAddress) internal {
    emit TargetAddressSet(_targetAddress, newTargetAddress);
    _targetAddress = newTargetAddress;
}
```
There are no checks for `newTargetAddress` that could be set as `address(0)` by mistake. This address is then used inside the `sendTokens()` function, where new tokens are minted.

**contracts/Collaborator.sol:L109-L117**
```
function sendTokens() public{
    require(block.number - _lastMinted > 0, "Must be at least one block between mints");
    require(ICOFinalized);
    uint256 numBlocks = block.number - _lastMinted;
    _lastMinted = block.number;

    _mint(_targetAddress, mintRate() * numBlocks);

}
```

If `_targetAddress` is set to `address(0)` then `sendTokens()` would act as a burn function, which is not the intended use.

**Recommendation**

Add a check inside `_setTargetAddress` that requires a non-zero address.

### [H2] `transferOwnership` could be set to zero address

There are no checks for `newTargetAddress` that could be set as `address(0)` by mistake. In this way there is no longer an owner, and `onlyOwner` functions cannot longer be used.

**contracts/Collaborator.sol:L193-L195**
```
function transferOwnership(address newOwner) public override ownershipModifier {
    super.transferOwnership(newOwner);
}
```

**Recommendation**

Add an additional check inside `transferOwnership` requiring a non-zero address.

## [M] Medium Risk 

### [M1] Division by zero

If both `x` and `y` are zero the transaction will fail.

**contracts/Collaborator.sol:L48-L53**
```
function percentDifference(uint256 x, uint256 y) pure public returns(uint256){
    uint256 numerator = x>y ? (x-y) : (y-x);
    uint256 denominator = x>y ? y : x;
    numerator *= 1e20; //Convert to percent and allow for division
    return numerator / denominator;
}
```

**Recommendation**

Either return 0 if this case occurs or revert the transaction with a custom error. 

## [L] Low Risk

### [L1] Different compiler versions between contracts

Contracts allow the use of an older compiler version, which differs from the version used to compile the project: `0.8.9`

**contracts/Collaborator.sol:L2**
**contracts/ICollaborator.sol:L2**
**contracts/Distribute.sol:L2**

```
pragma solidity ^0.8.4;
```

**Recommendation**

Use a fixed compiler version for all contracts:
`pragma solidity 0.8.9;`

### [L2] Use the `constant` keyword for constants to save gas

**contracts/Collaborator.sol:L12-L13**
```
uint256 BLOCK_TIME = 12000; //12 second block time
uint256 public INITIAL_COST_MULTIPLIER = 1 ether;
```

**contracts/Collaborator.sol:L15**
```
uint256 public ICOLength = (1000 * 60 * 20) / BLOCK_TIME; // 20min //1249920; //The length of the ICO in blocks (~1 month) using average block time of 15s CHANGE THIS BACK AFTER TESTING
```

**contracts/Distribute.sol:L9**
```
address public tokenAddress = 0x978867e9aC04688f7AbB70ffF3dd48e7ea8Bd4Be;
```

### [L3] Invert logic operators to save gas

**contracts/Collaborator.sol:L56**
```
require(block.number - deploymentBlock <= ICOLength);
```

**contracts/Collaborator.sol:L93**
```
require(block.number - deploymentBlock >= ICOLength, "ICO period not passed");
```

**contracts/Collaborator.sol:L102**
```
require(balanceOf(msg.sender) >= amount, "amount exceeds balance");
```

**contracts/Collaborator.sol:L122**
```
require(block.number - _lastChangedMintRate >= changeTime(), "Not enough time has passed since this parameter was last changed");
    
```

**contracts/Collaborator.sol:L129**
```
require(block.number - _lastChangedMaxChange >= changeTime(), "Not enough time has passed since this parameter was last changed");

```

**contracts/Collaborator.sol:L136**
```
require(block.number - _lastChangedChangeTime >= changeTime(), "Not enough time has passed since this parameter was last changed");

```

**contracts/Collaborator.sol:L142**
```
require(block.number - _lastChangedTargetAddress >= changeTime(), "Not enough time has passed since this parameter was last changed");

```       

**Recommendation**

Instead of using `>=`, use `<` (same concept for `<=` and `>`) and invert the order of the operators to save 3 gas/call.

### [L4] Cache result of `totalSupply() to save gas

**contracts/Collaborator.sol:L114-L115**
```
uint256 amountDue = (address(this).balance * amount) / totalSupply();
_setMintRate((mintRate() * (totalSupply() - amount)) / totalSupply());
```

**Recommendation**

Cache `totalSupply()` to save an average of ~200 gas/call.

```
uint256 ts = totalSupply();
uint256 amountDue = (address(this).balance * amount) / ts;
_setMintRate((mintRate() * (ts - amount)) / ts);
```