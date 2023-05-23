# Juicebox Buyback Delegate protocol Audit-submission

# Introduction

A time-boxed security review of the Juicebox Buyback Delegate protocol was done by Mohammed Rizwan, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About Juicebox Buyback Delegate

Thousands of projects use Juicebox to fund, operate, and scale their ideas & communities transparently on Ethereum.

# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

**_review commit hash_ - [2023-05-juicebox](https://github.com/code-423n4/2023-05-juicebox/commit/e844709f192a3691776dee0739f0894f520c9a94)**

### Scope

The following smart contracts were in scope of the audit:

- `juice-buyback/contracts/JBXBuybackDelegate.sol`

The following number of issues were found, categorized by their severity:

- Critical & High: 0 issues
- Medium: 1 issues
- Low: 1 issues
- Informational: 2 issues
- Gas: 2 issues

---

# Findings Summary

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [M-01] | Missing deadline checks allow pending transactions to be maliciously executed   | Medium   |
| [L-01] | For immutable variables, Zero address checks are missing in constructor      | Low      |
| [I-01] | Use latest version of PRBMath library      | Informational      |
| [I-02] | Use a more recent version of Solidity      | Informational      |
| [G-01] | Save gas by removing duplicate/refactoring code      | Gas      |
| [G-02] | Remove unused imports(ownable.sol)      | Gas      |

# Detailed Findings

# [M-01] {Missing deadline checks allow pending transactions to be maliciously executed}

## Severity
Medium

## Impact:
The JBXBuybackDelegate.sol contract does not allow users to submit a deadline for their actions which execute swaps on Uniswap V3. This missing feature enables pending transactions to be maliciously executed at a later point.


AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example see [Uniswap V2](https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/UniswapV2Router02.sol#L229) and [Uniswap V3](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/SwapRouter.sol#L119)). If such an option is not present, users can unknowingly perform bad trades:

1. Alice wants to swap 100 tokens for 1 ETH and later sell the 1 ETH for 1000 DAI.
2. The transaction is submitted to the mempool, however, Alice chose a transaction fee that is too low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.
3. When the average gas fee dropped far enough for Alice’s transaction to become interesting again for miners to include it, her swap will be executed. In the meantime, the price of ETH could have drastically changed. She will still get 1 ETH but the DAI value of that output might be significantly lower. She has unknowingly performed a bad trade due to the pending transaction she forgot about.

An even worse way this issue can be maliciously exploited is through MEV:

1. The swap transaction is still pending in the mempool. Average fees are still too high for miners to be interested in it. The price of tokens has gone up significantly since the transaction was signed, meaning Alice would receive a lot more ETH when the swap is executed. But that also means that her maximum slippage value (sqrtPriceLimitX96 and minOut in terms of the Spell contracts) is outdated and would allow for significant slippage.
2. A MEV bot detects the pending transaction. Since the outdated maximum slippage value now allows for high slippage, the bot sandwiches Alice, resulting in significant profit for the bot and significant loss for Alice.


## Proof of Concept
[Link to code](https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L258-L268)

## Recommendations
Introduce a deadline parameter to all functions which potentially perform a swap on the user’s behalf.


# [L-01] {For immutable variables, Zero address checks are missing in constructor}

## Severity
Low

## Impact:
Zero address check validations should be used in the constructors, to avoid the risk of setting a immutable storage variable as zero address at the time of deployment. If bymistake, address(0) is set it will cause redeployment of contract.

There are 4 instances of this issue:

## Proof of Concept
```solidity
File: juice-buyback/contracts/JBXBuybackDelegate.sol

118    constructor(
119        IERC20 _projectToken,
120        IWETH9 _weth,
121        IUniswapV3Pool _pool,
122        IJBPayoutRedemptionPaymentTerminal3_1 _jbxTerminal
123    ) {
124        projectToken = _projectToken;
125        pool = _pool;
126        jbxTerminal = _jbxTerminal;
127        _projectTokenIsZero = address(_projectToken) < address(_weth);
128        weth = _weth;
129    }
```
[Link to code](https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L118-L129)

## Recommendations
Add address(0) validation check in constructor.

```solidity
File: juice-buyback/contracts/JBXBuybackDelegate.sol

   constructor(
       IERC20 _projectToken,
       IWETH9 _weth,
       IUniswapV3Pool _pool,
       IJBPayoutRedemptionPaymentTerminal3_1 _jbxTerminal
   ) {
+       require(address(_projectToken) != address(0), "invalid address");
+       require(address(_weth) != address(0), "invalid address");
+       require(address(_pool) != address(0), "invalid address");
+       require(address(_jbxTerminal) != address(0), "invalid address");
       projectToken = _projectToken;
       pool = _pool;
       jbxTerminal = _jbxTerminal;
       _projectTokenIsZero = address(_projectToken) < address(_weth);
       weth = _weth;
   }
```

# [I-01] {Use latest version of PRBMath library}

## Severity
Informational

## Impact:
The contract uses old version of PRBMath library 3.7.0. The latest version v4.0.0 has lots of fixes and some breaking changes. It is recommended to use latest version of PRBMath library.

[Link to latest release features](https://github.com/PaulRBerg/prb-math/releases/tag/v4.0.0)

There is 1 instance of this issue.

## Proof of Concept
```solidity
File: /juice-buyback/package.json

"@paulrberg/contracts": "^3.7.0",
```

## Recommendations
Use latest version v4.0.0.

# [I-02] {Use a more recent version of Solidity}

## Severity
Informational

## Impact:
For security and optimization, it is best practice to use the latest Solidity version.
For the security fix list in the versions: [Link to reference](https://github.com/ethereum/solidity/blob/develop/Changelog.md)

There is 1 instance of this issue.


## Proof of Concept
```solidity
File: juice-buyback/contracts/JBXBuybackDelegate.sol

pragma solidity ^0.8.16;
```

## Recommendations
Use latest version v0.8.19.


# [G-01] {Save gas by removing duplicate/refactoring code}

## Severity
Gas

## Impact:
This code can be refactored and duplicate code can be removed. This will reduce number of bytes used which ultimately saves some gas.

## Proof of Concept

```solidity
File: juice-buyback/contracts/JBXBuybackDelegate.sol

200        if (_data.preferClaimedTokens) {
201            // Try swapping
202            uint256 _amountReceived = _swap(_data, _minimumReceivedFromSwap, _reservedRate);
203
204            // If swap failed, mint instead, with the original weight + add to balance the token in
205            if (_amountReceived == 0) _mint(_data, _tokenCount);
206        } else {
207            _mint(_data, _tokenCount);
208        }
```
[Link to code](https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L200-L207)

## Recommendations
```solidity
File: juice-buyback/contracts/JBXBuybackDelegate.sol

-        if (_data.preferClaimedTokens) {
-            // Try swapping
-            uint256 _amountReceived = _swap(_data, _minimumReceivedFromSwap, _reservedRate);
-
-            // If swap failed, mint instead, with the original weight + add to balance the token in
-            if (_amountReceived == 0) _mint(_data, _tokenCount);

+        if (_data.preferClaimedTokens && _swap(_data, _minimumReceivedFromSwap, _reservedRate) == 0){
+              _mint(_data, _tokenCount)
+        } else {
+            _mint(_data, _tokenCount);
+        }
```


# [G-02] {Remove unused imports(ownable.sol)}

## Severity
Gas

## Impact:
The contract is made ownable but it does not use ownable import functions in contract. Removing the unncessary imports saves some gas.
There is 1 instance of this issue.

## Proof of Concept
```solidity
File: juice-buyback/contracts/JBXBuybackDelegate.sol

contract JBXBuybackDelegate is IJBFundingCycleDataSource, IJBPayDelegate, IUniswapV3SwapCallback, Ownable {
```
[Link to code](https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L39)

## Recommendations
Remove ownable.sol imports.
