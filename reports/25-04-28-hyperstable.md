# Hyperstable Flashmint Audit Report

Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

Prepared For: Hyperstable

Review Date(s): 4/28/25

Fix Review Date(s): 4/29/25

## <br/> 0x52 Background

As a professional smart contract auditor, I have conducted over 100 security reviews for public and private clients. With 30+ first-place finishes in public contests on platforms like [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52), I have been recognized as a top-performing security expert. By prioritizing rigorous analysis and providing actionable recommendations, I have contributed to securing over **$1 billion in TVL across 100+ protocols**. Throughout my career I have collaborated with many organizations including  the prestigious [Blackthorn](https://www.blackthorn.xyz/) as a founding security researcher and as a Lead Security researcher at [SpearbitDAO](https://cantina.xyz/u/iam0x52).

## <br/> Protocol Summary

Hyperstable core update introducing flashminting, allowing users to mint and repay loans atomically in a single transaction for instant, permissionless liquidity.

[HyperEVM][DeFi][Stablecoin][Lending]

## <br/> Scope

Repo: [hyperstable/contracts](https://github.com/hyperstable/contracts)
Review Hash: [d92d9cd](https://github.com/hyperstable/contracts/pull/82/commits/d92d9cd8873928d7326bcfab4232aff23f0c7885)
Fix Review Hash: [c9112f7](https://github.com/hyperstable/contracts/pull/82/commits/c9112f7708bc7e59ed38b07ade34d6242097a16f)

In-Scope Contracts
- src/core/DebtToken.sol

Deployment Chain(s)
- HyperEVM

## <br/> Summary of Findings

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [L-01] | [flashFee and flashLoan should revert if `token != address(this)` to comply with ERC3156](#l-01-flashfee-and-flashloan-should-revert-if-token--addressthis-to-comply-with-erc3156) | LOW | ✔️ |
| [L-02] | [1% flashloan fee in inconsistent with vault fees](#l-02-1-flashloan-fee-in-inconsistent-with-vault-fees) | LOW | ✔️ |
| [L-03] | [flashLoan allows excessively large flashloans](#l-03-flashloan-allows-excessively-large-flashloans) | LOW | ✔️ |

## <br/> Low Findings

### [L-01] flashFee and flashLoan should revert if `token != address(this)` to comply with ERC3156

#### Details 

According to the ERC3156 [spec](https://eips.ethereum.org/EIPS/eip-3156) and reference implementation, `flashFee()` and `flashLoan()` MUST revert if the `token` is not supported. The current implementation of `flashFee()` and `flashLoan()` do not follow this recommendation and therefore not fully spec compliant.

#### Recommendation

Update `flashFee()` and `flashLoan()` to revert if `token != address(this)`

#### Remediation

Fixed as recommended in commit [5d5bace](https://github.com/hyperstable/contracts/pull/82/commits/5d5bace67875d15887d9379b2364239f187c4279)

### <br/> [L-02] 1% flashloan fee in inconsistent with vault fees

#### Details 

Currently a fee of 1% is charged on the flashloan amount. This in inconsistent with the vault implementation as a user can deposit -> borrow -> repay -> withdraw for no fee at all. This make the flashloan a unrealistic alternative as it is always better to use an atomic vault loan instead. Given that a majority of AMMs support flashloans at swap fees (often as low as 0.01%), collateral for the atomic vault loan can flash borrowed.

#### Recommendation

Reduce or remove flash loan fee.

#### Remediation

Flashloan fee can be adjusted post deployment

### <br/> [L-03] `maxFlashLoan()` allows excessively large flashloans

#### Details 

In the current implementation a user can flashloan up to `type(uint256).max - totalSupply()`. This is beyond the reasonable amount that any honest actor would use. Flashloans beyond a certain threshold no longer bring any additional opportunity to honest actors and only serve to empower malicious actors to exploit edge cases or liquidity pools. 

#### Recommendation

Limit `maxFlashLoan()` to some configurable amount. A reasonable starting value would be $5M that can be increased over time as the TVL of the protocol.

#### Remediation

Fixed as recommended in commit [a3ad597](https://github.com/hyperstable/contracts/pull/87/commits/a3ad597f8d35a54d00f5ab026705161669facc8d)