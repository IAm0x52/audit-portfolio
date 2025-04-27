# UBet Parlay Audit Report

Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

Prepared For: UBet

Review Date(s): 12/9/24 - 12/11/24

## <br/> 0x52 Background

As a professional smart contract auditor, I have conducted over 100 security reviews for public and private clients. With 30+ first-place finishes in public contests on platforms like [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52), I have been recognized as a top-performing security expert. By prioritizing rigorous analysis and providing actionable recommendations, I have contributed to securing over **$1 billion in TVL across 100+ protocols**. Throughout my career I have collaborated with many organizations including  the prestigious [Blackthorn](https://www.blackthorn.xyz/) as a founding security researcher and as a Lead Security researcher at [SpearbitDAO](https://cantina.xyz/u/iam0x52).

## <br/> Protocol Summary

UBet Parlay is a smart contract system for batch betting and parlay bets on the UBet platform, supporting multiple markets and event types.

[Polygon][ERC4626][Prediction][AMM]

## <br/> Scope

Repo: [ubet-contract-v1](https://github.com/SportsFI-UBet/ubet-contracts-v1)

Review Hash: [7b61ff6](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/)

Fix Review Hash: 

In-Scope Contracts
- `src/conditions/*`
- `src/funding/*`
- `src/markets/*`
- `src/AdminExecutorAccess.sol`
- `src/Math.sol`
- `src/PackedPrices.sol`

Deployment Chain(s)
- Polygon Mainnet

## <br/> Summary of Findings

| Identifier | Title | Severity | Mitigated |
| ------ | ----- | -------- | --------- |
| [L-01] | [Events in BatchBet can be spoofed using custom pool/market](#l-01-events-in-batchbet-can-be-spoofed-using-custom-poolmarket) | LOW | ✔️ |
| [L-02] | [Transferring ERC1155 tokens before applying parent return ops allowing reentrancy](#l-02-transferring-erc1155-tokens-before-applying-parent-return-ops-allowing-reentrancy) | LOW | ✔️ |

## <br/> Low Risk Findings

### [L-01] Events in BatchBet can be spoofed using custom pool/market

#### Details

[BatchBet.sol#L89-L111](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/BatchBet.sol#L89-L111)

```solidity
for (uint256 i = 0; i < buys.length; i++) {
    address market = buys[i].market;

    if (!market.supportsInterface(MARKET_MAKER_V1_2_INTERFACE_ID)) {
        revert NotAMarket(market);
    }

    token.safeApprove(market, buys[i].investmentAmount);

    (, uint256 feeAmount,) = IMarketMakerV1_2(market).buyFor(
        buyer,
        buys[i].investmentAmount,
        buys[i].outcomeIndex,
        buys[i].minOutcomeTokensToBuy,
        extraFeeDecimal,
        feeProfileId
    );
    totalFees += feeAmount;
}

if (totalInvestment > 0 && (data.length > 0 || affiliate != address(0x0))) {
    emit BuyWithData(buyer, affiliate, address(token), totalInvestment, data);
}
```

Above we see that the `market` interacted with can be arbitrarily supplied by the user. This allows specifying a contract different from a UBet owned market. Along with this take note that the `market` address is never specified inside the `BuyWithData` event. The result is that events can be spoofed to make user/affiliate volume look significantly higher than the real value. This can have a negative effect on any offchain analytics or calculations.

#### Lines of Code

[BatchBet.sol#L109-L111](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/BatchBet.sol#L109-L111)

[BatchBet.sol#L155-L157](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/BatchBet.sol#L155-L157)

#### Recommendation

Include the market address in event so that non-UBet market data can be filtered out

#### Remediation


### <br/> [L-02] Transferring ERC1155 tokens before applying parent return ops allowing reentrancy

#### Details

[MarketMaker.sol#L392-L397](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/MarketMaker.sol#L392-L397)

```solidity
conditionalTokens.safeTransferFrom(address(this), receiver, positionId(outcomeIndex), outcomeTokensBought, "");
// Last index outcome is the refund outcome. Give back the same amount of tokens as collateral invested, including fees
conditionalTokens.safeTransferFrom(address(this), receiver, positionId(refundIndex), investmentAmount, "");

// Return collateral back to parent once everything is settled with the buyer
_applyParentReturn(parentOps);
```

Above we see that inside the `buyFor` function the `conditionalTokens` (ERC1155) are transferred to the buyer prior to the the collateral being returned to the parent pool. This enables reentrancy prior to a significant state change to the market funding pool. Although no significant attack vectors were discovered as a result of this reentrancy is it still recommended to resolve this issue it may cause vulnerabilities in future versions.

#### Lines of Code

[MarketMaker.sol#L392-L397](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/MarketMaker.sol#L392-L397)

#### Recommendation

Move token transfers to after `_applyParentReturn`. Additionally utilize `safeBatchTransferFrom` to prevent reentrancy between conditional token transfers.

#### Remediation

