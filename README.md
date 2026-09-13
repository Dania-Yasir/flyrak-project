# Content Decline Ranking for Human Review

**FlyRank ML Internship Capstone — Dania Yasir**

This project builds a machine-learning ranking workflow that helps content and SEO analysts decide **which pages to review first** when review capacity is limited.

The system uses search-performance signals available before a prediction boundary to rank pages by risk of a later decline in Google Search impressions. It is intentionally designed as **decision support**, not as an automatic content-editing system.

> **Operating principle:** Model prioritizes → human investigates → human decides.

---

## What this project does

For each eligible content page, the workflow:

1. Reads approved pseudonymized search-performance data.
2. Builds features using only information available in the first half of March 2026.
3. Defines a later impression-decline proxy using the second half of March.
4. Trains and evaluates a Random Forest ranking model.
5. Validates it with **client-disjoint cross-validation** so complete clients are held out.
6. Produces risk scores and a ranked human-review policy.

The goal is not to predict Google's ranking algorithm and not to prove that editing a page will improve performance.

The goal is narrower:

**Put a denser concentration of likely decline cases near the top of an analyst's review queue.**

---

## Who it is for

This project is useful for:

* SEO and content analysts managing large page portfolios.
* Teams that cannot manually inspect every page regularly.
* ML practitioners interested in leakage-aware ranking and group-based validation.
* Reviewers who want an example of turning an ML score into a human-in-the-loop workflow.

---

## Project question

> Can search-performance signals available in the first half of a month help prioritize which content pages should be reviewed first for possible future decline?

The unit of analysis is one pseudonymized content page within one pseudonymized client.

The final model returns a risk score used to prioritize review.

---

## Data and prediction boundary

The final experiment uses the approved **FlyRank ML Internship warehouse release, build `v20260703`**, accessed through DuckDB and Hugging Face.

* Main table: `fact_content_daily_performance`
* Feature window: **March 1–15, 2026**
* Outcome window: **March 16–31, 2026**
* Final modeling cohort: **61,795 pages**
* Pseudonymized clients: **34**
* Observed decline rate: **32.4%**
* Decline proxy: average daily impressions in the outcome window are more than **20% lower** than in the feature window

Only pre-outcome information is allowed into the model.

Client/content IDs, future-window fields, the target, and target-derived fields are excluded from model inputs.

---

## Architecture

```text
FlyRank Approved Warehouse
        |
        v
March 2026 Search Data
        |
        v
Feature Window
March 1–15
        |
        v
21 Pre-Outcome Features
        |
        v
Random Forest
Full-Signal Model
        |
        v
5-Fold Client-Disjoint Validation
        |
        v
Risk Score
        |
        v
Ranked Review Queue
        |
        v
Human Analyst Review
        |
        v
Investigate / Watch / Monitor


Outcome Window
March 16–31
        |
        v
Evaluation Only
```

The second-half outcome window is used for evaluation only.

It is **not** used as model input.

---

## Key design decisions

### 1. Treat the task as ranking, not generic classification

The operational question is which pages should appear near the top of a limited review queue.

For that reason, the primary metric is **Precision@100**, not overall accuracy.

### 2. Hold out complete clients

Random-row validation allowed client-specific patterns to appear on both the training and validation sides and produced an optimistic result.

The final evaluation therefore uses **5-fold client-disjoint grouped validation**.

### 3. Keep a stronger simplicity check

In addition to the original Week-4 rule baseline, the final evaluation includes a **momentum-only Random Forest**.

This tests whether the additional contextual and trend-shape features add value beyond recent impression movement alone.

---

## V2 / Final Evaluation Results

All three methods below are evaluated on the same target and the same client-disjoint folds.

| Method                          | Mean Precision@100 |    SD |
| ------------------------------- | -----------------: | ----: |
| **Random Forest — Full Signal** |          **74.6%** | 13.6% |
| Random Forest — Momentum Only   |              66.2% | 12.8% |
| Week-4 Rule Baseline            |              35.2% |  6.4% |

The final full-signal model improves mean Precision@100 by **39.4 percentage points** over the Week-4 rule baseline.

Compared with the stronger momentum-only Random Forest, the improvement is **8.4 percentage points**.

A key validation finding is that the same full model reached **90.0% Precision@100** under random-row cross-validation but **74.6%** under client-disjoint validation.

The lower client-disjoint score is the primary result because it better matches the intended use case of generalizing to client groups not seen during training.

### Additional final checks

* Top 10% of the ranked queue captures **20.1%** of observed decline cases.
* Top 20% captures **34.6%**.
* Top 30% captures **48.5%**.
* Lowest-risk decile observed decline rate: **9.1%**.
* Highest-risk decile observed decline rate: **65.1%**.
* **24 of 34 clients** showed positive high-risk concentration in the final client-level summary.

The complete evidence and interpretation are documented in:

[`work/capstone_report.md`](work/capstone_report.md)

---

## Repository guide

| Path                                        | Purpose                                                     |
| ------------------------------------------- | ----------------------------------------------------------- |
| `work/notebooks/capstone.ipynb`             | Concise capstone narrative and final results                |
| `work/notebooks/w03_data_contract.ipynb`    | Data contract, cohort construction, and prediction boundary |
| `work/notebooks/w04_baseline_score.ipynb`   | Transparent rule baseline                                   |
| `work/notebooks/w05_model_training!.ipynb`  | Model development and feature experiments                   |
| `work/notebooks/w06_validation_audit.ipynb` | Client-disjoint validation and leakage audit                |
| `work/notebooks/w07_action_playbook.ipynb`  | Final queue, metrics, and human-review policy               |
| `work/outputs/w07_paper_metrics.json`       | Machine-readable final metrics                              |
| `work/capstone_report.md`                   | Full final model-development and validation report          |
| `work/research_paper.md`                    | Research-paper version of the project                       |
| `DATA_USE.md`                               | Data-use and privacy rules                                  |
| `SETUP.md`                                  | Detailed GitHub, Colab, and Hugging Face setup guidance     |

---

## Setup — Recommended Route: Google Colab

This is the easiest route for a new reviewer because it does not require a local Python installation.

### Step 1 — Get the repository

Open:

`https://github.com/Dania-Yasir/flyrak-project`

### Step 2 — Request access to the FlyRank dataset

Create a free Hugging Face account and request or accept access to:

`FlyRank/internship-warehouse`

Then create a **Read** access token.

Do not commit the token to GitHub or type it into a public notebook cell.

See `SETUP.md` for the detailed data-access instructions.

### Step 3 — Open the final notebook in Colab

Open:

`work/notebooks/capstone.ipynb`

Or use this Colab link:

https://colab.research.google.com/github/Dania-Yasir/flyrak-project/blob/main/work/notebooks/capstone.ipynb

### Step 4 — Full technical reproduction

For the complete workflow, run these notebooks in order:

1. `work/notebooks/w03_data_contract.ipynb`
2. `work/notebooks/w04_baseline_score.ipynb`
3. `work/notebooks/w05_model_training!.ipynb`
4. `work/notebooks/w06_validation_audit.ipynb`
5. `work/notebooks/w07_action_playbook.ipynb`

When a notebook requests Hugging Face authentication, provide the Read token through the secure prompt or Colab Secrets.

Do not hard-code the token.

---

## Local setup

### 1. Clone the repository

```bash
git clone https://github.com/Dania-Yasir/flyrak-project.git
cd flyrak-project
```

### 2. Create a virtual environment

#### Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

#### macOS / Linux

```bash
python -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
pip install jupyter
```

Core dependencies include:

* pandas
* NumPy
* scikit-learn
* matplotlib
* DuckDB
* ReportLab
* huggingface_hub

### 4. Authenticate for the gated dataset

Accept the dataset terms in Hugging Face first.

Create a **Read** token and provide it securely when the notebooks request access.

### 5. Launch Jupyter

```bash
jupyter lab
```

Run the technical notebooks in the order listed above.

The final deterministic workflow uses:

* Random seed: `42`
* Validation: `5` client-disjoint folds
* Feature window: March 1–15, 2026
* Outcome window: March 16–31, 2026
* Decline threshold: `-20%`

---

## Usage example

The project is intended to produce a **review priority**, not an automatic edit decision.

The final policy uses the within-fold risk percentile:

| Risk percentile | Operational use     |
| --------------- | ------------------- |
| `>= 0.80`       | Active human review |
| `0.50–0.79`     | Watchlist           |
| `< 0.50`        | Monitor             |

For a high-risk page, observable pre-outcome signals can route the page to one of several investigation types:

* `CONTENT_REFRESH_REVIEW`
* `SERP_AND_INTENT_REVIEW`
* `TITLE_META_CTR_REVIEW`
* `MANUAL_DIAGNOSTIC_REVIEW`
* `HUMAN_REVIEW_BEFORE_EDIT`

These labels mean:

**review and investigate**

not:

**automatically rewrite this page**

---

## Limitations

This project has several important limitations.

### 1. Proxy target

The target measures future impression decline.

It does not directly measure content quality or prove that a page should be refreshed.

### 2. Coverage selection

Only **61,795 of 331,437** March page-level candidates met the final complete-coverage requirements.

The results should therefore not automatically be generalized to every page in the warehouse.

### 3. Single primary month

March 2026 is the main completed experiment.

A completed next-month temporal holdout is not part of the final evidence.

### 4. Limited client count

The final cohort contains **34 pseudonymized clients**.

### 5. Unequal client sizes

Client-disjoint folds can differ substantially in page count and target prevalence.

### 6. Client heterogeneity

Performance is not equally strong across all held-out client groups.

The full model's fold-level Precision@100 standard deviation is **13.6%**.

### 7. Portfolio-context dependency

Client-relative features assume that enough same-client pages are available to define current portfolio context.

### 8. No causal interpretation

Feature importance and model scores show associations useful for ranking.

They do not explain why a page declined.

### 9. No Google-algorithm claim

This model does **not** predict Google's ranking algorithm.

### 10. No intervention-effect claim

The experiment does not prove that refreshing or rewriting a high-risk page will improve future performance.

Before making stronger production claims, the project would benefit from additional **temporal holdout and prospective validation**.

---

## Data safety

Only approved pseudonymized internship data is used.

Public project outputs do not include:

* Client names
* Domains
* URLs
* Private search queries
* Access credentials

Do not commit raw private client data or Hugging Face tokens to this repository.

See `DATA_USE.md` for the full rules.

---

## AI Transparency

I used **ChatGPT as an AI assistant** during this project for brainstorming, code and debugging support, reviewing implementation choices, and improving documentation.

I treated AI-generated suggestions as drafts rather than automatically correct answers.

I personally ran and reviewed the project notebooks and outputs, checked the prediction boundary and leakage controls, and verified the final metrics reported here against the repository evidence.

---

## What I would build next

The next validation step would be a true **temporal holdout** using a later month.

After that, I would test the ranked queue prospectively and monitor ranking quality per client over time.

I would not treat one aggregate score as a permanent production accuracy number.

---

## Final project artifacts

* [Final Capstone Report](work/capstone_report.md)
* [Capstone Notebook](work/notebooks/capstone.ipynb)
* [Final Metrics](work/outputs/w07_paper_metrics.json)
* [Research Paper](work/research_paper.md)
* [Setup Guide](SETUP.md)

---

## Demo Video

The **3–5 minute live demo video** for Assignment 8.1 will be added here after recording.

---

**Author:** Dania Yasir
**Project:** FlyRank ML Internship Capstone
**Repository:** Dania-Yasir/flyrak-project
