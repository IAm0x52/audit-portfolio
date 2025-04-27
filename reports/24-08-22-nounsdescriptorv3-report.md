# NounsDescriptorV3 Audit Report

Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

Prepared For: NounsDAO

Review Date(s): 8/22/24 - 8/24/24

## <br/> 0x52 Background

As a professional smart contract auditor, I have conducted over 100 security reviews for public and private clients. With 30+ first-place finishes in public contests on platforms like [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52), I have been recognized as a top-performing security expert. By prioritizing rigorous analysis and providing actionable recommendations, I have contributed to securing over **$1 billion in TVL across 100+ protocols**. Throughout my career I have collaborated with many organizations including  the prestigious [Blackthorn](https://www.blackthorn.xyz/) as a founding security researcher and as a Lead Security researcher at [SpearbitDAO](https://cantina.xyz/u/iam0x52).

## <br/> Protocol Summary

NounsDescriptorV3 is a rendering engine for the NounsDAO NFT ecosystem that enables governance-controlled updates to existing NFT traits, allowing for corrections to visual assets through properly validated trait management functions.

[Ethereum][ERC721]

## <br/> Scope

Repo: N/A (Deployed Contracts)

Review Hash: N/A

Fix Review Hash: N/A

The following contracts were reviewed as replacements for the [nounsDAO](https://nouns.wtf/) contract suite:

- `NFTDescriptorV2`: [0xdEdd7Ec3F440B19C627AE909D020ff037F618336](https://etherscan.io/address/0xdEdd7Ec3F440B19C627AE909D020ff037F618336)
- `SVGRenderer`: [0x535BD6533f165B880066A9B61e9C5001465F398C](https://etherscan.io/address/0x535BD6533f165B880066A9B61e9C5001465F398C)
- `NounsDescriptorV3`: [0x33A9c445fb4FB21f2c030A6b2d3e2F12D017BFAC](https://etherscan.io/address/0x33A9c445fb4FB21f2c030A6b2d3e2F12D017BFAC)
- `Inflator`: [0x6c14b7aB60d81d5F734B873126493de2E52d3eee](https://etherscan.io/address/0x6c14b7aB60d81d5F734B873126493de2E52d3eee)
- `NounsArt`: [0x6544bC8A0dE6ECe429F14840BA74611cA5098A92](https://etherscan.io/address/0x6544bC8A0dE6ECe429F14840BA74611cA5098A92)

Deployment Chain(s)
- Ethereum Mainnet

## <br/> Summary of Changes

`NounsArt.sol` and `NounsDescriptorV3.sol` were changed to allow the NounsDAO executor to update existing traits with new artwork. A suite of `updateTrait` and `updateTraitFromPointer` functions were added to NounsDescriptorV3.sol along with corresponding functions on NounsArt.sol. The motivation is to allow traits that are not displaying correct to be fixed via a governance proposal.

## <br/> Summary of Review

The code first underwent manual review. This was to identify all flows across the nounsDAO suite that would be altered by the proposed changes. Only cosmetic flows were altered and changes present no risk to core functionality such as minting or governance. Secondary manual review was completed to evaluate any structural security concerns raised by these changes. The functions added were derived from the existing functions used to add additional traits. State-altering functions such as `addPage` were reused rather than remade, which is a security best practice. The worst security outcomes stem from incorrect trait counts. The addition of the trait length checks before and after updating completely eliminate this form of input error. Contracts were subsequently fork tested to confirm desired functionality and access control were working as intended.

**No security concerns have been raised by this review**