---
name: analyze-flood-risk
description: Investigates flood zone designations and environmental risks for real estate investment opportunity analysis.
metadata:
  adk_additional_tools:
    - get_lot_data
---
# Analyze Flood Risk Skill

When evaluating environmental, flood, or geographic risks for a subject lot, you MUST evaluate **both** the subject parcel AND all neighboring/adjacent properties:

1. **Subject Parcel Flood Zone & Engineering Impact Analysis**:
   - Explicitly flag the subject parcel's exact FEMA flood zone designation from `flood_zone` (e.g., `Zone AE (High Risk)`, `Zone VE (Coastal High Hazard)`, `Zone A (100-year Floodplain)`, or `Zone X (Minimal Risk)`).
   - Detail specific vertical construction and site engineering requirements (e.g., Base Flood Elevation [BFE] compliance, engineered structural fill dirt, deep foundation pilings and breakaway walls for Zone VE coastal wave action, elevated mechanical/electrical pads, stormwater detention) and National Flood Insurance Program (NFIP) / private commercial flood insurance cost impacts.

2. **Exhaustive Neighboring / Adjacent Property Flood Risk Investigation (Mandatory)**:
   - Check the `adjacent_parcels` array in the subject parcel's data and call `get_lot_data` for **every** neighboring/adjacent parcel ID if not already retrieved.
   - Explicitly report the exact `plot_id`, specific FEMA `flood_zone` classification, `topography` (slope grade), `soil_classification`, and `easement_type` (especially any drainage easements) for **each neighboring/adjacent property**. Include a dedicated markdown comparison table comparing the subject lot and its neighboring lots.

3. **Critical Comparative Hydrological, Drainage, and Runoff Analysis (Mandatory)**:
   - Critically analyze how the neighboring properties' flood zones, topography slopes, soil drainage characteristics, and easements impact the subject parcel's hydrology and development feasibility:
     - **Cross-Boundary Drainage & Overland Runoff**: Evaluate how slope differentials (e.g., steeper adjacent grades vs. flatter subject grade, or vice versa) and soil permeability (e.g., clay loam with shrink-swell vs. sandy loam vs. bedrock) drive stormwater sheet flow, runoff velocity, and water accumulation across parcel boundaries.
     - **Floodplain Continuity & Hydrostatic/Backwater Risk**: If any adjacent parcel lies in a high-hazard flood zone (`Zone AE`, `Zone VE`, `Zone A`), analyze the risk of floodplain encroachment, tailwater/backwater effects on the subject parcel's storm sewer outfalls, elevated water tables, and shared coastal/riverine surge exposure—even if the subject parcel itself is in `Zone X`. Conversely, if the subject parcel is in a high-risk flood zone and neighbors are in `Zone X` or have steeper slopes, analyze how grading or importing fill dirt on the subject lot could displace floodwaters onto adjacent properties ("no adverse impact" / zero-rise floodplain regulations requiring compensatory storage).
     - **Shared Hydrological Mitigation & Development Feasibility Impacts**: Prescribe actionable civil engineering mitigations (perimeter interceptor swales, retention/detention basins, retaining walls, tie-ins to drainage easements, joint neighbor drainage agreements, or FEMA LOMA/LOMR survey requirements) and quantify their impact on site preparation CapEx and development feasibility.

4. **Environmental Status (Phase I / Phase II ESA) Due Diligence**:
   - Explicitly report the exact `environmental_status` (e.g., `Phase I ESA - Clean`, `Phase I ESA - Recognized Environmental Condition (REC flagged)`, or `Phase II ESA Required (Historical Underground Storage Tank)`) and prescribe concrete environmental due diligence next steps (such as Phase II subsurface soil/groundwater sampling, geophysical UST sweeps, vapor intrusion barriers, remedial cost caps, and environmental indemnity clauses).
