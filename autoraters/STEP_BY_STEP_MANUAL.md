# Autorater Reliability Analysis & Agent Skill Hill-Climbing — Step-by-Step Manual

This document provides the complete step-by-step instructions, commands, analysis outputs, and updated ADK skill files for **Autorater Intra-Rater Reliability Analysis (Task 5)** and **Hill-Climbing Agent Skills (Task 6)**.

---

## Task 1: Environment & Project Setup

```bash
gcloud auth application-default login
export PROJECT_ID=$(gcloud config get-value project)
gcloud config set project ${PROJECT_ID}

cd ~
gcloud storage cp -r gs://${PROJECT_ID}-bucket/autoraters .

# Install Python 3.13 via pyenv and project dependencies
sudo apt-get update && sudo apt-get install -y libreadline-dev
curl https://pyenv.run | bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
pyenv install -s 3.13
pyenv global 3.13

python3 -m pip install -r ~/autoraters/requirements.txt
pip install "google-adk[eval]"

cd ~/autoraters
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF
```

---

## Task 2: Generate Baseline Agent Evaluation Traces

```bash
cd ~/autoraters
python3 -m eval_harness.create_re_agent_evalset \
  --lots-file eval_harness/hillclimb_lots.txt \
  --out ./re_analyst/hillclimb_lots.evalset.json \
  --agent-dir ./re_analyst
```

---

## Task 3 & 4: Run Multi-Pass Autorater Evaluation (`k=3`)

```bash
cd ~/autoraters
# Dry run
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --dry-run \
  --k 3

# Live 3-pass evaluation
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --k 3
```

---

## Task 5: Analyze Autorater Intra-Rater Reliability

### Step 1 & 2: Inspect a Single Criterion (`environmental-esa`)

```bash
cd ~/autoraters
python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion environmental-esa
```

### Step 3: Compare All 19 Criteria Simultaneously

```bash
cd ~/autoraters
python3 -m eval_harness.all_criteria_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json
```

### Step 4: Submit the 3 Most Reliable Criteria

Selected criteria with high Krippendorff's Alpha and non-zero report score variance (`sd report means`):
- `monthly-tax-holding` (`alpha = 0.997`, `pairwise %_agree = 99.17%`, `sd report means = 0.8872`)
- `flood-zone-flagging` (`alpha = 0.897`, `pairwise %_agree = 94.41%`, `sd report means = 0.3357`)
- `environmental-esa` (`alpha = 0.897`, `pairwise %_agree = 79.10%`, `sd report means = 0.7042`)

```bash
cd ~/autoraters
echo "monthly-tax-holding,flood-zone-flagging,environmental-esa" > best_criteria_submission.txt
gcloud storage cp best_criteria_submission.txt gs://$(gcloud config get-value project)-bucket/best_criteria_submission.txt
```

---

## Task 6: Hill-Climb Agent Skills to Raise Autorater Scores

### Step 1: Run Initial Hill-Climbing Baseline (`hillclimb_1`)

```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```
- `cost-per-acre`: **1.5000**
- `neighboring-flood-risk`: **1.4000**

### Step 2: Inspect Judge Explanations

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

### Step 3: Modify Agent Skills (`re_analyst/skills/*/SKILL.md`)

Update the three skill markdown files:
1. `re_analyst/skills/analyze-opportunity/SKILL.md`
2. `re_analyst/skills/analyze-flood-risk/SKILL.md`
3. `re_analyst/skills/analyze-tax-implications/SKILL.md`

### Step 4: Run Hill-Climbing Iteration (`hillclimb_2`)

```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```
- `cost-per-acre`: **5.0000** (Target: > 4.0)
- `neighboring-flood-risk`: **5.0000** (Target: > 4.0)
