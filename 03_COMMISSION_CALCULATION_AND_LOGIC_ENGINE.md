# ETSWalletV2 Affiliate Management System: Commission Calculation & Logic Engine

**Document Version:** 1.0.0  
**Target Audience:** QA Engineers, Backend Developers, Financial Analysts, Operations Auditors  
**Related Documents:**
- `01_BUSINESS_REQUIREMENTS_SPECIFICATION.md`
- `02_TECHNICAL_ARCHITECTURE_AND_DATA_MODELS.md`
- `07_QA_TESTING_AND_VERIFICATION_MATRIX.md`

---

## 1. Calculation Pipeline Overview

The commission calculation engine is orchestrated by `AffiliateCommissionCalculationService` and `AffiliateCommissionCarryForwardService`. The engine runs on a monthly billing period cycle (e.g., `2026-08`), processing player betting transactions, payment processing fees, provider royalties, tiered commission hurdles, and multi-stage 90-day GGR rollover.

```mermaid
flowchart TD
    Start(["Start Period Calculation\n(AffiliateId, Period: YYYY-MM)"]) --> FetchBets["1. Ingest Settled Bets\n(DepositBet, DepositWin, BonusBet, BonusWin)"]
    FetchBets --> CalcGGR["2. Compute Current Company GGR\nDepositGGR = DepositBet - DepositWin"]
    CalcGGR --> FetchFees["3. Ingest Financial Fees\nDeposit Fees + Withdraw Fees"]
    FetchFees --> CycleEval["4. Evaluate 90-Day Staged Cycle\nCycleNumber, Stage (1, 2, or 3), IsNewCycle"]
    
    CycleEval --> CheckNewCycle{Is New Cycle?\n(Stage == 1)}
    CheckNewCycle -- Yes --> ExpirePrev["Expire Previous Cycle Records\nReset PreviousCarriedGGR = 0"]
    CheckNewCycle -- No --> FetchCarried["Accumulate Unused GGR from\nPrior Stages in Current Cycle"]
    
    ExpirePrev --> AccumulateGGR["Compute Accumulated GGR\nAccumulatedGGR = CurrentGGR + PreviousCarriedGGR"]
    FetchCarried --> AccumulateGGR
    
    AccumulateGGR --> TierMatch["5. Tier Evaluation\nSort Tiers by MinimumWinAmount DESC"]
    TierMatch --> CheckHurdle{AccumulatedGGR >=\nMinimumWinAmount?}
    
    CheckHurdle -- No --> NoTier["Commission = $0.00\nSave GGR Carry Forward Record\n(Status = Active)"]
    CheckHurdle -- Yes --> MatchTier["Match Highest Qualifying Tier\nRetrieve CommissionRate & MaxCap"]
    
    MatchTier --> MarkUsed["Mark Prior Cycle Records as 'Used'"]
    MarkUsed --> CalcRoyalty["6. Compute Royalty Fee\nRoyaltyBase * RoyaltyRateWinLoss%"]
    CalcRoyalty --> CalcNetLoss["7. Compute Adjusted Net Loss\nAccumulatedGGR - Tx Fees - Royalty Fee"]
    CalcNetLoss --> ApplyRate["8. Compute Gross Commission\nAdjustedNetLoss * CommissionRate%"]
    ApplyRate --> CheckCap{Gross Commission >\nMaxCommissionAmount?}
    CheckCap -- Yes --> CapCommission["Commission = MaxCommissionAmount"]
    CheckCap -- No --> FinalCommission["Commission = Gross Commission"]
    
    CapCommission --> DistributeDetail["9. Proportionally Distribute to\nAffiliateCommissionDetail per Member"]
    FinalCommission --> DistributeDetail
    NoTier --> DistributeDetail
    DistributeDetail --> SaveReport["10. Commit Report & Details\n(Status = Pending (1))"]
    SaveReport --> End(["Calculation Complete"])
```

---

## 2. Mathematical Formulations & Pipeline Steps

### Step 1: Bet Ingestion & Stream Separation
Bets are queried from `ReportProviderBetHistories` where:
- `TransactionStatus == BetTransactionStatus.Success (1)`
- `SettlementStatus == BetSettlementStatus.Settled (1)`
- `BetTime >= PeriodStartDate AND BetTime < PeriodEndDate`
- `Member.AffiliateId == TargetAffiliateId`

The engine segments wagers into deposit-backed bets and promotion-backed bets:

$$\text{DepositBet} = \sum (\text{DepositBet})$$

$$\text{DepositWin} = \sum (\text{DepositWin})$$

$$\text{BonusBet} = \sum (\text{BonusBet})$$

$$\text{BonusWin} = \sum (\text{BonusWin})$$

### Step 2: Company Gross Gaming Revenue (Company GGR)
Gross Gaming Revenue represents the net loss suffered by players (which constitutes operator profit):

$$\text{DepositGGR} = \text{DepositBet} - \text{DepositWin}$$

$$\text{BonusGGR} = \text{BonusBet} - \text{BonusWin}$$

> [!IMPORTANT]
> **Active Business Rule on Bonus GGR:**  
> In `AffiliateCommissionCalculationService.cs`, bonus GGR is segmented but **not** deducted from the company profit base:
> $$\text{CurrentPeriodGGR} = \text{DepositGGR}$$
> Bonus wagers do not diminish an affiliate's qualifying commission base.

---

### Step 3: Transaction & Royalty Fee Deductions
Operational overhead incurred by the operator is passed through and deducted before commission payout.

#### 1. Deposit Transaction Fee
Sourced from `TransDeposits` where `Status == TransactionStatus.Completed`:

$$\text{DepositFee} = \sum \text{DepositAmount} \times \left( \frac{\text{DepositTransactionFeeRate}}{100} \right)$$

#### 2. Withdrawal Transaction Fee
Sourced from `TransWithdraws` where `Status == WithdrawalStatus.Approved`:

$$\text{WithdrawFee} = \sum \text{WithdrawalAmount} \times \left( \frac{\text{WithdrawTransactionFeeRate}}{100} \right)$$

#### 3. Total Transaction Fee

$$\text{TotalTransactionFee} = \text{DepositFee} + \text{WithdrawFee}$$

#### 4. Game Provider Royalty Fee
The royalty fee compensates third-party game providers (e.g., Pragmatic Play, Evolution). The base for royalty is the total net gaming result:

$$\text{RoyaltyFeeBase} = \text{TotalBetAmount} - \text{TotalWinAmount}$$

$$\text{RoyaltyFeeAmount} = \text{RoyaltyFeeBase} \times \left( \frac{\text{RoyaltyRateWinLoss}}{100} \right)$$

---

### Step 4: The 90-Day Staged GGR Carry Forward Algorithm

To balance operator risk during months when players experience lucky winning streaks (negative GGR), the platform implements an anchored **90-Day Staged Carry Forward Engine**.

#### Mathematical Stage & Cycle Resolution:
- **Anchor Date:** `FirstActivatedAt` (recorded on `AffiliateGroup`).
- **Valid Period:** `ValidPeriod` (typically 90 days).
- **Stage Duration:** $\frac{\text{ValidPeriod}}{3} = 30\text{ days per stage}$.

Given a billing month $M$ with period end date $T_{\text{end}}$:

1. **Days Since Activation:**
   $$\Delta_{\text{days}} = T_{\text{end}} - \text{FirstActivatedAt}$$

2. **Cycle Index ($C$):**
   $$C = \lfloor \frac{\Delta_{\text{days}}}{\text{ValidPeriod}} \rfloor$$

3. **Cycle Anchor Start Date:**
   $$T_{\text{cycleStart}} = \text{FirstActivatedAt} + (C \times \text{ValidPeriod})$$

4. **Days Into Current Cycle:**
   $$\Delta_{\text{cycle}} = T_{\text{end}} - T_{\text{cycleStart}}$$

5. **Stage In Current Cycle ($S$):**
   $$S = \min\left(3, \max\left(1, \lfloor \frac{\Delta_{\text{cycle}}}{30} \rfloor + 1\right)\right)$$

6. **New Cycle Flag (`IsNewCycle`):**
   $$\text{IsNewCycle} = (S == 1)$$

#### State Transitions & Expiration Rules:
- **If $\text{IsNewCycle} == \text{True}$:**
  - All unfulfilled active carry forward records from cycle $C-1$ are updated to `Status = Expired (3)`.
  - $\text{PreviousCarriedGGR} = 0.00$.
  - The affiliate starts fresh in Stage 1 of the new cycle.
- **If $\text{IsNewCycle} == \text{False}$ (Stage 2 or Stage 3):**
  - The engine searches for the most recent period in cycle $C$ where the tier hurdle was met.
  - If a prior period met the hurdle, carry forward only accumulates from periods occurring *after* that successful period.
  - Otherwise, $\text{PreviousCarriedGGR} = \sum_{\text{prior stages in } C} \text{CurrentPeriodGGR}$.

$$\text{AccumulatedGGR} = \text{CurrentPeriodGGR} + \text{PreviousCarriedGGR}$$

---

### Step 5: Tier Matching Logic

Configured tiers in `AffiliateCommissionSetting` are sorted in descending order of `MinimumWinAmount`:

```sql
SELECT * FROM AffiliateCommissionSetting
WHERE AffiliateGroupId = @GroupId AND Active = 1
ORDER BY MinimumWinAmount DESC
```

The engine iterates through the sorted tiers and selects the first tier satisfying:

$$\text{AccumulatedGGR} \ge \text{MinimumWinAmount}$$

#### Outcomes:
1. **Tier Matched:**
   - Active tier setting is selected (yields `CommissionRate`, `RoyaltyRateWinLoss`, `MaximumCommissionAmount`).
   - If previous carry forward was included ($\text{PreviousCarriedGGR} > 0$), all previous records in cycle $C$ are updated to `Status = Used (2)`.
   - Engine proceeds to commission calculation.
2. **Tier NOT Matched ($\text{AccumulatedGGR} < \text{Lowest MinimumWinAmount}$):**
   - $\text{CommissionAmount} = \$0.00$.
   - A carry-forward ledger record is created in `AffiliateCommissionGGRCarryForward`:
     - `CurrentPeriodGGR = CurrentPeriodGGR`
     - `PreviousCarriedGGR = PreviousCarriedGGR`
     - `CarriedForwardGGR = AccumulatedGGR`
     - `Status = Active (1)`
     - `PeriodStage = S`
     - `IsResetPeriod = IsNewCycle`
   - Calculation terminates with \$0 commission.

---

### Step 6: Net Loss & Commission Amount Formula

When a tier matches, the payable commission is computed as follows:

$$\text{AdjustedNetLoss} = \text{AccumulatedGGR} - \text{TotalTransactionFee} - \text{RoyaltyFeeAmount}$$

$$\text{GrossCommission} = \text{AdjustedNetLoss} \times \left( \frac{\text{CommissionRate}}{100} \right)$$

#### Application of Ceiling (Maximum Commission Cap):
If `MaximumCommissionAmount` is configured on the matched tier and $\text{GrossCommission} > 0$:

$$\text{CommissionAmount} = \min(\text{GrossCommission}, \text{MaximumCommissionAmount})$$

If $\text{AdjustedNetLoss} \le 0$ (e.g., excessive payment processing fees exceeding GGR):

$$\text{CommissionAmount} = \$0.00$$

---

### Step 7: Downline Member Proportional Distribution

For full auditability, the total `CommissionAmount` is distributed to individual member line items in `AffiliateCommissionDetail`.

For each referred member $i$:

$$\text{TotalCurrentPeriodGGR} = \sum_{m=1}^{N} \text{CompanyGGR}_m$$

$$\text{Proportion}_i = \frac{\text{CompanyGGR}_i}{\text{TotalCurrentPeriodGGR}}$$

$$\text{MemberCommissionAmount}_i = \text{CommissionAmount} \times \text{Proportion}_i$$

$$\sum_{i=1}^{N} \text{MemberCommissionAmount}_i \equiv \text{CommissionAmount}$$

---

## 3. Concrete Numerical Worked Scenarios

To assist QA and developers in verifying algorithmic correctness, five comprehensive calculation scenarios are presented below.

### Group Configuration Parameters for Scenarios:
- **Group Name:** Gold VIP Affiliates
- **Valid Period:** 90 Days (3 stages of 30 days)
- **Deposit Fee Rate:** 2.00%
- **Withdraw Fee Rate:** 1.00%
- **Tier Configuration:**

| Tier Name | Minimum Win Amount (GGR) | Commission Rate | Royalty Rate | Maximum Commission Cap |
| :--- | :---: | :---: | :---: | :---: |
| **Tier 1 (Bronze)** | \$10,000.00 | 20.00% | 10.00% | \$5,000.00 |
| **Tier 2 (Silver)** | \$30,000.00 | 30.00% | 10.00% | \$15,000.00 |
| **Tier 3 (Gold)** | \$60,000.00 | 40.00% | 8.00% | None (Uncapped) |

---

### Scenario A: Standard Profitable Month (Tier 2 Reached)
- **Period:** 2026-06 (Cycle 1, Stage 1)
- **Previous Carried GGR:** \$0.00
- **Member Activity:**
  - Total Deposit Bets: \$200,000.00
  - Total Deposit Wins: \$160,000.00
  - Total Member Deposits: \$80,000.00
  - Total Member Withdrawals: \$40,000.00

#### Mathematical Step-by-Step:
1. $\text{CurrentPeriodGGR} = 200,000 - 160,000 = \$40,000.00$
2. $\text{AccumulatedGGR} = 40,000 + 0 = \$40,000.00$
3. **Tier Matching:**
   - Tier 3 (\$60,000): Not met ($40,000 < 60,000$).
   - Tier 2 (\$30,000): **Met** ($40,000 \ge 30,000$).
   - Selected: **Tier 2** (Rate: 30%, Royalty: 10%, Cap: \$15,000).
4. **Fees Calculation:**
   - $\text{DepositFee} = 80,000 \times 2.00\% = \$1,600.00$
   - $\text{WithdrawFee} = 40,000 \times 1.00\% = \$400.00$
   - $\text{TotalTransactionFee} = 1,600 + 400 = \$2,000.00$
   - $\text{RoyaltyFeeBase} = \$40,000.00$
   - $\text{RoyaltyFeeAmount} = 40,000 \times 10.00\% = \$4,000.00$
5. **Adjusted Net Loss:**
   - $\text{AdjustedNetLoss} = 40,000 - 2,000 - 4,000 = \$34,000.00$
6. **Commission Amount:**
   - $\text{GrossCommission} = 34,000 \times 30.00\% = \$10,200.00$
   - Cap Check: $\min(10,200, 15,000) = \mathbf{\$10,200.00}$
7. **Database State:**
   - `AffiliateCommissionReport`: `CommissionAmount = 10200.00`, `Status = 1 (Pending)`.
   - `AffiliateCommissionGGRCarryForward`: `Status = Used (2)`.

---

### Scenario B: Negative Month (Players Won - Rollover Stage 1 -> Stage 2)
- **Period:** 2026-07 (Cycle 1, Stage 1)
- **Previous Carried GGR:** \$0.00
- **Member Activity:**
  - Total Deposit Bets: \$100,000.00
  - Total Deposit Wins: \$120,000.00 (Players won \$20,000 net)
  - Total Deposits: \$30,000.00
  - Total Withdrawals: \$45,000.00

#### Mathematical Step-by-Step:
1. $\text{CurrentPeriodGGR} = 100,000 - 120,000 = -\$20,000.00$
2. $\text{AccumulatedGGR} = -20,000 + 0 = -\$20,000.00$
3. **Tier Matching:**
   - Lowest Tier requirement is \$10,000.00.
   - $-\$20,000.00 < \$10,000.00 \implies$ **No Tier Matched**.
4. **Commission Result:**
   - $\text{CommissionAmount} = \mathbf{\$0.00}$.
5. **Carry Forward Ledger State (Created):**
   - `CalculationPeriod`: `2026-07`
   - `CurrentPeriodGGR`: `-$20,000.00`
   - `PreviousCarriedGGR`: `$0.00`
   - `CarriedForwardGGR`: `-$20,000.00`
   - `PeriodStage`: `1`
   - `Status`: `Active (1)` (Rolled into Stage 2).

---

### Scenario C: Recovery Month in Stage 2 (Accumulated GGR Reaches Hurdle)
- **Period:** 2026-08 (Cycle 1, Stage 2)
- **Previous Carried GGR:** $-\$20,000.00$ (From Period 2026-07)
- **Member Activity:**
  - Total Deposit Bets: \$350,000.00
  - Total Deposit Wins: \$295,000.00
  - Total Member Deposits: \$90,000.00
  - Total Member Withdrawals: \$30,000.00

#### Mathematical Step-by-Step:
1. $\text{CurrentPeriodGGR} = 350,000 - 295,000 = \$55,000.00$
2. $\text{AccumulatedGGR} = 55,000 + (-20,000) = \mathbf{\$35,000.00}$
3. **Tier Matching:**
   - Tier 3 (\$60,000): Not met ($35,000 < 60,000$).
   - Tier 2 (\$30,000): **Met** ($35,000 \ge 30,000$).
   - Selected: **Tier 2** (Rate: 30%, Royalty: 10%, Cap: \$15,000).
4. **Fees Calculation:**
   - $\text{DepositFee} = 90,000 \times 2.00\% = \$1,800.00$
   - $\text{WithdrawFee} = 30,000 \times 1.00\% = \$300.00$
   - $\text{TotalTransactionFee} = 1,800 + 300 = \$2,100.00$
   - $\text{RoyaltyFeeBase} = \$55,000.00$
   - $\text{RoyaltyFeeAmount} = 55,000 \times 10.00\% = \$5,500.00$
5. **Adjusted Net Loss:**
   - $\text{AdjustedNetLoss} = 35,000 - 2,100 - 5,500 = \$27,400.00$
6. **Commission Amount:**
   - $\text{GrossCommission} = 27,400 \times 30.00\% = \$8,220.00$
   - Cap Check: $\min(8,220, 15,000) = \mathbf{\$8,220.00}$
7. **Carry Forward Ledger State (Updated):**
   - 2026-07 record updated to: `Status = Used (2)`.
   - 2026-08 record created with: `CarriedForwardGGR = 0.00`, `Status = Used (2)`.

---

### Scenario D: 90-Day Cycle Expiration (Stage 3 Fails Hurdle, Reset to 0)
- **Timeline:**
  - Stage 1 (2026-01): $\text{GGR} = -\$15,000.00 \implies$ Carried to Stage 2 ($-\$15,000$).
  - Stage 2 (2026-02): $\text{GGR} = +\$8,000.00 \implies \text{Accumulated} = -\$7,000 \implies$ Carried to Stage 3 ($-\$7,000$).
  - Stage 3 (2026-03): $\text{GGR} = +\$12,000.00 \implies \text{Accumulated} = +\$5,000.00$.
- **Tier Check Stage 3:**
  - Lowest tier minimum is \$10,000.00.
  - $\$5,000.00 < \$10,000.00 \implies$ No tier matched.
  - Stage 3 marks the end of Cycle 1 (90 days elapsed).
- **Execution in Stage 1 of Next Cycle (2026-04, Cycle 2, Stage 1):**
  - `CalculateStageInfoByPeriodAsync` detects $\text{CycleNumber} = 2$, $\text{CurrentStage} = 1$, $\text{IsNewCycle} = \text{True}$.
  - `ClearPreviousCycleCarryForward` executes:
    - All records for Cycle 1 updated to `Status = Expired (3)`.
  - For period 2026-04:
    - $\text{PreviousCarriedGGR} = \mathbf{\$0.00}$ (Reset).
    - Player wagers in 2026-04 are evaluated purely on current period performance without the $-\$7,000$ or $+\$5,000$ residue.

---

### Scenario E: High Profit Month Capped by MaximumCommissionAmount
- **Period:** 2026-09 (Cycle 2, Stage 1)
- **Previous Carried GGR:** \$0.00
- **Member Activity:**
  - Total Deposit Bets: \$500,000.00
  - Total Deposit Wins: \$380,000.00
  - Total Member Deposits: \$150,000.00
  - Total Member Withdrawals: \$50,000.00
- **Matched Tier:** Tier 2 (Minimum Win Amount: \$30,000.00, Commission Rate: 30%, Max Cap: \$15,000.00)

#### Mathematical Step-by-Step:
1. $\text{CurrentPeriodGGR} = 500,000 - 380,000 = \$120,000.00$
2. $\text{AccumulatedGGR} = \$120,000.00$
3. **Fees Calculation:**
   - $\text{DepositFee} = 150,000 \times 2.00\% = \$3,000.00$
   - $\text{WithdrawFee} = 50,000 \times 1.00\% = \$500.00$
   - $\text{TotalTransactionFee} = \$3,500.00$
   - $\text{RoyaltyFeeAmount} = 120,000 \times 10.00\% = \$12,000.00$
4. **Adjusted Net Loss:**
   - $\text{AdjustedNetLoss} = 120,000 - 3,500 - 12,000 = \$104,500.00$
5. **Gross Commission:**
   - $\text{GrossCommission} = 104,500 \times 30.00\% = \$31,350.00$
6. **Cap Enforcement:**
   - Matched tier specifies $\text{MaximumCommissionAmount} = \$15,000.00$.
   - $\text{CalculatedCommission} = \min(31,350, 15,000) = \mathbf{\$15,000.00}$
7. **Member Line-Item Proportion:**
   - If Member A contributed \$60,000 of the \$120,000 GGR (50%):
   - $\text{MemberCommissionAmount}_A = 15,000 \times \left(\frac{60,000}{120,000}\right) = \mathbf{\$7,500.00}$.

---

## 4. Edge Cases & Exception Handling Matrix

| Scenario / Edge Case | System Behavior | Data Impact & Audit Record |
| :--- | :--- | :--- |
| **No Bets Found in Billing Period** | Calculation gracefully exits. Returns summary with 0s across bets, wins, and commission. | No report created in DB. Informational log recorded. |
| **Negative Net Loss (Players Won)** | Tier matching fails. Commission evaluates to \$0.00. Entire negative GGR added to carry-forward ledger. | `AffiliateCommissionGGRCarryForward` created with `CarriedForwardGGR < 0`, `Status = Active (1)`. |
| **Fees Exceed Company GGR** | If $\text{AdjustedNetLoss} < 0$ after deducting transaction/royalty fees from positive GGR, commission evaluates to \$0.00. | Report created with `CommissionAmount = 0.00`. Notes state adjusted net loss was negative. |
| **Concurrent Calculation Execution** | Serializable transaction locks affiliate row. If report already exists for period and `forceRecalculate == false`, returns existing report. | Prevents duplicate billing statements or multiple ledger credits. |
| **Force Recalculation Request** | When `forceRecalculate == true`, existing `AffiliateCommissionDetail` records and `AffiliateCommissionReport` are purged and recomputed. | Database transaction wraps deletion and re-insertion. |
| **Anchor Date `FirstActivatedAt` Missing** | If `FirstActivatedAt == null` on `AffiliateGroup`, system automatically initializes it to `DateTime.UtcNow`. | Persisted to DB immediately before stage computation. |
| **Period Before Activation Date** | Period date preceding `FirstActivatedAt` throws `InvalidOperationException`. | Calculation halted with explicit error log. |
| **Active Member Count Below Group Threshold** | If active players with valid turnover are fewer than `TotalCommissionActiveMembers`, commission evaluates to \$0.00. | GGR carried forward or held pending activity requirement. |
