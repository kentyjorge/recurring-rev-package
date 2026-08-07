# Recurring Revenue Package (RRV)

A declarative Salesforce Revenue Cloud package that calculates ARR (Annual Recurring Revenue), TCV (Total Contract Value), and multi-year revenue breakdowns on the Quote object. All logic is implemented in Flow — no Apex required.

> This is **step 5** of the [RC-SE-Pack](https://github.com/kentyjorge/RC-SE-Pack)
> orchestrator, but it deploys fine standalone.
>
> **Target org:** an existing **sandbox / Developer Edition / dev** org with
> Revenue Cloud (RLM) enabled. **Never deploy to production** from this flow.

> **Note:** this repo also contains `CLAUDE.md` and `SKILLS.md` — those are
> AI-agent reference docs describing the calculation logic, **not** deploy
> instructions. This README is the deploy guide.

## Overview

This package adds a **Run ARR Calcs** quick action to the Quote record page. When triggered, it loops over all QuoteLineItems attached to the Quote and populates 10 custom currency fields with ARR classification, year-bucket breakdowns, non-recurring revenue, and services subtotals.

### What It Calculates

| Metric | Description |
|--------|-------------|
| **Total ARR** | Weighted-average ARR across non-zero year buckets |
| **New ARR** | Revenue from net-new lines (StartQuantity = 0 or null) |
| **Renewed ARR** | Revenue from renewed lines (Quantity = StartQuantity > 0) |
| **Upsell Amount** | Incremental revenue from expanded lines (Qty > StartQty) |
| **Churn Amount** | Lost revenue from contracted lines (Qty < StartQty) |
| **Total Contract Value** | Sum of NetTotalPrice across all recurring lines |
| **Year 1 ARR** | ARR from lines overlapping months 1–12 |
| **Year 2 ARR** | ARR from lines overlapping months 13–24 |
| **Year 3 ARR** | ARR from lines overlapping months 25–36 |
| **Non-Recurring Revenue** | Sum of one-time lines (null BillingFrequency) |
| **Services Subtotal** | Value of lines with "Services" in product family |

## Architecture

```
Quote Record Page
  └── Quick Action: RRV_Run_ARR_Calcs
        └── Screen Flow: RRV_Run_ARR_Calcs
              └── Subflow (Autolaunched): RRV_Calculate_Quote_ARR
                    ├── Loop 0: Find MIN(StartDate) → varQuoteStart
                    ├── Loop 1: Classify lines + accumulate TCV (Year 1 gate)
                    ├── Loop 2: Bucket lines into Year 1/2/3 windows
                    ├── Formula: Weighted-average ARR (frmAvgARR)
                    ├── Query: One-time lines → accumulate Non-Recurring Rev
                    ├── Query: Service lines → accumulate Services Subtotal
                    └── Update Quote with all calculated fields
```

### Key Design Decisions

- **Pure Flow, no Apex** — keeps the package declarative, admin-maintainable, and avoids governor limit complexity for typical quote sizes.
- **Three-loop pattern** — Loop 0 establishes `varQuoteStart` (earliest line start date) before any classification runs. Loop 1 handles TCV + ARR classification for Year 1 lines. Loop 2 handles multi-year bucketing independently.
- **Weighted-average Total ARR** — divides the sum of year buckets by the count of non-zero years, correctly averaging 1-, 2-, or 3-year deals rather than naively summing.
- **Separate one-time and service queries** — one-time lines (null BillingFrequency) and service lines (by product family) are queried and accumulated independently from the main recurring line loops.

## Metadata Components

### Custom Fields (Quote object, `RRV_` prefix)

| API Name | Type |
|----------|------|
| `RRV_Total_ARR__c` | Currency |
| `RRV_New_ARR__c` | Currency |
| `RRV_Renewed_ARR__c` | Currency |
| `RRV_Upsell_Amount__c` | Currency |
| `RRV_Churn_Amount__c` | Currency |
| `RRV_Total_Contract_Value__c` | Currency |
| `RRV_Year1_ARR__c` | Currency |
| `RRV_Year2_ARR__c` | Currency |
| `RRV_Year3_ARR__c` | Currency |
| `RRV_NonRecurring_Rev__c` | Currency |

### Flows

| Flow | Type | Purpose |
|------|------|---------|
| `RRV_Calculate_Quote_ARR` | Autolaunched | Core calculation logic (subflow) |
| `RRV_Run_ARR_Calcs` | Screen Flow | Quick action entry point; calls subflow and shows confirmation |

### Quick Action

| Component | Target Object |
|-----------|---------------|
| `Quote.RRV_Run_ARR_Calcs` | Quote |

### Profile

| Component | Purpose |
|-----------|---------|
| `Admin.profile-meta.xml` | Read/Edit access on all RRV fields |

## Calculation Logic

### Annualization Multipliers

| BillingFrequency | Multiplier |
|-----------------|------------|
| Monthly | 12 |
| Quarterly | 4 |
| SemiAnnual | 2 |
| Annual | 1 (default) |

**Line ARR** = `NetUnitPrice × Quantity × Multiplier`

### Classification Rules

| Category | Condition |
|----------|-----------|
| New | StartQuantity is null OR = 0 |
| Renewed | Quantity = StartQuantity AND StartQuantity > 0 |
| Upsell | Quantity > StartQuantity AND StartQuantity > 0 |
| Churn | Quantity < StartQuantity (default/else) |

### Year Window Logic

Year boundaries are calculated from the earliest line start date:
- **Year 1**: `varQuoteStart` to `ADDMONTHS(varQuoteStart, 12)`
- **Year 2**: Month 12 to `ADDMONTHS(varQuoteStart, 24)`
- **Year 3**: Month 24 to `ADDMONTHS(varQuoteStart, 36)`

A line contributes to a year bucket if: `StartDate < YearEnd AND EndDate > YearStart`

### Total ARR Formula

```
Total ARR = (Y1 + Y2 + Y3) ÷ count of non-zero year buckets
```

This handles variable-length deals correctly — a 1-year deal reports Year 1 ARR as Total ARR, while a 3-year deal reports the average.

## Prerequisites

- Salesforce org with Revenue Cloud (Quote/QuoteLineItem with `BillingFrequency`, `StartQuantity`, `NetUnitPrice` fields)
- `RLM_Product_Family__c` field on QuoteLineItem (used for Services line identification)
- API version 59.0+

## Deployment

> **Layout note:** the SFDX project is inside the `recurring-rev-package/`
> subfolder, not at the repo root. `cd` into it before deploying.

```bash
git clone https://github.com/kentyjorge/recurring-rev-package.git
cd recurring-rev-package/recurring-rev-package     # nested project folder

sf project deploy start --source-dir force-app --target-org <alias> --dry-run
sf project deploy start --source-dir force-app --target-org <alias> --wait 10
```

> **Gotcha — the repo ships an `Admin` profile.** Deploying `force-app` also
> deploys `Admin.profile-meta.xml`, which applies field-level access for the
> `RRV_` fields to the **standard Admin profile**. If you'd rather not touch the
> Admin profile, deploy everything **except** the profile and grant FLS your own
> way:
> ```bash
> sf project deploy start \
>   --metadata "CustomField,Flow,QuickAction" \
>   --target-org <alias>
> ```

### Post-Deploy Setup

1. **Add Quick Action to Page Layout**: Navigate to Setup → Object Manager → Quote → Lightning Record Pages → Edit the active page → Add the "Run ARR Calcs" action to the Highlights Panel or Actions section.

2. **Verify Field Visibility**: Ensure the target profile/permission set has Read/Edit access to all `RRV_*` fields on Quote.

3. **Confirm the flows are active** (both deploy `Active`) — Setup → Flows.

4. **Grant field access to non-admins** if you skipped the Admin profile (surface
   the `RRV_` fields on the page layout / via a permission set).

### Verify

```bash
sf project deploy report --target-org <alias>
```
Open a Quote with line items → click **Run ARR Calcs** → confirm `RRV_Total_ARR__c`
and the year buckets populate.

## Known gotchas & limitations

- **Manual trigger only** — ARR is recomputed when a user clicks **Run ARR Calcs**,
  not on save.
- **`StartQuantity` is the correct API name** for prior quantity in this data
  model (not `StartingQuantity`).
- Year buckets assume a 36-month (Y1–Y3) horizon from the quote start date.

## Safety

- Metadata-only deploy — no data is loaded or deleted.
- Review the `--dry-run` output; be aware of the **Admin profile** deploy note
  above before deploying the whole `force-app`.

## Project Structure

```
recurring-rev-package-project/
├── recurring-rev-package/           # Deployable SFDX package
│   ├── force-app/main/default/
│   │   ├── flows/
│   │   │   ├── RRV_Calculate_Quote_ARR.flow-meta.xml
│   │   │   └── RRV_Run_ARR_Calcs.flow-meta.xml
│   │   ├── objects/Quote/fields/    # 10 custom currency fields
│   │   ├── quickActions/
│   │   │   └── Quote.RRV_Run_ARR_Calcs.quickAction-meta.xml
│   │   └── profiles/
│   │       └── Admin.profile-meta.xml
│   ├── sfdx-project.json
│   └── CLAUDE.md                    # Detailed technical reference
├── force-app/                       # Empty SFDX shell (outer project)
├── config/
│   └── project-scratch-def.json
├── scripts/                         # Utility scripts
├── package.json                     # Prettier, ESLint, Husky config
├── sfdx-project.json
└── README.md                        # This file
```

## Development

### Code Quality

The project uses pre-commit hooks via Husky + lint-staged:
- **Prettier** formats all metadata files (XML, JSON, etc.)
- **ESLint** lints any LWC/Aura JavaScript
- **sfdx-lwc-jest** runs related unit tests

```bash
# Install dependencies (for pre-commit hooks)
npm install

# Format all files
npm run prettier

# Verify formatting
npm run prettier:verify
```

## Session History

### Session 1 — 2026-04-30
- Designed and built all metadata from scratch
- Deployed 12 components: 9 fields, 2 flows, 1 quick action
- Fixed deploy errors: Flow XML element grouping, `StartQuantity` field name
- Added `Admin.profile-meta.xml` for field visibility
- Initialized git repo and committed all files

### Session 2 — 2026-07-28
- Restructured Total ARR from Year-1-only accumulation to weighted average of year buckets
- Added `frmAvgARR` formula and `assignAvgARR` element
- Added `RRV_NonRecurring_Rev__c` field and one-time line accumulation
- Added Services Subtotal calculation (query by `RLM_Product_Family__c`)
- Deployed and verified on `maintwo` org

### Session 3 — 2026-07-29
- Created comprehensive README for GitHub publication
- Pushed to GitHub repository
