---
sponsor: "Beta Finance"
slug: "2023-11-betafinance"
date: "2023-12-06"
title: "Beta Finance Invitational"
findings: "https://github.com/code-423n4/2023-11-betafinance-findings/issues"
contest: 303
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Beta Finance smart contract system written in Solidity. The audit took place between November 1—November 6 2023.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 5 wardens contributed reports to Beta Finance.

  1. [ladboy233](https://code4rena.com/@ladboy233)
  2. [T1MOH](https://code4rena.com/@T1MOH)
  3. [0xStalin](https://code4rena.com/@0xStalin)
  4. [bin2chen](https://code4rena.com/@bin2chen)
  5. [dirk\_y](https://code4rena.com/@dirk_y)


This audit was judged by [cccz](https://code4rena.com/@cccz).

Final report assembled by PaperParachute.

# Summary

The C4 analysis yielded an aggregated total of 5 unique vulnerabilities. Of these vulnerabilities, 0 received a risk rating in the category of HIGH severity and 5 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 5 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Beta Finance repository](https://github.com/code-423n4/2023-11-betafinance), and is composed of 8 smart contracts written in the Solidity programming language and includes 999 lines of Solidity code.

In addition to the known issues identified by the project team, an [Automated Findings report](https://gist.github.com/JustDravee/c830761b3625c972499e279782dcb508) was generated using the [4naly3er bot](https://github.com/Picodes/4naly3er) and all findings therein were classified as out of scope.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Medium Risk Findings (5)
## [[M-01] Borrower can abuse enterMarkets to force liquidator can pay more funds](https://github.com/code-423n4/2023-11-betafinance-findings/issues/37)
*Submitted by [ladboy233](https://github.com/code-423n4/2023-11-betafinance-findings/issues/37)*

https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L331

https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L232

https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniToken.sol#L81

https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L96

Borrower can abuse enterMarkets to force liquidator can pay more fund.

### Proof of Concept

Liquidation process is in place to make sure the bad debt is paid, and when the liquidator repays the debt, they can seize the asset of the borrower as reward.

However, a bad user who does not want to repay the debt can force liquidator to pay more funds and even block liquidation.

When liquidating, this [line of code](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L331) is called:

```solidity
Evaluation memory evalBefore = _evaluateAccountInternal(_params.targetAccountId, poolMarkets, targetAccount);
```

For every pool market, the liquidator needs to pay the gas to call [accrue interest](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L232)

```solidity
    function _evaluateAccountInternal(bytes32 _accountId, address[] memory _poolMarkets, AccountInfo memory _account)
        internal
        returns (Evaluation memory eval)
    {
        ModeConfiguration memory mode;
        if (_account.modeId != 0) { mode = modeConfigurations[_account.modeId]; }
        for (uint256 i = 0; i < _poolMarkets.length; ++i) {
            // Accrue interest for all borrowable markets
            IOmniToken(_poolMarkets[i]).accrue();
        }
```

Note the function call:

```solidity
 IOmniToken(_poolMarkets[i]).accrue();
```

For each pool market, at most the for accrue function for loop runs [255 interaction](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniToken.sol#L81)

```solidity
uint8 trancheIndex = trancheCount;
uint256 totalBorrow = 0;
uint256 totalDeposit = 0;
uint256[] memory trancheDepositAmounts_ = new uint256[](trancheCount);
uint256[] memory trancheAccruedDepositCache = new uint256[](trancheCount);
while (trancheIndex != 0) {
	unchecked {
		--trancheIndex;
	}
```

The max trancheCount is uint8, but note there is no check for borrower to [add pool market](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L96) any time by calling enterMarkets.

The enterMarkets does not restrict the max number of market entered and does not validate if the borrower already has borrow position.

Before the liquidation happens, the borrower can select a lot of markets with high tranche count that does not accrue interest yet to enter the market.

Then liquidator has to pay the gas to run the for loop of accruing, which is clearly a loss of fund for liquidator.

The number of for loop iteration is :

```solidity
number of pool market added by borrower * tranche count
```

This is an unbounded loop and can exceed the block limit.

The liquidator may not have incentive to liquidate once the gas paid for accruing exceeds the bonus, then bad debt is accruing and make the pool insolvent.

Proof of Concept:

Can add this test to TestOmniPool.t.sol

<details>

```solidity
     function test_LiquidateNoIsolated_poc() public {

        uint256 length = 1000;
        address[] memory newMarkets = new address[](length);

        uint256[] memory borrowCaps = new uint256[](3);
        borrowCaps[0] = 1e9 * (10 ** uToken.decimals());
        borrowCaps[1] = 1e3 * (10 ** uToken.decimals());
        borrowCaps[2] = 1e2 * (10 ** uToken.decimals());

        for (uint256 i; i < length; ++i) {
            OmniToken oooToken = new OmniToken();
            oooToken.initialize(address(pool), address(uToken), address(irm), borrowCaps);
            IOmniPool.MarketConfiguration memory mConfig1 =
            IOmniPool.MarketConfiguration(0.9e9, 0.9e9, uint32(block.timestamp + 1000 days), 0, false);
            pool.setMarketConfiguration(address(oooToken), mConfig1);
            newMarkets[i] = address(oooToken);
        }

        test_SetLiquidationBonus();
        IIRM.IRMConfig[] memory configs = new IIRM.IRMConfig[](3);
        configs[0] = IIRM.IRMConfig(0.01e9, 1e9, 1e9, 1e9);
        configs[1] = IIRM.IRMConfig(0.85e9, 0.02e9, 0.08e9, 1e9);
        configs[2] = IIRM.IRMConfig(0.8e9, 0.03e9, 0.1e9, 1.2e9);
        uint8[] memory tranches = new uint8[](3);
        tranches[0] = 0;
        tranches[1] = 1;
        tranches[2] = 2;
        irm.setIRMForMarket(address(oToken), tranches, configs);
        oToken.deposit(0, 2, 1e2 * 1e18);
        vm.startPrank(ALICE);

        oToken.deposit(0, 0, 0.1e2 * 1e18);
        oToken.deposit(0, 1, 0.1e2 * 1e18);
        oToken.deposit(0, 2, 0.8e2 * 1e18);
        address[] memory markets = new address[](1);
        markets[0] = address(oToken);
        pool.enterMarkets(0, markets);
        pool.borrow(0, address(oToken), 0.9216e2 * 1e18);
      
        // pool.enterMarkets(0, newMarkets);

        vm.stopPrank();
        vm.warp(365 days);

        pool.enterMarkets(0, markets);

        uint256 gas = gasleft();
        uint256[] memory seizedShares = pool.liquidate(
            IOmniPool.LiquidationParams(
                address(ALICE).toAccount(0),
                address(this).toAccount(1),
                address(oToken),
                address(oToken),
                0.2e2 * 1e18
            )
        );
        console.log("gas used: ", gas - gasleft());
  
    }
```

</details>

Basically, we construct 1000 markets.

First we comment out the bad user's:

```solidity
// pool.enterMarkets(0, newMarkets);
```

Then run the test:

```solidity
forge test -vv --match-test "test_LiquidateNoIsolated_poc"
```

The output is:

```solidity
Running 1 test for src/tests/TestOmniPool.t.sol:TestOmniPool
[PASS] test_LiquidateNoIsolated_poc() (gas: 3177349635)
Logs:
  gas used:  159429
```

This means the liquidator pays 159429 amount of gas to liquidate the user, but if we uncomment the line of code "pool.enterMarkets(0, newMarkets)", the liquidator is forced to call accrue thousands of times in the loop.

We run the test again using the same comment.

The gas used is:

```solidity
Running 1 test for src/tests/TestOmniPool.t.sol:TestOmniPool
[PASS] test_LiquidateNoIsolated_poc() (gas: 3207995288)
Logs:
  gas used:  31572426
```

If the user call enter markets to enter more market, liquidator is likely to run out of gas.

As long as the user makes the liquidator feel like the gas cost of calling accrue is greater than liquidation seized asset, there will be no liquidation.

### Recommended Mitigation Steps

Validate the max number of entered market for borrower.

Do not allow user to add more market if the user has borrow position.

**[cccz (Judge) increased severity to Medium](https://github.com/code-423n4/2023-11-betafinance-findings/issues/37)**

**See full discussion on severity [here](https://github.com/code-423n4/2023-11-betafinance-findings/issues/19#issuecomment-1798303605).**

**[allenjlee (BetaFinance) confirmed](https://github.com/code-423n4/2023-11-betafinance-findings/issues/19#issuecomment-1807167104)**

***

## [[M-02] Users can't repay their debts if the OmniPool contract is paused which can cause users to fall into liquidation and lose their collateral](https://github.com/code-423n4/2023-11-betafinance-findings/issues/32)
*Submitted by [0xStalin](https://github.com/code-423n4/2023-11-betafinance-findings/issues/32), also found by ladboy233 ([1](https://github.com/code-423n4/2023-11-betafinance-findings/issues/34), [2](https://github.com/code-423n4/2023-11-betafinance-findings/issues/5))*

Users can't repay their debts if the OmniPool contract is paused which can cause users to fall into liquidation and lose their collateral

### Proof of Concept

The [`OmniPool::repay()` function](https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniPool.sol#L303-L310) has implemented the [`whenNotPaused` modifier](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/49c0e4370d0cc50ea6090709e3835a3091e33ee2/contracts/security/Pausable.sol#L44-L47), which will prevent the function from being used if the contract is paused. The problem is that the usage of this function should not be prevented because if users are unable to repay their debts, their accounts can fall into liquidation status while the OmniPool contract is paused, and once the contract is unpaused, and liquidations are enabled too, if the account felt into liquidation status, now the users and liquidators will be in a mev run to either repay the debt or liquidate the collateral.

This presents an unnecessary risk to users by preventing them from repaying their debts.

> OmniPool contract

```solidity
//@audit-issue -> If contract is paused, this function can't be called if the contract is paused because of the whenNotPaused modifier!
function repay(uint96 _subId, address _market, uint256 _amount) external whenNotPaused {
    ...
}
```
### Recommended Mitigation Steps

The mitigation is very straight forward, don't disable the borrower's repayments, and don't interrupt the repayments. Remove the whenNotPaused modifier:

> OmniPool contract

```solidity
- function repay(uint96 _subId, address _market, uint256 _amount) external whenNotPaused {
//@audit-ok => Allow repayments even if the contract is paused!
+ function repay(uint96 _subId, address _market, uint256 _amount) external {
    ...
}
```
**[allenjlee (BetaFinance) confirmed and commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/32#issuecomment-1807208128):**
 > We will remove the `whenNotPaused` modifier for `repay`

**[cccz (Judge) commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/32#issuecomment-1808342436):**
 > I think the root cause of this issue is that users are not able to repay when paused and the interest accrued may lead to the user's account being unhealthy.

***

## [[M-03] paucheTranche state can be set to arbitrary value](https://github.com/code-423n4/2023-11-betafinance-findings/issues/27)
*Submitted by [ladboy233](https://github.com/code-423n4/2023-11-betafinance-findings/issues/27)*

<https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L348> 

<https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniToken.sol#L180>

paucheTranche state can be set to arbitrary value

### Proof of Concept

The protocol has this concept of tranche id and borrower tier, the higher borrower tier means higher risk.

Lower borrower tier means lower risk.

When liquidation happens, if the borrowTrueValue is greater than depositTrueValue, which can happen because of the underlying oracle price can change:

```solidity
if (evalAfter.borrowTrueValue > evalAfter.depositTrueValue) {
	pauseTranche = borrowTier;
	emit PausedTranche(borrowTier);
}
```

When the pauseTranche id is set to the borrower tier in this [line of code](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniPool.sol#L348), the user that deposited below the paused tranche id should not be able to withdraw / tranche id share because [this check](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniToken.sol#L180) is in-place:

    require(_trancheId < IOmniPool(omniPool).pauseTranche(), "OmniToken::withdraw: Tranche paused.");

The problem is that the paucheTranche state can be set to arbitrary value.

Suppose a user has borrower tier 10 and borrowTrueValue exceed depositTrueValue, the pauseTranche is set to 10, then a second user has borrower tier 0, he is subject to liquidation as well.

borrowTrueValue exceed depositTrueValue as well.

The pauseTranche is reset to 0 again, which defeats the check of:

    require(_trancheId < IOmniPool(omniPool).pauseTranche(), "OmniToken::withdraw: Tranche paused.");

### Recommended Mitigation Steps

Change the code to:

    if (evalAfter.borrowTrueValue > evalAfter.depositTrueValue) {
    	pauseTranche = max(borrowTier, pauseTranche)
    	emit PausedTranche(borrowTier);
    }

**[cccz (Judge) commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/27#issuecomment-1797832516):**
 > Nice Find!

> `pauseTranche` is used to disallow the user from calling `withdraw()` to withdraw assets before `socializeLoss()` is called.
> As for severity, I'm not so sure it's a High, need more thought on that.

**[Ladboy233 (Warden) commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/27#issuecomment-1798287715):**
> I submitted as High because this finding allows user to perform privilege escalation and act as admin to pause arbitrary tranche.
> 
> Users can select a tranche and deposit and borrow and liquidation themselves to set pauseTranche to any value.
> 
> Unless admin kept a step into to unpause, all other user's withdraw transaction can revert because of this check.
> 
> ```solidity
> require(_trancheId < IOmniPool(omniPool).pauseTranche(), "OmniToken::withdraw: Tranche paused.");
> ```


**[cccz (Judge) decreased severity to Medium and commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/27#issuecomment-1799060067):**
 > This requires not only that the user can be liquidated, but also that the bad debt arises (there is big gap between the two), so I think it would be very difficult for an attacker to actively exploit this issue to set pauseTranche.

> And as `TRST-M-1 Users can avoid losses by withdrawing assets before pausing due to bad debt` noted, users can withdraw preemptively to prevent socializeLoss() from causing losses.

 > Users can't actively arise bad debt, medium likelihood.

> Users can withdraw their assets before socialize loss, medium impact.

> So consider it medium severity.

**[allenjlee (BetaFinance) confirmed and commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/27#issuecomment-1807187753):**
 > Agree with judge's assessment, medium severity. There should be no loss to users, and this situation arises in the case only where a higher tranche has bad debt after a lower tranche has bad debt. The worst case scenario, is that one depositor is left with all the bad debt, which is unintended by the protocol but does not cause loss of funds compared to existing protocols -- which rely on manual pause intervention, which the protocol also has as we inherit `Pausable`, so this is medium severity. 
> 
> We will fix to the latest recommendation, which is using the `pauseTranche = borrowTier > pauseTranche ? pauseTranche : borrowTier;`

***

## [[M-04] Users pay higher fee than intended](https://github.com/code-423n4/2023-11-betafinance-findings/issues/13)
*Submitted by [T1MOH](https://github.com/code-423n4/2023-11-betafinance-findings/issues/13)*

Protocol mints incorrect depositAmount and depositShare to protocol. Such that reserveFee is higher than defined. Suppose following scenario:

1.  Tranche 2 has 20% APR, has 5\_000 borrowed
2.  Tranche 1 has 10% APR, has 10\_000 borrowed
3.  ReserveFee is 10%
4.  It means that reserveFee that must be paid after 1 year is `5_000 * 20% * 10% + 10_000 * 10% * 10% = 200`. However current implementation will calculate 208, which will be shown PoC.

Lender pays this extra fee.

### Proof of Concept

Here is gist with all tests <https://gist.github.com/T1MOH593/34729b5333fe43eb58cf8b4948ef137f>

First part: Show that issue exists. It is shown in `custom_test1()`.

Second part: Explaining the issue.

Let's set initial values, for example:

Tranche 2 has 20% APR, 5\_000 borrowed, 20\_000 deposited; Tranche 1 has 10% APR, 10\_000 borrowed, 20\_000 depositeed. And we use 1 year as time difference to ease calculations.

Tranche struct has 2 types of variables, let's focus regarding deposit: `totalDepositAmount` and `totalDepositShare`. Initially without accrued interest they equal, 1 : 1, in above scenario will be 20\_000 in both tranches. As the time passes, interest accrues. It means that `totalDepositAmount` increases. If reserveFee is 0%, that's it - for example 1000 of interests accrued, `totalDepositAmount` is increased by 1000.

However in case 10% of interest must be paid to reserve. It means that extra shares will be minted increasing `totalDepositShare` AND `totalDepositAmount` should be also increased because otherwise user's balance will decrease. Let's take delta of `totalDepositAmount` as X and `totalDepositShare` as Y. And calculate these values for above scenario.

*   Total interest for Tranche2 is `5_000 * 20% = 1_000`. For Tranche1 is `10_000 * 10% = 1_000`
*   Interest for Tranche 2 stays in this tranche. Interest for Tranche 1 is divided between tranche1 and tranche2 based on deposit amounts, such that tranche1 receives `1000 * 20_000 / 40_000 = 500`, tranche2 receives 500 too. Finally tranche2 has 1500 of interest, tranche1 has 500 of interest
*   Reserve fee must be paid. `1500 * 10% = 150` in tranche2, and `500 * 10% = 50` in tranche1
*   We need to mint such amount of share Y to Reserve and increase `totalDepositAmount` by X so that Reserve balance is 150 AND User's balance is `20_000 + 1500 * 90% = 21350` in Tranche2. So let's create system of equations.

<!---->

    (20_000 / (20_000 + Y)) * (20_000 + X) = 21_350 is balance of User
    Y / (20_000 + Y) * (20_000 + X) = 150 is balance of Reserve

Solving this we get `X = 1500`, `Y = 140.51`. I.e. 140.51 of shares must be minted to Reserve, and `totalDepositAmount` must be increased by 1500 (amount of accrued interest) in Tranche2.
Now let's calculate accordingly for Tranche1:

    (20_000 / (20_000 + Y)) * (20_000 + X) = 20_450 is balance of User
    Y / (20_000 + Y) * (20_000 + X) = 50 is balance of Reserve

`X = 500`, `Y = 48.89`.

However current implementation mints 100 shares to Reserve and increases `totalDepositAmount` by 1450 in Tranche 2; Also mints 100 shares to reserve and increases `totalDepositAmount` by 550 in Tranche1 as shown in test `custom_test2`

This is the core issue: Code always mints amount of shares equal to reserveInterest to Reserve (100 in this example) and calculates sub-optimal amount of depositAmount to increase.

### Tools Used

Foundry

### Recommended Mitigation Steps

Fix is not obvious at all, it's the most difficult part of report to be honest.
Algorithm should be completely refactored, should be implemented following accounting:

*   `totalDepositAmount` should be increased by the amount of accrued interest allocated to this tranche. From above example it's 1500 for Tranche2 and 500 for Tranche1
*   Number of shares minted to reserve in tranche should be calculated as solution of one of these equations

<!---->

    1) (totalDepositSharesOfTranceBeforeAccruing / (totalDepositSharesOfTranceBeforeAccruing + Y)) * (totalDepositAmountBeforeAccruing + interestAmountAllocatedToThisTrance) = totalDepositSharesOfTranceBeforeAccruing + interestAmountAllocatedToThisTrance * 0.9
    If we paste values from above example with Tranche 2, we get (20_000 / (20_000 + Y)) * (20_000 + 1_500) = 21_350
    2) (Y / (totalDepositSharesOfTranceBeforeAccruing + Y)) * (totalDepositAmountBeforeAccruing + interestAmountAllocatedToThisTrance) = interestAmountAllocatedToThisTrance * 0.9
    If we paste values from above example with Tranche 2, we get (Y / (20_000 + Y)) * (20_000 + 1_500) = 150

And also write tests to check reserveFee amounts, now they absent

**[cccz (Judge) commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/13#issuecomment-1798846556):**
 > I've noticed this issue before, the reason for it is that the `reserveShare` is calculated by dividing by `trancheDepositAmount_` instead of `trancheDepositAmount_+interestAmountProportion`, so the calculated reserveShare will be larger.
> ```solidity
>                 tranche.totalDepositAmount = trancheDepositAmount_ + interestAmountProportion + reserveInterestAmount;
>                 tranche.totalBorrowAmount = trancheBorrowAmount_ + depositInterestAmount + reserveInterestAmount;
>             }
> 
>             // Pay reserve fee
>             uint256 reserveShare;
>             uint256 totalDepositShare_ = tranche.totalDepositShare;
>             if (trancheDepositAmount_ == 0) {
>                 reserveShare = reserveInterestAmount;
>             } else {
>                 reserveShare = (reserveInterestAmount * totalDepositShare_) / trancheDepositAmount_; // Cannot divide by 0
>             }
>             trancheAccountDepositShares[trancheIndex][reserveReceiver] += reserveShare;
>             tranche.totalDepositShare = totalDepositShare_ + reserveShare;
> ```
> A simple example is
> ```
> Tranche 1 has 5% APR, has 10000 borrowed
> ReserveFee is 10%.
> interestAmount = 10000 * 5% = 500
> reserveInterestAmount = 500 * 10% = 50
> interestAmountProportion = 500 * 90% = 450
> 
> cur: reserveShare = 50 * 10000 / 10000 = 50
> 
>  reserveAmount = 50 * (10000+500) / (1000+50) = 52.24
> 
> fix: reserveShare = 50 * 10000 / (10000+450) = 47.847
> 
>  reserveAmount = 47.847 * (10000+500) / (10000+47.847) = 50
> ```
> I previously thought this was intentional because it was to handle the case where `interestAmountProportion` wouldn't be distributed when trancheDepositAmount_ = 0.
> But thinking about it again, it's just a simple fix
> ```diff
>             if (trancheDepositAmount_ == 0) {
> -               reserveShare = reserveInterestAmount;
> +               reserveShare = reserveInterestAmount + interestAmountProportion;
>             } else {
> -               reserveShare = (reserveInterestAmount * totalDepositShare_) / trancheDepositAmount_; // Cannot divide by 0
> +               reserveShare = (reserveInterestAmount * totalDepositShare_) / trancheDepositAmount_ + interestAmountProportion; // Cannot divide by 0
>             }
> ```

**[allenjlee (BetaFinance) confirmed and commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/13#issuecomment-1807164106):**
 > I believe this is a valid issue, we have resolved the issue and reserve payments are correct now. See screenshot from the PoC test case.
> 
> ![image](https://user-images.githubusercontent.com/131902879/285870411-e3d41334-8fdb-4284-86c3-bf8b44d12120.png)
> 
***

## [[M-05] accrue interest function is likely failed to accrue interest for token with low decimal](https://github.com/code-423n4/2023-11-betafinance-findings/issues/8)
*Submitted by [ladboy233](https://github.com/code-423n4/2023-11-betafinance-findings/issues/8)*

Loss of precision is too high when accruing interest

### Proof of Concept

When interest accrues, [we are calling](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/OmniToken.sol#L104)

```solidity
uint256 interestAmount;
{
	uint256 interestRate = IIRM(irm).getInterestRate(address(this), trancheIndex, totalDeposit, totalBorrow);
	interestAmount = (trancheBorrowAmount_ * interestRate * timePassed) / 365 days / IRM_SCALE;
}
```

Note that the loss of precision is too high:

```solidity
	interestAmount = (trancheBorrowAmount_ * interestRate * timePassed) / 365 days / IRM_SCALE;
```

We are dividing 365 days and then divde by IRM_SCALE (which is 1e9)

Combing the fact that the IRM_SCALE is hardcoded to 1e9

Also, the time passed between two accruing is small,

Also, the underlying tokens have low decimals (6 decimals, even two decimals).

The interest accrued will be always rounded to 0.

In tests/MockERC20.sol,

[The decimal](https://github.com/code-423n4/2023-11-betafinance/blob/0f1bb077afe8e8e03093c8f26dc0b7a2983c3e47/Omni_Protocol/src/tests/mock/MockERC20.sol#L9) is hardcoded to 18

If we change the hardcode decimal to 6 and we add the POC below in [TestOmniToken.t.sol](https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/tests/TestOmniToken.t.sol)

```solidity
    function test_Accrue_POC_Low_decimal() public {
        setUpBorrow();
        (uint256 tda0, uint256 tba0,,) = oToken.tranches(0);
        (uint256 tda1, uint256 tba1,,) = oToken.tranches(1);
        (uint256 tda2, uint256 tba2,,) = oToken.tranches(2);
        uint256 td = tda0 + tda1 + tda2;

        uint256 borrowAmount = 10 * (10 ** uToken.decimals());
        IOmniPool(pool).borrow(0, address(oToken), borrowAmount);

        uint256 time_elapse = 120 seconds;
        vm.warp(time_elapse);

        uint256 interestAmount;
        {
            uint256 interestRate = irm.getInterestRate(address(oToken), 0, td, borrowAmount);
            interestAmount = borrowAmount * interestRate * time_elapse / 365 days / oToken.IRM_SCALE();
        }
        uint256 feeInterestAmount = interestAmount * oToken.RESERVE_FEE() / oToken.FEE_SCALE();
        interestAmount -= feeInterestAmount;

        oToken.accrue();

        vm.warp(time_elapse * 2);

        oToken.accrue();

        vm.warp(time_elapse * 4);

        oToken.accrue();
    
    }
```

We are now accruing the interest every two minutes

Before running the POC we import the

```solidity
import "forge-std/console.sol";
```

In OmniToken.sol

and add the console.log to log the interest

```solidity
{
	uint256 interestRate = IIRM(irm).getInterestRate(address(this), trancheIndex, totalDeposit, totalBorrow);
	// @audit
	// this can be trancated to 0
	interestAmount = (trancheBorrowAmount_ * interestRate * timePassed) / 365 days / IRM_SCALE;
	console.log("interestAmount", interestAmount);
}
```

We then run the POC:

```solidity
forge test -vv --match-test "test_Accrue_POC_Low_decimal"
```

The output is:

```solidity
Running 1 test for src/tests/TestOmniToken.t.sol:TestOmniToken
[PASS] test_Accrue_POC_Low_decimal() (gas: 737833)
Logs:
  interestAmount 0
  interestAmount 0
  interestAmount 0
```

This mean within every two minutes time elapse, the interest is round down to 0 and the accrue function is called in every borrow / withdraw / transfer, so the interest will be rounded down heavily.

Alternatively, user can keep calling accrue for every two minutes (or x minutes) to avoid paying interest.

### Recommended Mitigation Steps

Modify the interest, do not do multiplication before division to avoid precision loss.

Do not hardcode  IRM_SCALE and consider token decimals when accruing interest

**[allenjlee (BetaFinance) confirmed and commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/8#issuecomment-1805438448):**
 > We acknowledge this issue happens when values are small. I believe that the loss of precision causing 0 rounding is mainly from the 365 days value, which is equivalent to 31,536,000 seconds (a little over 7 decimals of precision loss). For the interest rate, let's say -- worst case -- the lowest interest rate is 0.1%, which would be 3 decimals of precision loss as well. Therefore, there is a divisor of 31,536,000,000 every second, where the borrow amount must exceed this in the worst case. For tokens with greater than 11 decimals, there should be no issues. For tokens with less than 11 decimals, there will be issues.
> 
> For USDC, this means there must be greater than 31,536 USDC per tranche being actively borrowed otherwise the interest rounds to 0 (assuming `accrue()` is called every second). If we relax this assumption, and say `accrue()` is called once every 10 seconds, the requirement becomes 3,153.60 USDC per tranche. Assuming it's below the 31,536 USDC threshold, this would mean ~31.5 USDC is lost in interest every year. 
> 
> Similarly, for USDT (w/ 8 decimals) it will require greater than 315.36 (1 second) and 3.1536 (10 seconds) USDT per tranche being actively borrowed, otherwise interest rounds to 0. 
> 
> For WBTC, this would also mean that we would need a 1% base interest rate for it to be 0.31536 (10 second block time) to compensate for no interest. The loss on this (assuming BTC price is $100K) would also be $300 in interest lost. Practically the value is small.
> 
> We think refactoring the code to address this issue would be quite challenging and introduce potential problems/edge cases and the changes outweigh the problems solved, so we will instead make it aware in our documentation that there is this limitation. We will also be mindful of setting the configurations for the asset IRMs. Thank you for bringing up this issue, we agree with the severity.

***

# Low Risk and Non-Critical Issues

For this audit, 5 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2023-11-betafinance-findings/issues/23) by **T1MOH** received the top score from the judge.

*The following wardens also submitted reports: [0xStalin](https://github.com/code-423n4/2023-11-betafinance-findings/issues/33), [ladboy233](https://github.com/code-423n4/2023-11-betafinance-findings/issues/28), [bin2chen](https://github.com/code-423n4/2023-11-betafinance-findings/issues/3), and [dirk\_y](https://github.com/code-423n4/2023-11-betafinance-findings/issues/12).*

## 1. OmniOracle.sol doesn't work with tokens returning symbol as bytes32
https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniOracle.sol#L48

Some tokens (e.g. MKR) have metadata fields (name / symbol) encoded as bytes32 instead of the string prescribed by the ERC20 specification.
Thus such tokens can't be used in Band Oracle
```solidity
    function getPrice(address _underlying) external view returns (uint256) {
        OracleConfig memory config = oracleConfigs[_underlying];
        if (config.provider == Provider.Band) {
            IStdReference.ReferenceData memory data;
            if (_underlying == WETH) {
                data = IStdReference(config.oracleAddress).getReferenceData("ETH", USD);
            } else {
@>              data = IStdReference(config.oracleAddress).getReferenceData(IERC20Metadata(_underlying).symbol(), USD);
            }
            ...
    }
```
### Recommended Mitigation Steps
Use the BoringCrypto safeSymbol() function code with the returnDataToString() parsing function to handle the case of a bytes32 return value: https://github.com/boringcrypto/BoringSolidity/blob/ccb743d4c3363ca37491b87c6c9b24b1f5fa25dc/contracts/libraries/BoringERC20.sol#L15-L39

## 2. Restrict updating isIsolatedCollateral from true to false and vice-versa

https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniPool.sol#L508-L522

Current implementation allows to set new config. In case normal market will be marked isolated or vice versa, it will break internal accounting of OmniPool.sol. Because code assumes that this variable will never change

### Recommended Mitigation Steps
Explicitly disallow updating `isIsolatedCollateral` on configured markets

## 3. Remove dust deposit amount after socializing loss

https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/OmniPool.sol#L397

There can be dust amount of deposit `socializeLoss` is called
```solidity
    function socializeLoss(address _market, bytes32 _account) external onlyRole(DEFAULT_ADMIN_ROLE) {
        uint8 borrowTier = getAccountBorrowTier(accountInfos[_account]);
        Evaluation memory eval = evaluateAccount(_account);
        uint256 percentDiff = eval.depositTrueValue * 1e18 / eval.borrowTrueValue;
@>      require(percentDiff < 0.00001e18, "OmniPool::socializeLoss: Account not fully liquidated, please call liquidate prior to fully liquidate account.");
        IOmniToken(_market).socializeLoss(_account, borrowTier);
        emit SocializedLoss(_market, borrowTier, _account);
    }
```

However it socializes all debt of user and leaves deposit to the user. But it should firstly reduce user's debt by that deposit, and then to socialize debt.
```solidity
    function socializeLoss(bytes32 _account, uint8 _trancheId) external nonReentrant {
        require(msg.sender == omniPool, "OmniToken::socializeLoss: Bad caller");
        uint256 totalDeposits = 0;
        for (uint8 i = _trancheId; i < trancheCount; ++i) {
            totalDeposits += tranches[i].totalDepositAmount;
        }
        OmniTokenTranche storage tranche = tranches[_trancheId];
        uint256 share = trancheAccountBorrowShares[_trancheId][_account];
@>      //@audit It is debt amount which must be socialized
        uint256 amount = Math.ceilDiv(share * tranche.totalBorrowAmount, tranche.totalBorrowShare); // Represents amount of bad debt there still is (need to ensure user's account is emptied of collateral before this is called)
        uint256 leftoverAmount = amount;
@>      //@audit full amount is socialized
        for (uint8 ti = trancheCount - 1; ti > _trancheId; --ti) {
            OmniTokenTranche storage upperTranche = tranches[ti];
            uint256 amountProp = (amount * upperTranche.totalDepositAmount) / totalDeposits;
            upperTranche.totalDepositAmount -= amountProp;
            leftoverAmount -= amountProp;
        }
        tranche.totalDepositAmount -= leftoverAmount;
        tranche.totalBorrowAmount -= amount;
        tranche.totalBorrowShare -= share;
        trancheAccountBorrowShares[_trancheId][_account] = 0;
        emit SocializedLoss(_account, _trancheId, amount, share);
    }
```
### Recommended Mitigation Steps
Subtract user's deposit before socializing debt
```diff
    function socializeLoss(bytes32 _account, uint8 _trancheId) external nonReentrant {
        require(msg.sender == omniPool, "OmniToken::socializeLoss: Bad caller");
        uint256 totalDeposits = 0;
+       uint256 userDepositShares = 0;
        for (uint8 i = _trancheId; i < trancheCount; ++i) {
            totalDeposits += tranches[i].totalDepositAmount;
+           userDepositShares += trancheAccountDepositShares[i][_account];
        }
        OmniTokenTranche storage tranche = tranches[_trancheId];
        uint256 share = trancheAccountBorrowShares[_trancheId][_account];
        uint256 amount = Math.ceilDiv(share * tranche.totalBorrowAmount, tranche.totalBorrowShare); // Represents amount of bad debt there still is (need to ensure user's account is emptied of collateral before this is called)
+       
+       amount -= userDepositShares * tranche.totalDepositAmount / tranche.totalDepositShare;
+       
        uint256 leftoverAmount = amount;
        for (uint8 ti = trancheCount - 1; ti > _trancheId; --ti) {
            OmniTokenTranche storage upperTranche = tranches[ti];
            uint256 amountProp = (amount * upperTranche.totalDepositAmount) / totalDeposits;
            upperTranche.totalDepositAmount -= amountProp;
            leftoverAmount -= amountProp;
        }
        tranche.totalDepositAmount -= leftoverAmount;
        tranche.totalBorrowAmount -= amount;
        tranche.totalBorrowShare -= share;
        trancheAccountBorrowShares[_trancheId][_account] = 0;
        emit SocializedLoss(_account, _trancheId, amount, share);
    }
```

## 4. `toAddress()` logic can be simpler without `ADDRESS_MASK`
https://github.com/code-423n4/2023-11-betafinance/blob/main/Omni_Protocol/src/SubAccount.sol#L27

### Recommended Mitigation Steps
```diff
    function toAddress(bytes32 _account) internal pure returns (address) {
-       return address(uint160(uint256(_account) & ADDRESS_MASK));
+       return address(uint160(uint256(_account)));
    }
```

## 5. Remove unused variable from struct `ModeConfiguration`
Field `modeMarketCount` is never used, consider removing it

**[allenjlee (BetaFinance) confirmed and commented](https://github.com/code-423n4/2023-11-betafinance-findings/issues/23#issuecomment-1806702947):**
 > Good quality QA report. All good points.

> 1. I think to handle issues w/ symbol better we should create an explicit mapping in our protocol, e.g. `mapping(address => string) public underlyingSymbols`. This should handle these issues as well as other potential issues w/ token symbol changes, etc.
> 
> 2. added:
> ```
> MarketConfiguration memory currentConfig = marketConfigurations[_market];
> if (currentConfig.borrowFactor != 0 || currentConfig.collateralFactor != 0) {
>     require(
>         _marketConfig.isIsolatedCollateral == currentConfig.isIsolatedCollateral,
>         "OmniPool::setMarketConfiguration: Cannot change isolated collateral status."
>     );
> }
> ```
> 
> 3. I think this is a fair point, tried to be practical here and accept some rounding issues when liquidating s.t. there's a $1000 left on $10 million debt value (hopefully there's never $10 million bad debt). I think the mitigation should be slightly adjusted though, as the `userDepositShare` could result in a different amount depending on each `tranche`. So would need to sum the amounts across each tranche then subtract that amount.
> 
> 4. Good point.
> 
> 5. Also good point.

***
# Disclosures

C4 is an open organization governed by participants in the community.

C4 Audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
