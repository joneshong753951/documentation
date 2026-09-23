# ETSWalletV2 - Affiliate Management & Commission Engine
## Technical & Business Documentation Suite for QA & Development

---

### Executive Summary

The **Affiliate Management System** in **ETSWalletV2** is a multi-tier affiliate marketing, attribution, reporting, and commission calculation platform. It enables:
1. **Affiliate Portal**: A dedicated web portal (`Affiliate` project) allowing affiliates to register, manage profile details, track referral URLs, view live player statistics, inspect member wagers and net wins/losses (GGR), and audit commission reports.
2. **Member Tracking & Attribution**: Real-time referral tracking (`Member/Middleware/AffiliateTrackingMiddleware.cs`) using 30-day first-touch attribution cookies (`SiteAff`) and digital marketing ad click identifiers (`vcid`, `bcid`, `rtid`, `clickid`).
3. **Multi-Tier Commission Engine**: An automated financial calculation pipeline (`AffiliateCommissionCalculationService.cs`) that processes settled wagers from gaming providers, calculates separated Deposit GGR and Bonus GGR, applies tiered thresholds, deducts transaction and platform royalty fees, and enforces maximum commission ceilings.
4. **90-Day Staged GGR Carry-Forward Subsystem**: An automated 3-stage rollover mechanism (`AffiliateCommissionCarryForwardService.cs`) that preserves affiliate earnings when monthly minimum win amounts are not reached, rolling them over within a 90-day cycle before expiring.
5. **BackOffice Administration & Finance Workflows**: Administrative interfaces for group setup, settlement parameters, tier definitions, manual commission execution, data integrity validation, and one-click financial approvals/rejections with automated balance crediting and debiting.

---

### Documentation Suite Navigation

| Document | Purpose | Target Audience |
| :--- | :--- | :--- |
| [**01_BUSINESS_REQUIREMENTS_SPECIFICATION.md**](./01_BUSINESS_REQUIREMENTS_SPECIFICATION.md) | Business concepts, affiliate lifecycle, attribution models, affiliate groups, financial workflows, and business rules. | Product Managers, Business Analysts, QA Leads, Dev Leads |
| [**02_TECHNICAL_ARCHITECTURE_AND_DATA_MODELS.md**](./02_TECHNICAL_ARCHITECTURE_AND_DATA_MODELS.md) | Architecture diagrams, component breakdown, database ERD, data dictionary, API endpoints, security, and authorization. | Developers, Architects, Database Administrators, Security Testers |
| [**03_COMMISSION_CALCULATION_AND_LOGIC_ENGINE.md**](./03_COMMISSION_CALCULATION_AND_LOGIC_ENGINE.md) | Comprehensive mathematical formulas, GGR/NGR definitions, separated wager streams, tier selection algorithms, 90-day staged rollover state machine, and numerical examples. | Core Developers, Financial QA, Algorithmic Auditors |
| [**04_AFFILIATE_PORTAL_FUNCTIONAL_SPECIFICATION.md**](./04_AFFILIATE_PORTAL_FUNCTIONAL_SPECIFICATION.md) | Page-by-page functional and UI specification of the Affiliate Portal (`Affiliate`), views, modals, DataTables, client-side scripts, and localization. | Frontend Developers, QA Engineers, UX/UI Designers |
| [**05_BACKOFFICE_ADMIN_AND_WORKFLOW_GUIDE.md**](./05_BACKOFFICE_ADMIN_AND_WORKFLOW_GUIDE.md) | BackOffice configuration, group and tier settings, manual calculations, approval/rejection workflows, balance management, and audit logging. | BackOffice Developers, Operational QA, Operations Staff |
| [**06_DEVELOPER_INTEGRATION_AND_CODE_REFERENCE.md**](./06_DEVELOPER_INTEGRATION_AND_CODE_REFERENCE.md) | Codebase walkthrough, class hierarchies, dependency injection, cron job handlers, background services, and extension guidelines. | Backend Developers, Systems Engineers |
| [**07_QA_TESTING_AND_VERIFICATION_MATRIX.md**](./07_QA_TESTING_AND_VERIFICATION_MATRIX.md) | Detailed QA test strategy, calculation test vectors with exact numerical inputs/expected outputs, edge cases, and compliance checklists. | QA Engineers, Automation Testers, Release Managers |

---

### Solution Architecture Overview

```
                      +---------------------------------------+
                      |           Online Player               |
                      |   (Visits with ?aff=AFF001123456)     |
                      +-------------------+-------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| ETSWalletV2.Member (Web Application)                                              |
| - AffiliateTrackingMiddleware: Captures ?aff=, stores 30-day "SiteAff" cookie      |
| - AccountController: Registers Member with AffiliateId foreign key                |
| - TransDeposits / TransWithdraws: Records financial transactions                  |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| ETSWalletV2.WalletData & SQL Database (Shared Persistence Layer)                   |
| - Affiliates & AffiliateGroups (Config, Limits, Settlement Rules)                  |
| - AffiliateTiers & AffiliateCommissionSettings (Tiers, Rates, Royalty Fees)        |
| - ReportProviderBetHistories (Raw Provider Bets/Wins: DepositBet, BonusBet, etc.) |
| - AffiliateCommissionReports & AffiliateCommissionDetails (Calculated Summaries)   |
| - AffiliateCommissionGGRCarryForwards (90-Day Staged Rollover State)              |
| - AffiliateTransactions (Ledger of Balance Credits and Debits)                    |
+-----------------------------------------------------------------------------------+
         ^                                                          ^
         |                                                          |
+--------+----------------------------+    +------------------------+---------------+
| ETSWalletV2.WalletCore (Cron Engine)|    | ETSWalletV2.BackOffice (Admin Portal)  |
| - CalculateAffiliateCommissionJob   |    | - AffiliateController (CRUD, Reset Pwd)|
| - AffiliateCommissionSummaryService |    | - AffiliateSettingController (CMS/Tiers|
| - Runs automated settlement cron    |    | - AffiliateCommissionReportController  |
|   evaluating completed periods      |    |   (Review, Approve, Reject, Ledger)    |
|                                     |    | - AffiliateCommissionValidationService |
|                                     |    |   (Integrity & Discrepancy Audits)     |
+-------------------------------------+    +----------------------------------------+
         ^
         |
+--------+--------------------------------------------------------------------------+
| ETSWalletV2.Affiliate (Web Portal - MVC 8.0)                                      |
| - AccountController: Login, Overview, MyAccount, Reports, Balance, Profile        |
| - HomeController: Index, Commission Plans, FAQ, Terms, Contact                    |
| - API/StatusController: Commission Status Badges & Display Names                  |
| - API/LocalizationController: Multi-Language & Translation Strings                |
+-----------------------------------------------------------------------------------+
```

---

### Quick Reference: Key Database Entities

| Entity / Table Name | Primary Role | Key Columns |
| :--- | :--- | :--- |
| `Affiliate` | Master affiliate record linked to `ApplicationUser` | `AffiliateId`, `AffiliateCode`, `AffiliateGroupId`, `Balance`, `Status` |
| `AffiliateGroup` | Configuration profile governing settlement and fees | `AffiliateGroupId`, `CommissionSettlementPeriod`, `ValidPeriod`, `FirstActivatedAt`, fee rates |
| `AffiliateTier` | Tier levels assigned to an Affiliate Group | `AffiliateTierId`, `AffiliateGroupId`, `TierName`, `DisplayOrder` |
| `AffiliateCommissionSetting` | Mathematical rules & rates per tier | `AffiliateTierId`, `MinimumWinAmount`, `CommissionRate`, `RoyaltyRateWinLoss`, `MaximumCommissionAmount` |
| `AffiliateCommissionReport` | Period commission summary per affiliate | `AffiliateCommissionReportId`, `CalculationPeriod`, `NetWinLoss`, `CommissionAmount`, `Status` |
| `AffiliateCommissionDetail` | Member-level commission contribution breakdown | `AffiliateCommissionDetailId`, `MemberId`, `BetAmount`, `WinAmount`, `CommissionAmount` |
| `AffiliateCommissionGGRCarryForward` | 90-day staged rollover tracking | `CarriedForwardGGR`, `PreviousCarriedGGR`, `PeriodStage`, `Status` (Active, Used, Expired) |
| `AffiliateTransactions` | Financial audit ledger for affiliate balance | `ReferenceNumber`, `Amount`, `CreditType` (Increase/Deduct), `TransactionType` (CA/CR), `Balance` |

---

### Contact & Maintenance
- **Repository Location**: `C:\Users\lovec\source\repos\ETSWalletV2\Affiliate`
- **Documentation Package Path**: `C:\Users\lovec\OneDrive\Desktop\Affiliate_System_Documentation`
- **Target Runtime**: .NET 8.0, ASP.NET Core, Entity Framework Core 8, Microsoft SQL Server.
