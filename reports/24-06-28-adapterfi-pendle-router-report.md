# AdapterFi Pendle Adapter Audit Report

Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

Prepared For: AdapterFi

Review Date(s): 6/28/24

## <br/> 0x52 Background

As a professional smart contract auditor, I have conducted over 100 security reviews for public and private clients. With 30+ first-place finishes in public contests on platforms like [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52), I have been recognized as a top-performing security expert. By prioritizing rigorous analysis and providing actionable recommendations, I have contributed to securing over **$1 billion in TVL across 100+ protocols**. Throughout my career I have collaborated with many organizations including  the prestigious [Blackthorn](https://www.blackthorn.xyz/) as a founding security researcher and as a Lead Security researcher at [SpearbitDAO](https://cantina.xyz/u/iam0x52).

## <br/> Protocol Summary

AdapterFi Pendle Adapter is a specialized yield strategy component that incorporates modified pricing methodology for Pendle's principal tokens, enhancing accuracy by valuing assets against standardized yield tokens rather than underlying assets.

[Ethereum][ERC4626][DeFi][Yield Aggregator]

## <br/> Scope

Repo: [AdapterFi](https://github.com/adapter-fi/AdapterVault)

Review Hash: [148e0df](https://github.com/adapter-fi/AdapterVault/commit/148e0df4aabc104ac0ec913897dd2ccc3b67ba10)

Fix Review Hash: N/A

Only the very small subset of changes made in commit [148e0df](https://github.com/adapter-fi/AdapterVault/commit/148e0df4aabc104ac0ec913897dd2ccc3b67ba10) were reviewed and any other changes were not considered.

In-Scope Contracts
- `contracts/adapters/PendleAdapter.vy`

Deployment Chain(s)
- Ethereum Mainnet
- Arbitrum Mainnet

## <br/> Summary of Changes

The underlying pricing for Pendle PT's was changed. Previously each PT was valued against the underlying asset of the derivative (i.e. ETH for stETH PT) rather than derivative itself. This was leading to incorrect price quotes and reverting transactions. With this new methodology the PT (principle tokens) are quoted against the SY (standardized yield) token. SY tokens can be directly converted to the underlying derivative token via `previewWithdraw` or `previewDeposit` allowing proper pricing.

## <br/> Summary of Review

No vulnerabilities were found to have been introduced by these changes.