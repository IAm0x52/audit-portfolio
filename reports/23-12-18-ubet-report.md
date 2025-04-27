# UBet v1 Audit Report

Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

Prepared For: UBet

Review Date(s): 12/18/23 - 12/20/23

Fix Review Date(s): 1/10/24

## <br/> 0x52 Background

As a professional smart contract auditor, I have conducted over 100 security reviews for public and private clients. With 30+ first-place finishes in public contests on platforms like [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52), I have been recognized as a top-performing security expert. By prioritizing rigorous analysis and providing actionable recommendations, I have contributed to securing over **$1 billion in TVL across 100+ protocols**. Throughout my career I have collaborated with many organizations including  the prestigious [Blackthorn](https://www.blackthorn.xyz/) as a founding security researcher and as a Lead Security researcher at [SpearbitDAO](https://cantina.xyz/u/iam0x52).

## <br/> Protocol Summary

UBet is a decentralized sports betting platform utilizing conditional tokens, automated market makers, and funding pools to enable permissionless wagering with transparent pricing and liquidity provision mechanisms.

[Polygon][ERC4626][Prediction][AMM]

## <br/> Scope

Repo: [ubet-contracts-v1](https://github.com/SportsFI-UBet/ubet-contracts-v1)  

Review Hash: [2766b47](https://github.com/SportsFI-UBet/ubet-contracts-v1/tree/2766b47bed2cf027e29053af2afc4d35256747a5)  

Fix Review Hash: [7cfb625](https://github.com/SportsFI-UBet/ubet-contracts-v1/commit/7cfb6257f7249284d0e6107f3f5c2762d6a97a5a)

**NOTE:** This was not a complete review of the codebase and was instead a review of the changes proposed in [PR #69](https://github.com/SportsFI-UBet/ubet-contracts-v1/pull/69/files)

## <br/> Summary of Findings

| Identifier | Title                                                                                   | Severity | Mitigated |
|-----------|-------------------------------------------------------------------------------------------|----------|-----------|
| [H-01]    | [MarketMaker.sol is vulnerable to inflation attacks](#h-01-marketmakersol-is-vulnerable-to-inflation-attacks) | HIGH     | ✔️        |
| [H-02]    | [After migration, the old MarketFundingPool contract will fail to correctly distribute fees to new MarketFundingPool](#h-02-after-migration-the-old-marketfundingpool-contract-will-fail-to-correctly-distribute-fees-to-new-marketfundingpool) | HIGH     | ✔️        |

## <br/> High Risk Findings

### [H-01] MarketMaker.sol is vulnerable to inflation attacks

#### Details

[FundingMath.sol#L21-L37](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/2766b47bed2cf027e29053af2afc4d35256747a5/contracts/funding/FundingMath.sol#L21-L37)

```solidity
    function calcFunding(uint256 collateralAdded, uint256 totalShares, uint256 poolValue)
        internal
        pure
        returns (uint256 sharesMinted)
    {
        if (totalShares == 0) {
            // funding when LP pool is empty
            sharesMinted = collateralAdded;
        } else {
            // mint LP tokens proportional to how much value the new investment
            // brings to the pool

            // Something is very wrong if poolValue has gone to zero
            if (poolValue == 0) revert FundingErrors.PoolValueZero();
@>          sharesMinted = (collateralAdded * totalShares).ceilDiv(poolValue);
        }
    }
```

When adding liquidity the above lines are used to determine the number of shares to mint to the depositor. The use of `ceilDiv` in the `sharesMinted` calculation means that a user is guaranteed to receive at least 1 share when depositing. This allows the vault to become vulnerable to inflation attacks.

To execute this a malicious user would do as follows:
1. Deposit a single wei of liquidity  
2. Donate a large amount of collateral to inflate poolValue  
3. Buy a large number of a single outcome token to pull a large amount of funding from the MarketFundingPool  
4. Make a large number of 1 wei deposits  
5. Each deposit mints 1 share which dilutes the liquidity share of MarketMakerFunding  
6. After pool has closed withdraw all shares for a large profit.

#### Lines of Code

[FundingMath.sol#L35](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/2766b47bed2cf027e29053af2afc4d35256747a5/contracts/funding/FundingMath.sol#L35)

#### Recommendation

There are quite a few different ways to approach the solution. OpenZeppelin recommends using a [virtual offset](https://docs.openzeppelin.com/contracts/4.x/erc4626#defending_with_a_virtual_offset).

#### Remediation

Mitigated [here](https://github.com/SportsFI-UBet/ubet-contracts-v1/pull/72). A virtual offset of 10,000 has been added to the pool. This makes the capital requirements significantly higher as well as the number of transactions to exploit this.

### <br/> [H-02] After migration, the old MarketFundingPool contract will fail to correctly distribute fees to new MarketFundingPool

#### Details

[FundingPool.sol#L135-L142](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/2766b47bed2cf027e29053af2afc4d35256747a5/contracts/funding/FundingPool.sol#L135-L142)

```solidity
    function _beforeTokenTransfer(address from, address to, uint256 amount) internal override {
        if (from != address(0)) {
            // LP tokens being transferred away from a funder - any fees that
            // have accumulated so far due to trading activity should be given
            // to the original owner for the period of time he held the LP
            // tokens
@>          withdrawFees(from);
        }
```

`FundingPool#_beforeTokenTransfer` causes fees to be claimed when claiming new MarketFundingPool shares from the old MarketFundingPool. When receiving fees, they must be confirmed with the parent pool using `MarketMaker#_afterFeesWithdrawn`. When the shares are claimed for the new MarketFundingPool the fees are sent to the old MarketFundingPool but are never confirmed. When this happens the fees are instead distributed across all shares, causing loss of yield for the user claiming their shares.

#### Lines of Code

[FundingPool.sol#L135](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/608f889583ba86bf3df0b54c179b49afde69aeb7/contracts/funding/FundingPool.sol#L135)  
[MarketMaker.sol#L603-L607](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/608f889583ba86bf3df0b54c179b49afde69aeb7/contracts/markets/MarketMaker.sol#L603-L607)

#### Recommendation

Implement `_afterFeesWithdrawn` on MarketFundingPool to call `MarketMaker#_afterFeesWithdrawn` when sending fees to the previous MarketFundingPool.

#### Remediation

Mitigated [here](https://github.com/SportsFI-UBet/ubet-contracts-v1/pull/71). `_afterFeesWithdrawn` is now implemented as recommended above on `ParentFundingPool` (inherited by `MarketFundingPool`).
