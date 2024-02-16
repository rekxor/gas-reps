
# Gas Report : AI Arena Contest | 9<sup>th</sup> Feb 2024 - 21<sup>st</sup> Feb 2024

## Executive summary
 **AI Arena** is a PvP platform fighting game where the fighters are AIs that were trained by humans.

## Overview

|              		|                                                         |
| --------------------- | ------------------------------------------------------- |
| **Project Name**	| **AI Arena**                                            |
| **Repository**   	| [Github](https://github.com/code-423n4/2024-02-ai-arena)|
| **Website**      	| [AI Arena](https://aiarena.io/#/)                       |
| **X(Twitter)**   	| [X](https://twitter.com/aiarena_)             	  |
| **Discord**      	| [Discord](https://discord.gg/aiarena)                   |
| **Total SLoC**  	| **1197** over **8** contracts                           |

---

## List of Findings 

| ID              | Title                                                                                                
| --------------- | ---------------------------------------------------------------------------------------------------- |  
| [G-01](#g-01-check-for-amount--0-before-making-_burn-call) | Unspecified Compiler version pragma                              |  
| [G-02](#g-02-cache-attributeslength-only-once-to-avoid-extra-gas-usage) | Consideration for Timestamp overflow in `_replenishVoltage()`                           |  
| [L - 03](#l-03-use-openzeppelins-ownable-contract-for-ownership-related-actions) | Use OpenZeppelin's Ownable contract for `ownership` related actions                |  
| [L - 04](#l-04-setupairdrop-function-arg-recipients-can-have-duplicates) | `setupAirdrop()` function arg: recipients[] can have duplicates                    | 
| [L - 05](#l-05-unsafe-erc20-operations-use-safeerc20-library) | Unsafe ERC20 Operation(s), use `SafeERC20` library            |  
| [L - 06](#l-06-transferfrom-returns-a-bool-value-in-claim-which-is-not-checked) | `transferFrom()` returns a bool value in `claim()`, which is not checked                  |  
| [NC-01](#nc-01-contracts-doesnt-follow-the-layout-ordering) | Contracts doesn't follow the layout ordering                  | 
| [NC-02](#nc-02-functions-mutating-storage-can-better-emit-events-for-off-chain-tracking) | Functions mutating storage, can better emit events for off-chain tracking                                                     | _Non Critical_ |
| [NC-03](#nc-03-better-to-emit-events-before-making-an-external-call) | Better to emit events before making an external call                    |  

---
---
## [G-01] Check for `amount = 0` before making `_burn()` call
#### Summary:
It is unnecessary to call `_burn()`  on amount  = 0, it leads to wastage of gas as `_burn()`  itself doesn't have such check before writing to state variables of the contract, thereby calling `SSTORE` EVM opcode, which costs significant amount of gas.

*"SSTORE, the opcode which saves data in storage, costs 20,000 gas when a storage value is set to a non-zero value from zero and costs 5000 gas when a storage variable’s value is set to zero or remains unchanged from zero."*

#### [Neuron.sol::burn()](https://github.com/code-423n4/2024-02-ai-arena/blob/main/src/Neuron.sol#L163-L165)
**Code:**
```javascript
function  burn(uint256 amount) public virtual {
	//qa/gas- don't call for amt = 0, wastage of gas on add/sub 0 
	//to/from state variable.
	_burn(msg.sender, amount);
}
```
#### Recommendation: 
Updating the above code snippet to the below given can avoid unnecessary usage of excessive gas caused by absurd operations.
```javascript
function  burn(uint256 amount) public virtual {
	 require(amount!=0,"Zero Amount not allowed")
	_burn(msg.sender, amount);
}
```
##
## [G-02] Cache 	`attributes.length` only once to avoid extra gas usage
#### Summary: 
The `attributesLength` caches to avoid computation of array length of state variable (costly) with each iteration is appreciated.

But, if we declare it before doing `require(attributeDivisors.length == attributes.length);`
it can save some gas, though we might introduce `mstore` opcode use, but since the function is only callable by owner, it is less likely that above require check to fail. 
#### [AiArenaHelper::addAttributeDivisor](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L68-L76)

**Code:** 
 ```javascript
 function addAttributeDivisor(uint8[] memory attributeDivisors) external {
        require(msg.sender == _ownerAddress);
        require(attributeDivisors.length == attributes.length);
	
        uint256 attributesLength = attributes.length;
        for (uint8 i = 0; i < attributesLength; i++) {
            attributeToDnaDivisor[attributes[i]] = attributeDivisors[i];
        }
    }    
```

#### Recommendation: 
 ```diff
 function addAttributeDivisor(uint8[] memory attributeDivisors) external {
      + uint256 attributesLength = attributes.length;
        require(msg.sender == _ownerAddress);
        require(attributeDivisors.length == attributesLength);

      - uint256 attributesLength = attributes.length;
        for (uint8 i = 0; i < attributesLength; i++) {
            attributeToDnaDivisor[attributes[i]] = attributeDivisors[i];
        }
    }    
```
### Another Instance: 
#### [createPhysicalAttributes](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L96-L98)

**Code:**
```javascript
uint256[] memory finalAttributeProbabilityIndexes =  new  uint[](attributes.length);

uint256 attributesLength = attributes.length;
```
#### Recommendation: 
```diff
+ uint256 attributesLength = attributes.length;
  uint256[] memory finalAttributeProbabilityIndexes =  new  uint[](attributes.length);
- uint256 attributesLength = attributes.length;
```
##
