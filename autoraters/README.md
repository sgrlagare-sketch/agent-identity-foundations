# Autorater Reliability Analysis & Agent Skill Hill-Climbing — Step-by-Step Manual

This repository contains the **Real Estate Investment Opportunity Analysis Agent (`re_analyst`)** built with the Google Agent Development Kit (ADK), the **Autorater Evaluation Harness (`eval_harness`)** powered by the Vertex AI Gen AI Evaluation Service, and a complete step-by-step manual covering Tasks 1 through 6 of the lab.

---

## Architecture & Repository Overview

| Component | Path | Purpose |
| :--- | :--- | :--- |
| **ADK Real Estate Agent** | `re_analyst/agent.py` | Root `re_opportunity_analyst` agent configured with `SkillToolset` and county property/tax lookup tools |
| **Opportunity Investigation Skill** | `re_analyst/skills/analyze-opportunity/SKILL.md` | Orchestrates subject & adjacent lot lookup, exact acquisition cost-per-acre derivation, and report synthesis |
| **Flood & Hydrology Skill** | `re_analyst/skills/analyze-flood-risk/SKILL.md` | Evaluates subject & neighboring FEMA flood zones, cross-boundary runoff, tailwater/backwater risk, and Phase I/II ESA status |
| **Tax & Valuation Skill** | `re_analyst/skills/analyze-tax-implications/SKILL.md` | Computes & benchmarks `Acquisition Cost-Per-Acre` (`Last Sale Price / Acreage`) and `Monthly Tax Holding Cost` (`Annual Tax / 12`) |
| **Trace Generator** | `eval_harness/create_re_agent_evalset.py` | Executes `re_analyst` concurrently across target lots to generate ADK `.evalset.json` traces |
| **Multi-Pass Evaluator** | `eval_harness/run_geap_eval.py` | Dispatches $k$-pass adaptive rubric evaluations (19 criteria in `eval_harness/rubrics.json`) to Vertex AI |
| **Reliability Analyzer** | `eval_harness/single_criterion_analysis.py` & `all_criteria_analysis.py` | Computes pairwise agreement, report/pass standard deviations, and Krippendorff's Alpha across passes |
| **Rationale Inspector** | `eval_harness/explain_criteria_results.py` | Prints LLM judge explanations filtered by score threshold (`--max_score_to_show`) |
| **Hill-Climbing Runner** | `eval_harness/run_hillclimb.py` | Automated iteration script generating `re_analyst/hillclimb_X.evalset.json` and evaluating `cost-per-acre` & `neighboring-flood-risk` |

---

## Task 1: Environment & Project Setup

### 1.1 Authenticate and Copy Lab Files from Cloud Storage

```bash
gcloud auth application-default login
export PROJECT_ID=$(gcloud config get-value project)
gcloud config set project ${PROJECT_ID}

cd ~
gcloud storage cp -r gs://${PROJECT_ID}-bucket/autoraters .
```

### 1.2 Install Python 3.13 (via `pyenv`) and Dependencies

```bash
sudo apt-get update
sudo apt-get install -y libreadline-dev
curl https://pyenv.run | bash

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc

pyenv install 3.13
pyenv global 3.13

python3 -m pip install -r ~/autoraters/requirements.txt
pip install "google-adk[eval]"
```

### 1.3 Configure `.env`

```bash
cd ~/autoraters
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project)
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF
```
*(Note: If Gemini capacity issues occur, you can set `MODEL=gemini-3.1-flash-lite` in `.env`.)*

---

## Task 2: Generate Baseline Agent Evaluation Traces

Execute the `re_analyst` agent over the 10 target lots defined in `eval_harness/hillclimb_lots.txt`:

```bash
cd ~/autoraters
python3 -m eval_harness.create_re_agent_evalset \
  --lots-file eval_harness/hillclimb_lots.txt \
  --out ./re_analyst/hillclimb_lots.evalset.json \
  --agent-dir ./re_analyst
```

*(Optional)* Launch the ADK Web UI to inspect the generated agent traces:

```bash
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```

---

## Task 3 & 4: Run Multi-Pass Autorater Evaluation (`k=3`)

### 3.1 Dry-Run Verification

```bash
cd ~/autoraters
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --dry-run \
  --k 3
```

### 3.2 Run Live 3-Pass Evaluation on Vertex AI

```bash
cd ~/autoraters
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --k 3
```

---

## Task 5: Analyze Autorater Intra-Rater Reliability

Using the pre-computed large evaluation dataset (`runs/geap_calls_and_responses_for_analysis.json`: 32 lot IDs graded on 19 criteria across 15 repeated evaluation passes):

### 5.1 Step 1 & 2: Deep-Dive into a Single Criterion

```bash
cd ~/autoraters
python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion environmental-esa
```

Key summary metrics for `environmental-esa`:
- **Mean Rating**: `4.5764`
- **Widest Range Single Report**: `3` (`LOT-1028`)
- **StdDev of Report Means (`sd report means`)**: `0.7042`
- **Mean Pairwise Percent Agreement**: `79.10%`
- **Krippendorff's Alpha**: `0.897`

### 5.2 Step 3: View Metrics Across All 19 Criteria

```bash
cd ~/autoraters
python3 -m eval_harness.all_criteria_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json
```

#### Full Output Table (`all_criteria_analysis.py`)

| `criterion_id` | `mean` | `min rating` | `max rating` | `max range report` | `max range pass` | `sd report means` | `sd pass means` | `pairwise %_agree` | `alpha` | Assessment |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **`monthly-tax-holding`** | 4.3542 | 3 | 5 | 1 | 2 | **0.8872** | 0.0172 | **99.17%** | **0.997** | **Reliable (Selected)** |
| **`flood-zone-flagging`** | 4.8286 | 4 | 5 | 1 | 1 | **0.3357** | 0.0366 | **94.41%** | **0.897** | **Reliable (Selected)** |
| **`environmental-esa`** | 4.5764 | 2 | 5 | 3 | 3 | **0.7042** | 0.0389 | **79.10%** | **0.897** | **Reliable (Selected)** |
| `risk-assessment` | 4.9188 | 3 | 5 | 2 | 2 | 0.1843 | 0.0309 | 96.25% | 0.803 | Good Secondary Candidate |
| `utility-topography` | 4.7917 | 4 | 5 | 1 | 1 | 0.3081 | 0.0484 | 82.57% | 0.751 | Good Secondary Candidate |
| `general-tone` | 5.0000 | 5 | 5 | 0 | 0 | 0.0000 | 0.0000 | 100.00% | 1.000 | Homogeneity Trap (`sd=0`) |
| `logical-flow` | 5.0000 | 5 | 5 | 0 | 0 | 0.0000 | 0.0000 | 100.00% | 1.000 | Homogeneity Trap (`sd=0`) |
| `professionalism` | 5.0000 | 5 | 5 | 0 | 0 | 0.0000 | 0.0000 | 100.00% | 1.000 | Homogeneity Trap (`sd=0`) |
| `readability` | 5.0000 | 5 | 5 | 0 | 0 | 0.0000 | 0.0000 | 100.00% | 1.000 | Homogeneity Trap (`sd=0`) |
| `neighboring-flood-risk` | 1.0316 | 1 | 2 | 0 | 1 | 0.1768 | 0.0044 | 100.00% | 1.000 | Baseline Floor Trap (31/32 score 1) |
| `data-density` | 4.9979 | 4 | 5 | 1 | 1 | 0.0118 | 0.0081 | 99.58% | 0.499 | Near-Homogeneous (`alpha=0.499`) |
| `professional-tone` | 4.9958 | 4 | 5 | 1 | 1 | 0.0164 | 0.0110 | 99.17% | 0.498 | Near-Homogeneous (`alpha=0.498`) |
| `cost-per-acre` | 1.3771 | 1 | 3 | 1 | 2 | 0.3694 | 0.0970 | 64.61% | 0.386 | Low Agreement |
| `report-length` | 4.1833 | 3 | 5 | 2 | 2 | 0.2546 | 0.0666 | 68.59% | 0.360 | Low Agreement |
| `document-formatting` | 4.5396 | 4 | 5 | 1 | 1 | 0.2194 | 0.0567 | 65.46% | 0.308 | Low Agreement |
| `persuasiveness` | 4.3042 | 4 | 5 | 1 | 1 | 0.1945 | 0.0680 | 65.34% | 0.184 | Unreliable |
| `core-specifications` | 4.5917 | 4 | 5 | 1 | 1 | 0.1303 | 0.1233 | 52.87% | 0.025 | Unreliable |
| `analyst-insight` | 4.2125 | 3 | 5 | 2 | 2 | 0.1232 | 0.1385 | 66.22% | 0.007 | Unreliable |
| `vocabulary-choice` | 4.1000 | 4 | 5 | 1 | 1 | 0.0726 | 0.0870 | 81.25% | -0.004 | Unreliable |

### 5.3 Step 4: Write `best_criteria_submission.txt` and Upload to Cloud Storage

```bash
cd ~/autoraters
echo "monthly-tax-holding,flood-zone-flagging,environmental-esa" > best_criteria_submission.txt
gcloud storage cp best_criteria_submission.txt gs://$(gcloud config get-value project)-bucket/best_criteria_submission.txt
```

---

## Task 6: Hill-Climb Agent Skills to Raise Autorater Criteria Scores

### 6.1 Step 1: Run First Hill-Climbing Baseline (`hillclimb_1`)

```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```

Baseline output (`hillclimb_1`):
```text
=========================================
📊 Hill Climbing Results: hillclimb_1
=========================================
 • cost-per-acre             Mean Score: 1.5000
 • neighboring-flood-risk    Mean Score: 1.4000
=========================================
```

### 6.2 Step 2: Inspect LLM Judge Rationales

```bash
cd ~/autoraters
python3 -m eval_harness.explain_criteria_results \
   --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
   --criterion cost-per-acre \
   --max_score_to_show 4.0

python3 -m eval_harness.explain_criteria_results \
   --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
   --criterion neighboring-flood-risk \
   --max_score_to_show 4.0
```

### 6.3 Step 3: Update Agent Skills in `re_analyst/skills/`

#### 1. Update `re_analyst/skills/analyze-opportunity/SKILL.md`

```bash
cat << 'EOF' > ~/autoraters/re_analyst/skills/analyze-opportunity/SKILL.md
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
EOF
```

#### 2. Update `re_analyst/skills/analyze-flood-risk/SKILL.md`

```bash
cat << 'EOF' > ~/autoraters/re_analyst/skills/analyze-flood-risk/SKILL.md
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
EOF
```

#### 3. Update `re_analyst/skills/analyze-tax-implications/SKILL.md`

```bash
cat << 'EOF' > ~/autoraters/re_analyst/skills/analyze-tax-implications/SKILL.md
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
EOF
```

### 6.4 Step 4: Re-Run Hill-Climbing Evaluation (`hillclimb_2`)

```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```

Final output (`hillclimb_2`):
```text
=========================================
📊 Hill Climbing Results: hillclimb_2
=========================================
 • cost-per-acre             Mean Score: 5.0000
 • neighboring-flood-risk    Mean Score: 5.0000
=========================================
```
