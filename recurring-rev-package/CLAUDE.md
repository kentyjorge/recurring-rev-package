# recurring-rev-package

Revenue Cloud ARR calculation package for the `main` org (`main_demo_ks@salesforce.com`). All calculation logic lives in Flow — no Apex. Triggered manually via a **Run ARR Calcs** Quick Action on the Quote record.

---

## Project Structure

```
force-app/main/default/
├── objects/Quote/fields/       # 9 custom currency fields (RRV_ prefix)
├── flows/
│   ├── RRV_Calculate_Quote_ARR.flow-meta.xml   # Autolaunched subflow (core logic)
│   └── RRV_Run_ARR_Calcs.flow-meta.xml         # Screen flow (quick action entry point)
├── quickActions/
│   └── Quote.RRV_Run_ARR_Calcs.quickAction-meta.xml
└── profiles/
    └── Admin.profile-meta.xml                  # Read/edit on all 9 RRV fields
```

---

## Fields (Quote object, `RRV_` prefix)

| API Name | Label | Description |
|----------|-------|-------------|
| `RRV_Total_ARR__c` | Total ARR | Sum of annualized ARR across all lines |
| `RRV_New_ARR__c` | New ARR | Lines where StartQuantity = 0 or null |
| `RRV_Renewed_ARR__c` | Renewed ARR | Lines where Quantity = StartQuantity > 0 |
| `RRV_Upsell_Amount__c` | Upsell Amount | (Qty − StartQty) × unit price × multiplier |
| `RRV_Churn_Amount__c` | Churn Amount | (StartQty − Qty) × unit price × multiplier |
| `RRV_Total_Contract_Value__c` | Total Contract Value | SUM(NetTotalPrice) — no annualization |
| `RRV_Year1_ARR__c` | Year 1 ARR | Lines overlapping months 1–12 from quote start |
| `RRV_Year2_ARR__c` | Year 2 ARR | Lines overlapping months 13–24 from quote start |
| `RRV_Year3_ARR__c` | Year 3 ARR | Lines overlapping months 25–36 from quote start |

---

## Calculation Logic

### Source fields on QuoteLineItem

| Field | Notes |
|-------|-------|
| `NetUnitPrice` | Price per unit per billing period |
| `NetTotalPrice` | Full line total — used for TCV |
| `BillingFrequency` | Drives annualization multiplier |
| `Quantity` | Current quantity |
| `StartQuantity` | Prior quantity — **`StartQuantity` is the correct API name in this org** (not `StartingQuantity`) |
| `StartDate` | Line start date |
| `EndDate` | Line end date |

### Annualization multipliers

| BillingFrequency | Multiplier |
|-----------------|------------|
| `Monthly` | × 12 |
| `Quarterly` | × 4 |
| `SemiAnnual` | × 2 |
| `Annual` | × 1 |

`Line ARR = NetUnitPrice × Quantity × multiplier`

### Flow architecture

**Loop 1** — ARR / TCV accumulation:
- Decision on `BillingFrequency` → set `varMultiplier`
- Track `varQuoteStart` = MIN(StartDate) across lines
- Classify line (New / Renewed / Upsell / Churn) → accumulate into appropriate variable
- Accumulate `varTCV += NetTotalPrice`

**Loop 2** — Multi-year window (runs after Loop 1 so `varQuoteStart` is final):
- Year boundaries: `ADDMONTHS(varQuoteStart, 12/24/36)`
- Overlap check: `StartDate < yearEnd AND EndDate > yearStart`
- A line can contribute to multiple years

**Update Record** — writes all 9 fields to the Quote.

---

## Deployment

```bash
sf project deploy start \
  --source-dir force-app \
  --target-org main \
  --wait 10
```

**Post-deploy manual step:** Add `Run ARR Calcs` to the Quote Lightning record page via Setup → Lightning App Builder (highlights panel or record page actions).

---

## Key Technical Decisions

- **No Apex** — all logic in autolaunched Flow to keep it declarative and maintainable.
- **Two-loop pattern** — Loop 2 for year windows runs after Loop 1 so `varQuoteStart` is correctly set to the earliest line start date before year boundary formulas are evaluated.
- **`StartQuantity` field name** — confirmed via `sf sobject describe` on this org; the plan originally referenced `StartingQuantity` which does not exist.
- **Flow XML element grouping** — Salesforce Flow metadata requires all elements of the same type to be grouped together in the XML. Execution order is defined by connectors, not XML position.

---

## Session History

### Session 1 — 2026-04-30
- Designed and built all metadata from scratch
- Deployed 12 components: 9 fields, 2 flows, 1 quick action
- Fixed two deploy errors: Flow XML element grouping, `StartQuantity` field name
- Added `Admin.profile-meta.xml` for field visibility
- Initialized git repo and committed all files
