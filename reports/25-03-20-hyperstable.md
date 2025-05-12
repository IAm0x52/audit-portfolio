# Hyperstable Perpetual Lock Audit Report

Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

Prepared For: Hyperstable

Review Date(s): 3/18/25 - 3/20/25

Fix Review Date(s): 4/11/25

## <br/> 0x52 Background

As a professional smart contract auditor, I have conducted over 100 security reviews for public and private clients. With 30+ first-place finishes in public contests on platforms like [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52), I have been recognized as a top-performing security expert. By prioritizing rigorous analysis and providing actionable recommendations, I have contributed to securing over **$1 billion in TVL across 100+ protocols**. Throughout my career I have collaborated with many organizations including  the prestigious [Blackthorn](https://www.blackthorn.xyz/) as a founding security researcher and as a Lead Security researcher at [SpearbitDAO](https://cantina.xyz/u/iam0x52).

## <br/> Protocol Summary

Hyperstable core update introducing a perpetual lock feature, letting users secure positions indefinitely and eliminating the need to manually relock.

[HyperEVM][DeFi][Stablecoin][Lending]

## <br/> Scope

Repo: [hyperstable/contracts](https://github.com/hyperstable/contracts)

Review Hash: [4f65012](https://github.com/hyperstable/contracts/commit/4f650122d3927fd45015c40ad58172508b148876)

Fix Review Hash: [f0168cc](https://github.com/hyperstable/contracts/commit/f0168ccd8489d155363a802721556b3a42339b2b)

As an update audit, only changes from commit [cb30a68](https://github.com/hyperstable/contracts/commit/cb30a683971974f6205c453a0d172a5431daa935) to commit [4f65012](https://github.com/hyperstable/contracts/commit/4f650122d3927fd45015c40ad58172508b148876) were reviewed and other changes were considered out-of-scope.

In-Scope Contracts
- src/core/InterestRateStrategyV1.sol
- src/core/LiquidationBuffer.sol
- src/core/LiquidationManager.sol
- src/core/PositionManager.sol
- src/core/Vault.sol
- src/governance/EmissionScheduler.sol
- src/governance/RewardDistributor.sol
- src/governance/TokenRewardsDistributor.sol
- src/governance/vePeg.sol
- src/libraries/AdaptiveIRM.sol
- src/libraries/PegIRM.sol

Deployment Chain(s)
- HyperEVM mainnet

## <br/> Summary of Findings

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | [vePeg#deposit_for fails to increment perpetualLockAmount leading to incorrect supply values](#h-01-vepegdeposit_for-fails-to-increment-perpetuallockamount-leading-to-incorrect-supply-values) | HIGH | ✔️ |
| [H-02] | [PositionManager#setLiquidationManager fails to give allowances to new liquidation manager](#h-02-positionmanagersetliquidationmanager-fails-to-give-allowances-to-new-liquidation-manager) | HIGH | ✔️ |
| [H-03] | [vePeg#delegate lacks any access control allowing all votes to be stolen through delegation](#h-03-vepegdelegate-lacks-any-access-control-allowing-all-votes-to-be-stolen-through-delegation) | HIGH | ✔️ |
| [M-01] | [Using a value of 0 for MIN_RATE_AT_TARGET in AdaptiveIRM causes interest rate to reset incorrectly](#m-01-using-a-value-of-0-for-min_rate_at_target-in-adaptiveirm-causes-interest-rate-to-reset-incorrectly) | MED | ✔️ |
| [M-02] | [vePeg#increase_amount and deposit_for incorrectly revert for perpetual locks](#m-02-vepegincrease_amount-and-deposit_for-incorrectly-revert-for-perpetual-locks) | MED | ✔️ |
| [M-03] | [vePeg#_delegate perpetualLocked requirement can easily be bypassed](#m-03-vepeg_delegate-perpetuallocked-requirement-can-easily-be-bypassed) | MED | ✔️ |
| [M-04] | [PositionManager#setInterestRateStrategy fails to re-register existing vaults with the new strategy](#m-04-positionmanagersetinterestratestrategy-fails-to-re-register-existing-vaults-with-the-new-strategy) | MED | ✔️ |
| [L-01] | [PositionManger#setLiquidationManager fails to revoke allowances to replaced liquidation manager](#l-01-positionmangersetliquidationmanager-fails-to-revoke-allowances-to-replaced-liquidation-manager) | LOW | ✔️ |
| [L-02] | [InterestRateStrategyV1 has no method to adjust risk parameters such as targetUtilization and MCR](#l-02-interestratestrategyv1-has-no-method-to-adjust-risk-parameters-such-as-targetutilization-and-mcr) | LOW | ✔️ |
| [L-03] | [vePeg#unlock_perpetual will redelegate all tokens](#l-03-vepegunlock_perpetual-will-redelegate-all-tokens) | LOW | ✔️ |

## <br/> Detailed Findings

### [H-01] vePeg#deposit_for fails to increment perpetualLockAmount leading to incorrect supply values

#### Details 

[vePeg.sol#L757-L764](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L757-L764)

    function deposit_for(uint256 _tokenId, uint256 _value) external nonreentrant {
        LockedBalance memory _locked = locked[_tokenId];

        require(_value > 0); // dev: need non-zero value
        require(_locked.amount > 0, "No existing lock found");
        require(_locked.end > block.timestamp, "Cannot add to expired lock. Withdraw");
        _deposit_for(_tokenId, _value, 0, _locked, DepositType.DEPOSIT_FOR_TYPE);
    }

perpetualLockAmount is used to track the total supply of points across all tokens that have been locked perpetually and is essential for properly tracking the total supply. Above we see that the amount of tokens locked increases but perpetualLockAmount is not correctly incremented with the amount being added. This leads to an inaccurate total supply and over allocation of tokens.

#### Lines of Code

[vePeg.sol#L757-L764](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L757-L764)

#### Recommendation

perpetualLockAmount should be incremented by _value.

#### Remediation

Fixed as recommended in commit [9f6591b](https://github.com/hyperstable/contracts/commit/9f6591bc3864e2365e9e9ac50f0bcd86efcb5a0a).

### <br/> [H-02] PositionManager#setLiquidationManager fails to give allowances to new liquidation manager

#### Details 

[PositionManager.sol#L57-L82](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/core/PositionManager.sol#L57-L82)

        function registerVault(address _vaultAddress, uint256 _mcr, uint256 _minDebt, bytes memory _registerData)
            external
            onlyOwner
            returns (uint8)
        {
            uint8 index = lastVaultIndex;

            VaultData memory data;
            data.addr = _vaultAddress;
            data.MCR = _mcr;
            data.asset = IVault(_vaultAddress).asset();
            data.interestIndex = INTEREST_PRECISION;
            data.lastInterestIndexUpdate = block.timestamp;
            data.debtCap = type(uint256).max;
            data.minDebt = _minDebt;

            vaults[index] = data;
            lastVaultIndex = index + 1;

    @>      IVault(_vaultAddress).approve(address(liquidationManager), type(uint256).max);
            IERC20(data.asset).approve(_vaultAddress, type(uint256).max);

            interestRateStrategy.registerVault(index, _registerData);

            return index;
        }

When vaults are registered to the PositionManager, it grants max approval to the liquidation manager for the vault asset. This is essential for the liquidation manager to function.

[PositionManager.sol#L259-L263](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/core/PositionManager.sol#L259-L263)

    function setLiquidationManager(address _newLiquidationManager) external onlyOwner {
        emit NewLiquidationManager(liquidationManager, _newLiquidationManager);

        liquidationManager = _newLiquidationManager;
    }

We notice above that when a new manager is set, no approvals are granted. As a result the new liquidation manager will be unable to functions. This will lead to bad debt that will damage the entire system.

#### Lines of Code

[PositionManager.sol#L259-L263](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/core/PositionManager.sol#L259-L263)

#### Recommendation

`setLiquidationManager` should grant approval to new `LiquidationManager`

#### Remediation

Fixed as recommended in commit [665d860](https://github.com/hyperstable/contracts/commit/665d860637e13ddaec0ed0ab30719779420fe2e0).

### <br/> [H-03] vePeg#delegate lacks any access control allowing all votes to be stolen through delegation

#### Details 

[vePeg.sol#L1389-L1391](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L1389-L1391)

    function delegate(uint256 _from, uint256 _to) public {
        return _delegate(_from, _to);
    }

We see above that when delegating there is no access control. This means that anyone can call this function and re-delegate for anyone else. This allows a malicious user to steal all voting power. They could use this to funnel rewards towards a beneficial pool or use it to vote on governance actions.

#### Lines of Code

[vePeg.sol#L1389-L1391](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L1389-L1391)

#### Recommendation

delegate should check that msg.sender is approved by the owner of from.

#### Remediation

Fix in commit [7466285](https://github.com/hyperstable/contracts/commit/7466285da514c96e755d72366dcee29d9d37a3ad) by completely removing delegation.

### <br/> [M-01] Using a value of 0 for MIN_RATE_AT_TARGET in AdaptiveIRM causes interest rate to reset incorrectly

#### Details 

[AdaptiveIRM.sol#L67-L74](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/libraries/AdaptiveIRM.sol#L67-L74)

        function updateInterestRateAtTarget(AdaptiveIRMStorage storage s, uint8 _vault, int256 _newInterestRateAtTarget)
            internal
        {
    @>      s.endRateAt[_vault] = _newInterestRateAtTarget;
            s.lastUpdate[_vault] = int256(block.timestamp);

            emit UpdateInterestRateAtTarget(_vault, _newInterestRateAtTarget);
        }

When using 0 for MIN_RATE_AT_TARGET, in the event that the interest rate decrease to the minimum then s.endRateAt will be set to zero.

[AdaptiveIRM.sol#L96-L105](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/libraries/AdaptiveIRM.sol#L96-L105)

        int256 err = (utilization - targetUtilization).sDivWad(errNomFactor);

    @>  int256 startRateAtTarget = s.endRateAt[_vault];

        int256 avgRateAtTarget;
        int256 endRateAtTarget;

    @>  if (startRateAtTarget == 0) {
            avgRateAtTarget = INITIAL_RATE_AT_TARGET;
            endRateAtTarget = INITIAL_RATE_AT_TARGET;

The next iteration of the interest rate changes will see this 0 value and it will function as if the vault was never initialized. This will incorrectly set the interest rates to the initial rate rather than charging the 0% interest rate.

#### Lines of Code

[AdaptiveIRM.sol#L84-L120](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/libraries/AdaptiveIRM.sol#L84-L120)

#### Recommendation

MIN_RATE_AT_TARGET should always be a nonzero value

#### Remediation

Fixed as recommended in commit [eb10f23](https://github.com/hyperstable/contracts/commit/eb10f23a8922e0be62c30b023e8129e8a09f61c8).

### <br/> [M-02] vePeg#increase_amount and deposit_for incorrectly revert for perpetual locks

#### Details 

[vePeg.sol#L868-L888](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L868-L888)

        function lock_perpetually(uint256 _tokenId) external nonreentrant {
            assert(_isApprovedOrOwner(msg.sender, _tokenId));

            LockedBalance memory currentLock = locked[_tokenId];
            require(currentLock.perpetuallyLocked == false, "Lock is perpetual");
            require(currentLock.end > block.timestamp, "Lock expired");
            require(currentLock.amount > 0, "Nothing is locked");

            uint256 amount = uint256(int256(currentLock.amount));

            LockedBalance memory newLock;
    @>      newLock.end = 0;
            newLock.perpetuallyLocked = true;
            newLock.amount = currentLock.amount;

            perpetuallyLockedBalance += amount;

            _checkpoint(_tokenId, currentLock, newLock);

    @>      locked[_tokenId] = newLock;
        }

We see above that when a lock is changed to a perpetual lock, that newLock.end is set to 0.

[vePeg.sol#L806-L820](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L806-L820)

        function increase_amount(uint256 _tokenId, uint256 _value) external nonreentrant {
            assert(_isApprovedOrOwner(msg.sender, _tokenId));

            LockedBalance memory _locked = locked[_tokenId];

            assert(_value > 0); // dev: need non-zero value
            require(_locked.amount > 0, "No existing lock found");
    @>      require(_locked.end > block.timestamp, "Cannot add to expired lock. Withdraw");

            if (_locked.perpetuallyLocked) {
                perpetuallyLockedBalance += _value;
            }

            _deposit_for(_tokenId, _value, 0, _locked, DepositType.INCREASE_LOCK_AMOUNT);
        }

The problem is that these tokens can no longer increase in value since 0 is always less than block.timestamp. This leads to issues when users wish to increase the amount or to receive their PEG staking rewards. 

#### Lines of Code

[vePeg.sol#L806-L820](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L806-L820)

[vePeg.sol#L757-L764](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L757-L764)

#### Recommendation

Check should be changed to allow perpetually locked tokens.

#### Remediation

Fixed as recommended in commit [fa26780](https://github.com/hyperstable/contracts/commit/fa26780cccb4edd8ef936ff8fb5d1ba74e46cd99).

### <br/> [M-03] vePeg#_delegate perpetualLocked requirement can easily be bypassed

#### Details 

[vePeg.sol#L1371-L1384](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L1371-L1384)

        function _delegate(uint256 _from, uint256 _to) internal {
            LockedBalance memory currentLock = locked[_from];

    @>      require(currentLock.perpetuallyLocked == true, "Lock is not perpetual");

            address delegator = ownerOf(_from);
            address delegatee = _to == 0 ? address(0) : ownerOf(_to);

            address currentDelegate = delegates(delegator);

            _delegates[delegator] = delegatee;

    @>      _moveAllDelegates(delegator, currentDelegate, delegatee);
        }

We see above that when delegating a token, it is required the from token is perpetually locked. However we see in the lines below that all tokens from the delegator are moved. As a result, all tokens perpetual or not are delegated to the new delegatee. This makes the perpetual check useless as even a small perpetual lock can be used to bypass this check for all tokens.

#### Lines of Code

[vePeg.sol#L1371-L1384](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L1371-L1384)

#### Recommendation

The methodology for delegation should be carefully considered and redesigned.

#### Remediation

Fix in commit [7466285](https://github.com/hyperstable/contracts/commit/7466285da514c96e755d72366dcee29d9d37a3ad) by completely removing delegation.

### <br/> [M-04] PositionManager#setInterestRateStrategy fails to re-register existing vaults with the new strategy

#### Details 

[InterestRateStrategyV1.sol#L43-L59](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/core/InterestRateStrategyV1.sol#L43-L59)

        function registerVault(uint8 _vaultId, bytes memory _registerData) external {
            _onlyPositionManager();

            (uint256 targetUtilization) = abi.decode(_registerData, (uint256));

            AdaptiveIRMStorage storage adaptiveIRM = AdaptiveIRM.getStorage();

            IPositionManager.VaultData memory vaultData = IPositionManager(positionManagerAddress).getVault(_vaultId);

    @>      adaptiveIRM.setTargetUtilization(_vaultId, targetUtilization);

            (, int256 end) = adaptiveIRM.interestRate(_vaultId, 0, 0, vaultData.MCR);

    @>      adaptiveIRM.updateInterestRateAtTarget(_vaultId, end);

            emit VaultRegistered(_vaultId, _registerData);
        }

We see above that when vaults are registered, essential information about the vaults such as target utilization is communicated with the interest rate strategy.

[PositionManager.sol#L96-L104](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/core/PositionManager.sol#L96-L104)

        function setInterestRateStrategy(address _newInterestRateStrategy) external onlyOwner {
            if (_newInterestRateStrategy == address(0)) {
                revert ZeroAddress();
            }

            emit NewInterestRateStrategy(address(interestRateStrategy), address(_newInterestRateStrategy));

    @>      interestRateStrategy = IInterestRateStrategy(_newInterestRateStrategy);
        }

When switching interest rate strategies, notice that this is not done for any of the existing vaults. As a result the new interestRateStrategy will be unable to function correctly.

#### Lines of Code

[PositionManager.sol#L96-L104](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/core/PositionManager.sol#L96-L104)

#### Recommendation

All existing vaults should be re-registered with the new interest rate strategy

#### Remediation

Fixed in commit [eec38e8](https://github.com/hyperstable/contracts/commit/eec38e87cc617941eb65641298f830b349f4dee4) by passing through `_initData`

### <br/> [L-01] PositionManger#setLiquidationManager fails to revoke allowances to replaced liquidation manager

#### Details 

[PositionManager.sol#L259-L263](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/core/PositionManager.sol#L259-L263)

    function setLiquidationManager(address _newLiquidationManager) external onlyOwner {
        emit NewLiquidationManager(liquidationManager, _newLiquidationManager);

        liquidationManager = _newLiquidationManager;
    }

Above we see the liquidation manager is changed but none of the previous allowances are revoked. In the event that the liquidation manager contract has some vulnerability it may still be able to drain the position manager even though it has been changed.

#### Recommendation

All allowances to the previous liquidation manager should be revoked

#### Remediation

Fixed as recommended in commit [665d860](https://github.com/hyperstable/contracts/commit/665d860637e13ddaec0ed0ab30719779420fe2e0).

### <br/> [L-02] InterestRateStrategyV1 has no method to adjust risk parameters such as targetUtilization and MCR

#### Details 

A vaults target utilization and MCR are set on vault registration but there is no way to adjust these parameters at a later time. Given these are key risk factors for a vault it may be useful to allow changing them later.

#### Recommendation

Create a method on InterestRateStrategyV1 to change a vaults target utilization and create a method on PositionManager to adjust a vault's MCR.

#### Remediation

Fixed as recommended in commit [498f00c](https://github.com/hyperstable/contracts/commit/498f00c73aa2e4dd7d93d7142e017f85781f6a62)

### <br/> [L-03] vePeg#unlock_perpetual will redelegate all tokens

#### Details 

[vePeg.sol#L890-L908](https://github.com/hyperstable/contracts/blob/4f650122d3927fd45015c40ad58172508b148876/src/governance/vePeg.sol#L890-L908)

        function unlock_perpetual(uint256 _tokenId) external nonreentrant {
            assert(_isApprovedOrOwner(msg.sender, _tokenId));
            LockedBalance memory currentLock = locked[_tokenId];

            require(currentLock.perpetuallyLocked == true, "Lock is not perpetual");

            uint256 amount = uint256(int256(currentLock.amount));

            LockedBalance memory newLock;
            newLock.end = (block.timestamp + MAXTIME) / WEEK * WEEK;
            newLock.amount = currentLock.amount;

            perpetuallyLockedBalance -= amount;

    @>      _delegate(_tokenId, 0);
            _checkpoint(_tokenId, currentLock, newLock);

            locked[_tokenId] = newLock;
        }

When unlocking a perpetual token, _delegate will remove delegation from ALL tokens including those that are still perpetual.

#### Recommendation

The methodology for delegation should be carefully considered and redesigned.

#### Remediation

Fix in commit [7466285](https://github.com/hyperstable/contracts/commit/7466285da514c96e755d72366dcee29d9d37a3ad) by completely removing delegation.