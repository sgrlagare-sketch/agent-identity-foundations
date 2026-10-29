---
name: analyze-tax-implications
description: Investigates property tax holding costs and assessment records for a parcel.
metadata:
  adk_additional_tools:
    - get_tax_assessment_data
    - get_lot_data
---
# Analyze Tax Implications Skill

When investigating acquisition valuation basis, financial carrying costs, or tax implications for a lot, you MUST perform and prominently highlight the following exact mathematical calculations:

1. **Retrieve Official Tax Assessment Records**:
   - Locate the `parcel_apn` from the subject lot's property records.
   - Call `get_tax_assessment_data` with that `parcel_apn` to retrieve the official `annual_tax_amount`, `assessed_land_value`, `assessed_improvement_value`, `total_assessed_value`, `tax_year`, and `payment_status`.

2. **Derive, State, and Highlight Exact Acquisition Cost-Per-Acre (Mandatory Core Valuation Metric)**:
   - Using the subject parcel's `last_sale_price` and `acreage` from `get_lot_data`, explicitly calculate the **Acquisition Cost-Per-Acre** using the formula:
     `Acquisition Cost-Per-Acre = Last Sale Price / Acreage`
   - **Show the explicit mathematical derivation formula and exact result** (for example: `$450,000 / 1.2 acres = $375,000.00/acre`, or `$680,000 / 4.4 acres = $154,545.45/acre`).
   - **Highlight Acquisition Cost-Per-Acre as a core valuation metric** in both the **Executive Summary** and the **Acquisition Basis and Holding Costs** section (in both a financial summary table and narrative callout).
   - **Benchmark the Acquisition Cost-Per-Acre**:
     - Compare the Acquisition Cost-Per-Acre (`last_sale_price / acreage`) against the **Assessed Land Value Per Acre** (`assessed_land_value / acreage`) to evaluate whether the acquisition price is at a discount or premium to the county assessor's valuation and identify post-acquisition tax reassessment risk.
     - Compare the subject parcel's Acquisition Cost-Per-Acre against the Acquisition Cost-Per-Acre of the **adjacent/neighboring parcels** (retrieved via `get_lot_data` for each lot in `adjacent_parcels`) to benchmark relative land basis across the immediate submarket.

3. **Calculate Monthly Tax Holding Cost (Mandatory Carrying Cost Metric)**:
   - Explicitly calculate and state the **Monthly Tax Holding Cost** by dividing the annual taxes by 12:
     `Monthly Tax Holding Cost = Annual Tax Amount / 12`
   - Show the exact formula and monthly dollar figure for the official county tax assessment (`annual_tax_amount / 12 = $X/month`) as well as for the preliminary `estimated_annual_taxes / 12` if they differ.
   - Analyze the pre-development carrying cost implications for pre-construction budgeting across typical entitlement and permitting horizons (e.g., 12-month, 18-month, and 24-month cumulative tax carrying reserves).
