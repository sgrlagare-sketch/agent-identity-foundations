---
name: analyze-opportunity
description: Investigates a land parcel (and its adjacent parcels, acquisition cost-per-acre, flood/hydrological risks, and tax holding costs) for real estate investment opportunity analysis. Always load this skill along with analyze-flood-risk and analyze-tax-implications.
metadata:
  adk_additional_tools:
    - get_lot_data
    - get_tax_assessment_data
---
# Analyze Opportunity Skill

When asked to investigate a property or analyze an investment opportunity for a subject lot, you MUST follow every step below:

1. **Retrieve Subject Parcel & Tax Assessment Data**:
   - Call `get_lot_data` for the subject `plot_id` to retrieve all county property records (`plot_id`, `parcel_apn`, `acreage`, `zoning`, `last_sale_price`, `estimated_annual_taxes`, `flood_zone`, `environmental_status`, `utility_access`, `topography`, `soil_classification`, `easement_type`, and `adjacent_parcels`).
   - Call `get_tax_assessment_data` using the subject lot's `parcel_apn` to retrieve official county tax assessment records (`annual_tax_amount`, `assessed_land_value`, `total_assessed_value`, `payment_status`).

2. **Retrieve All Neighboring / Adjacent Parcel Data (Mandatory)**:
   - Inspect the `adjacent_parcels` array in the subject lot's record.
   - You MUST call `get_lot_data` for **every** parcel ID listed in `adjacent_parcels` to obtain each neighboring property's exact `flood_zone`, `topography`, `soil_classification`, `easement_type`, `acreage`, `zoning`, and `last_sale_price`.

3. **Derive, State, and Highlight Exact Acquisition Cost-Per-Acre (Mandatory Core Valuation Metric)**:
   - Explicitly calculate and state the exact mathematical **Acquisition Cost-Per-Acre** using the formula:
     `Acquisition Cost-Per-Acre = Last Sale Price / Acreage`
   - Write out the full mathematical derivation equation with exact numbers (e.g., `$450,000 / 1.2 acres = $375,000.00 per acre`) and state the exact figure.
   - **Highlight and emphasize Acquisition Cost-Per-Acre as a core valuation metric** in **both** the `## Executive Summary` (in text and a summary table) and the `## Acquisition Basis and Holding Costs` section.
   - **Benchmark the Acquisition Cost-Per-Acre** against:
     1. The subject parcel's **Assessed Land Value Per Acre** (`assessed_land_value / acreage`).
     2. The **Neighboring Parcels' Acquisition Cost-Per-Acre** (`last_sale_price / acreage` for each adjacent lot in `adjacent_parcels`) to evaluate relative land valuation basis and density value.
   - Also explicitly calculate the **Monthly Tax Holding Cost** (`annual_tax_amount / 12 = $X/month`) and discuss pre-development carrying cost budgeting (e.g., 12- to 24-month holding periods).

4. **Exhaustively Analyze Subject & Neighboring Property Flood Risks and Hydrology (Mandatory)**:
   - Consult the `analyze-flood-risk` and `analyze-tax-implications` skills.
   - For the **subject parcel**, flag the exact FEMA `flood_zone` designation and detail specific vertical construction requirements (Base Flood Elevation [BFE], structural fill dirt, deep foundation pilings/breakaway walls for Zone VE, elevated mechanicals) and NFIP flood insurance impacts.
   - For **all neighboring/adjacent parcels** (`adjacent_parcels`), include a comparative table and detailed narrative in `## Risk Assessment` (and highlight in `## Executive Summary`) that:
     1. Explicitly states each adjacent parcel's `plot_id`, exact FEMA `flood_zone` classification (e.g., `Zone AE (High Risk)`, `Zone VE (Coastal High Hazard)`, `Zone A (100-year Floodplain)`, `Zone X (Minimal Risk)`), `topography` slope grade, `soil_classification`, and `easement_type`.
     2. **Critically analyzes cross-boundary drainage, overland stormwater runoff, and shared hydrological impacts** on the subject parcel's development feasibility—including how slope differentials, soil permeability (clay loam vs. sandy loam vs. bedrock), drainage easements, adjacent floodplain/coastal surge proximity, tailwater/backwater effects, and zero-rise / no-adverse-impact stormwater retention requirements affect site grading, foundation design, and civil engineering CapEx.

5. **Evaluate Infrastructure, Topography, and Environmental Status**:
   - In `## Zoning and Infrastructure Readiness`, connect `utility_access` distances and `topography` slope grade to site preparation feasibility and estimated civil CapEx.
   - In `## Risk Assessment`, report the exact `environmental_status` (Phase I ESA Clean vs. REC flagged vs. Phase II ESA Required) and prescribe exact environmental due diligence steps (Phase II subsurface testing, UST remediation, vapor mitigation, environmental escrow).
