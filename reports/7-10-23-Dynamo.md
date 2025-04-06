
# Dynamo Audit Report

### Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

### Review Dates: 7/10/23 - 7/14/23

# <br/> About 0x52

As a professional smart contract auditor, I have conducted over 100 security reviews for public and private clients. With 30+ first-place finishes in public contests on platforms like [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52), I have been recognized as a top-performing security expert. By prioritizing rigorous analysis and providing actionable recommendations, I have contributed to securing over **$1 billion in TVL across 100+ protocols**. Throughout my career I have collaborated with many organizations including  the prestigious [Blackthorn](https://www.blackthorn.xyz/) as a founding security researcher and as a Lead Security researcher at [SpearbitDAO](https://cantina.xyz/u/iam0x52).

# <br/> Scope

The [Dynamo](https://github.com/DynamoFinance/vault/tree/master) repo was reviewed at hash [c331ffefad](https://github.com/DynamoFinance/vault/tree/c331ffefadec7406829fc9f2e7f4ee7631bef6b3)

In-Scope Contracts
- contracts/aaveAdapter.vy
- contracts/compoundadapter.vy
- contracts/Dynamo4626.vy
- contracts/fraxlendadapter.vy
- contracts/FundsAllocator.vy
- contracts/Governance.vy

Deployment Chain(s)
- Ethereum Mainnet

# <br/> Summary of Findings

|  Identifier  | Title                        | Severity      | Fixed |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | Access control modifiers on _claim_fees will permanently lock proposer | High | X |
| [H-02] | Math error in Dynamo4626#_claimable_fees_available will lead to fees or strategy lockup | High | X |
| [H-03] | Malicious user can disable compound integration via share manipulation  | High | X |
| [H-04] | Dangerous approval/rejection criteria when number of guards is odd | High | X |
| [H-05] | A single malfunctioning/malicious adapter can permanently DOS entire vault | High | X |
| [M-01] | Governance#replaceGovernance is unable to actually change vault governance | Med | X |
| [M-02] | Assert statement in Dynamo4626#_claimable_fees_available can cause vault softlock in the event of partial fund loss | Med | X |
| [L-01] | aaveAdapter.vy has no method to claim LP incentives | Low |  |
| [L-02] | Ownership of governance.vy can't be changed after initialization | Low | X |

# <br/> Detailed Findings

## [H-01] Access control modifiers on _claim_fees will permanently lock proposer

### Details 

[Dynamo4626.vy#L519-L521](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Dynamo4626.vy#L519-L521)

    elif _yield == FeeType.PROPOSER:
        assert msg.sender == self.current_proposer, "Only curent proposer may claim strategy fees."
        self.total_strategy_fees_claimed += claim_amount        

Dynamo4626#_set_strategy attempts to distribute fees to proposer when proposer changes. The problem is that _claim_fees requires that msg.sender == proposer. Since _set_strategy can only be called by governance this subcall will always revert. The result is that the first proposer will have a monopoly on all proposals since any strategy that wasn't submitted by them would fail when attempting to activate it.

### Lines of Code

https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Dynamo4626.vy#L496-L533

### Recommendation

Revise access control on _set_strategy. I would suggest allowing anyone to claim tokens but sending to the correct target instead of msg.sender

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/1601b0acd23783ed87b9b3ae01c6a97a462a41a8) by allowing governance to claim on behalf of proposer

## <br/> [H-02] Math error in Dynamo4626#_claimable_fees_available will lead to fees or strategy lockup

### Details 

[Dynamo4626.vy#L428-L438](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Dynamo4626.vy#L428-L438)

    fee_percentage: uint256 = YIELD_FEE_PERCENTAGE
    if _yield == FeeType.PROPOSER:
        fee_percentage = PROPOSER_FEE_PERCENTAGE
    elif _yield == FeeType.BOTH:
        fee_percentage += PROPOSER_FEE_PERCENTAGE
    elif _yield != FeeType.YIELD:
        assert False, "Invalid FeeType!" 

    total_fees_ever : uint256 = (convert(total_returns,uint256) * fee_percentage) / 100

    assert self.total_strategy_fees_claimed + self.total_yield_fees_claimed <= total_fees_ever, "Total fee calc error!"

In the assert statement, total_fees_ever is compared against both fees types of fees claimed. The issue with this is that this is a relative value depending on which type of fee is being claimed. The assert statement on the other hand always compares as if it is FeeType.BOTH. This will lead to this function unexpectedly reverting when trying to claim proposer fees. This leads to stuck fees but also proposer locked as described in H-01.

### Lines of Code

[Dynamo4626.vy#L418-L459](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Dynamo4626.vy#L418-L459)

### Recommendation

Check should be made against the appropriate values (i.e. proposer should be check against only self.total_strategy_fees_claimed).

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/6e762711f55f9cf4bece42706ddef7c92d5b4ac4) and [here](https://github.com/DynamoFinance/vault/commit/c649ceda1b7b15b14486bd1332aefb8d48ed5279) by splitting fee calculations base on fee type being claimed

## <br/> [H-03] Malicious user can disable compound integration via share manipulation 

### Details 

It's a common assumption that Compound V2 share ratio can only ever increase but with careful manipulation it can actually be lowered. The full explanation is a bit long but you can find it [here](https://github.com/code-423n4/2023-01-reserve-findings/issues/310) in one of my public reports.

[FundsAllocator.vy#L67-L71](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/FundsAllocator.vy#L67-L71)

          if pool.current < pool.last_value:
              # We've lost value in this adapter! Don't give it more money!
              blocked_adapters[blocked_pos] = pool.adapter
              blocked_pos += 1
              pool.delta = 0 # This will result in no tx being generated.

This quirk of Compound V2 can be used to trigger the check in FundsAllocator to block the Compound V2 adapter. This is useful if the user wants to push their own proposal allowing them to sabotage other users and cause loss of yield to the vault.
                    
### Lines of Code

[FundsAllocator.vy#L67-L71](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/FundsAllocator.vy#L67-L71)

### Recommendation

Instead of using an absolute check, instead only block the adapter if there is reasonable loss.

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/e3092bf4908e5f1d049e18fc52d310c9d8ce29ae) by allowing larger nominal (but still very small) loss before disabling it

## <br/> [H-04] Dangerous approval/rejection criteria when number of guards is odd

### Details 

[Governance.vy#L310](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Governance.vy#L310)

    assert (len(pending_strat.VotesEndorse) >= len(self.LGov)/2) or \

The above assert statement requires that the number of endorsements equals or exceeds the number of guards / 2. This becomes an issue with odd numbers due to truncation. If you were to have 3 guards then even a single approval would allow instant approval ( 3/2 = 1). In this scenario even a single malicious or compromised guard could drain the entire vault via a malicious proposal.

### Lines of Code

[Governance.vy#L291-L324](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Governance.vy#L291-L324)

### Recommendation

Make the requirement that it must equal or exceed length / 2 + length % 2

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/835993b6e9357b2246139cca43502cfc65a34e0a) by changing `len(self.LGov)/2` to `len(self.LGov)/2 + 1` so that a majority vote is required to pass

## <br/> [H-05] A single malfunctioning/malicious adapter can permanently DOS entire vault

### Details 

https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/FundsAllocator.vy#L47-L57

    for pos in range(MAX_POOLS):
        pool : BalancePool = _pool_balances[pos]
        if pool.adapter == empty(address): break

        # If the pool has been removed from the strategy then we must empty it!
        if pool.ratio == 0:
            pool.target = 0
            pool.delta = convert(pool.current, int256) * -1 # Withdraw it all!
        else:
            pool.target = (total_pool_target_assets * pool.ratio) / _total_ratios      
            pool.delta = convert(pool.target, int256) - convert(pool.current, int256)            

When rebalancing the vault, FundsAllocator attempts to withdraw/deposit from each adapter. In the event that the underlying protocol (such as AAVE) disallows deposits or withdrawals (or is hacked), the entire vault would be DOS'd since rebalancing is called on every withdraw, deposit or strategy change.

### Lines of Code

[FundsAllocator.vy#L29-L94](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/FundsAllocator.vy#L29-L94)

### Recommendation

Add an emergency function to force remove adapters and make it accessible via Governance.vy

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/24f9d95cc6a7ce62c5a0229c103fe9a95cc39e12) by simply bypassing failed adapter calls

## <br/> [M-01] Governance#replaceGovernance is unable to actually change vault governance 

### Details 

[Governance.vy#L472-L477](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Governance.vy#L472-L477)

    for guard_addr in self.LGov:
        if self.VotesGCByVault[vault][guard_addr] == NewGovernance:
            VoteCount += 1

    # if len(self.LGov) == VoteCount:
    #     Vault(self.Vault).replaceGovernanceContract(NewGovernance)

The code to actually replace the governance contract has been commented out resulting in it being impossible to ever change the governance contract.

### Lines of Code

[Governance.vy#L472-L477](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Governance.vy#L472-L477)

### Recommendation

Restore code by removing comment

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/eb85de2bf7aadacf529691c42ba83d404a3519e9) by removing comment

## <br/> [M-02] Assert statement in Dynamo4626#_claimable_fees_available can cause vault softlock in the event of partial fund loss

### Details 

[Dynamo4626.vy#L438](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Dynamo4626.vy#L438)

    assert self.total_strategy_fees_claimed + self.total_yield_fees_claimed <= total_fees_ever, "Total fee calc error!"

In the event of partial fund loss there may be legit cases where this assert statement is triggered. If the vault suffers a partial loss but still maintains a positive return (i.e. it has made 100e18 but suffers a loss of 50e18) then this statement will improperly revert. Given this function is called with every deposit and withdraw the vault would be completely DOS'd until yield (or donation) recovered the difference. 

### Lines of Code

[Dynamo4626.vy#L418-L459](https://github.com/DynamoFinance/vault/blob/c331ffefadec7406829fc9f2e7f4ee7631bef6b3/contracts/Dynamo4626.vy#L418-L459)

### Recommendation

Instead of reverting, simply return 0.

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/6e762711f55f9cf4bece42706ddef7c92d5b4ac4) by returning 0 instead of reverting

## <br/> [L-01] aaveAdapter.vy has no method to claim LP incentives

### Details 

Considering this low since the primary scope for this audit is mainnet Ethereum.

On alt L1's and L2's AAVE V3 frequently has LP incentives (such as OP tokens on Optimism). The current adapter has no methodology to claim these tokens. Any tokens accumulated to the vault would be impossible to claim leading to loss of yield for all LP's in the vault.

### Lines of Code

https://github.com/DynamoFinance/vault/blob/master/contracts/aaveAdapter.vy

### Recommendation

Before deploying anything other than mainnet make sure to include a way to claim and distribute/swap rewards to the vault.

### Remediation

Acknowledged that this is a limitation of their current implementation 

## <br/> [L-02] Ownership of governance.vy can't be changed after initialization

### Details 

**Additional change requested after the audit period**

Ownership of governance.vy is set in the constructor but there is no function for change the ownership after it has been set. This hinders future migrations and adds extra complexity to the deployment process.

### Lines of Code

https://github.com/DynamoFinance/vault/blob/master/contracts/aaveAdapter.vy

### Remediation

Fixed [here](https://github.com/DynamoFinance/vault/commit/acd2325b9f2a5c9fbed1b52aa5c774eff0f64897) by adding the replaceOwner function.