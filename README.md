# Wallet Risk & Behavior Analytics

**End‑to‑end exploration of how to score and segment crypto wallets using on‑chain data, with unsupervised clustering (Part A) and supervised fraud modeling (Part B).**

### Explore the Project
[![Website](https://img.shields.io/badge/Website-GitHub_Pages-1db954?style=flat&logo=google-chrome&logoColor=white)](https://jessicabat.github.io/crypto-risk-scoring/)
[![Dashboard](https://img.shields.io/badge/Dashboard-Looker_Studio-4285F4?style=flat&logo=google-looker-studio&logoColor=white)](https://lookerstudio.google.com/s/vub9BwmCKDU)

---

## Table of Contents

1. [Project Overview](#project-overview)  
2. [Part A – Behavioral Segmentation (Unsupervised)](#part-a--behavioral-segmentation-unsupervised)  
   - [Data Pipeline](#1-data-pipeline)  
   - [Feature Engineering](#2-feature-engineering)  
   - [Clusters and Wallet Personas](#3-clusters-and-wallet-personas)  
   - [Operational Dashboard](#4-operational-dashboard)  
3. [Part B – Fraud Detection (Supervised)](#part-b--fraud-detection-supervised)  
   - [Dataset & Problem Framing](#1-dataset--problem-framing)  
   - [Modeling Scenarios (A vs B)](#2-modeling-scenarios-a-vs-b)  
   - [Precision–Recall & Financial Impact](#3-precisionrecall--financial-impact)  
   - [Explainability & Error Analysis](#4-explainability--error-analysis)  
4. [How This Fits a Real Risk Engine](#how-this-fits-a-real-risk-engine)  
5. [Tech Stack](#tech-stack)  

---

## Project Overview

Crypto exchanges and fintech apps face a similar problem: when a new wallet shows up, there is very little labeled history, but the platform still has to make risk decisions (limits, extra checks, monitoring).

This project is a small prototype of how a platform could combine:

1. **Part A – Behavioral Segmentation:** Use raw Ethereum transactions and token transfers to cluster wallets into behavioral “types,” then reason about risk and product policies for each segment.  
2. **Part B – Fraud Detection:** Train supervised models on a labeled Ethereum fraud dataset (ETFD) and study precision–recall tradeoffs, financial cost, and how scores would be used in a real risk engine.

---

## Part A – Behavioral Segmentation (Unsupervised)

**Goal:** Start from unlabeled Ethereum data and see what kinds of wallet behaviors exist, then map those behaviors to simple, practical risk policies a consumer crypto platform could use.

### 1. Data Pipeline

- **Source:** Google BigQuery public Ethereum dataset (`bigquery-public-data.crypto_ethereum`).  
- **Scope:** Roughly 1.2M active wallets over a recent ~1‑month window in Q4 2025 (recent enough to be interesting, small enough to work with).  
- **Tables used:**
  - `transactions` – native ETH transfers.
  - `token_transfers` – ERC‑20 / ERC‑721 token movements.

Instead of downloading raw CSVs, I wrote SQL directly in BigQuery to aggregate to the wallet level. The query:

- Unions `from_address` and `to_address` into a single `address` field.  
- Computes basic statistics per wallet (transaction counts, ETH sent/received, unique counterparties).  
- Joins in token activity from `token_transfers` (e.g., number of token transfers, number of distinct token contracts touched).

This produces one row per wallet with features describing its behavior over the window. The SQL query is in [`data/query.sql`](data/query.sql).

### 2. Feature Engineering

Simple counts and sums are a good start, but they don’t fully capture risky or unusual behavior. I engineered several additional features inspired by how fraud and abuse often show up in transaction patterns:

| Feature | Intuition | What it can indicate |
| :--- | :--- | :--- |
| **Flow ratio / “mule score”** | Compare total in vs total out for a wallet. Wallets that mostly pass funds straight through (in ≈ out, low residual balance) behave like intermediaries. | Potential “pass‑through” or money‑mule style behavior. |
| **Fan‑out ratio** | Number of unique counterparties relative to total transactions. | High fan‑out (one-and-done with many recipients) can look like spam, airdrops, or Sybil activity. |
| **Gas usage and gas price** | Average gas used and gas price for outgoing transactions. | Distinguishes simple transfers (~21k gas) from complex contract interactions and high-priority / bot‑like activity. |
| **Activity window / “tenure”** | Difference between first and last transaction timestamps in the window. | Many transactions in a very short window can look bursty or automated; longer, steady histories tend to be less suspicious. |
| **Token activity** | Number of token transfers and number of distinct tokens moved. | Whether the wallet behaves more like an ETH‑only user, a DeFi/NFT power user, or something in between. |

These sit on top of core signals like `n_sent_tx`, `n_received_tx`, `total_sent_eth`, `total_received_eth`, `n_unique_counterparties`, and simple token-transfer counts.

### 3. Clusters and Wallet Personas

K‑Means clustering on log-scaled and robust-scaled features suggested **k = 4** as a reasonable balance between simplicity and expressiveness.

The clusters are not ground truth labels, but they give a useful mental model for different wallet behaviors:

#### 🐋 Cluster 2 – Large, High‑Volume Wallets (~0.2%)

- High ETH volumes and frequent activity.  
- Interact with many counterparties.  
- Likely to include institutional, exchange, or sophisticated trading wallets.

**Risk / Policy idea:**  
Avoid automatic blocks. Treat these wallets as “high‑touch”: stronger monitoring and enhanced due diligence, but be careful about adding friction that could push them away. Use manual review for anomalous behavior rather than blanket rules.

---

#### 🦄 Cluster 1 – DeFi‑Heavy / Contract Users (~1–2%)

- Lower raw ETH volume than Cluster 2 but more complex behavior:  
  - Higher gas usage per tx.  
  - Many token transfers and multiple distinct token contracts.  
- Likely to be DeFi/NFT users or bots interacting with contracts frequently.

**Risk / Policy idea:**  
Focus on account protection (e.g., strong authentication for large withdrawals) and clear UX around contract permissions, rather than aggressive blocking of normal activity.

---

#### 🛒 Cluster 3 – Active Retail Users (a few %)

- Moderate ETH volumes and transaction counts.  
- Some token activity but not too many different tokens.  
- Fairly regular activity across the window.

**Risk / Policy idea:**  
These look like “core” retail customers. Reasonable buy/fund limits that increase with clean history and standard KYC/monitoring.

---

#### 🤖 Cluster 0 – Long Tail / Low-Value / Noisy Wallets (majority)

- Many wallets with very small balances or just a handful of transactions.  
- Often higher fan‑out ratios and short activity spans.  
- This group likely includes:
  - Bots, airdrop participants, experimental wallets, and one‑time users.

**Risk / Policy idea:**  
Automate as much as possible: strict KYC and simple rules for this group to keep human review focused on higher‑value segments. The goal is to keep operational overhead low while still catching obviously problematic behavior.

### 4. Operational Dashboard

To make the clusters and features easier to explore, I built a dashboard in **Looker Studio** (link at the top of this README).

- Downsampled the large “long tail” cluster while keeping all high‑volume segments to keep the dashboard responsive.  
- Used log scales (e.g., for activity vs wealth) to reflect the power‑law nature of crypto activity.  
- For each cluster, the dashboard shows typical ranges for key features and a few example wallets.

The goal is not to build a production tool, but to show how analytics and product/risk teams could explore wallet segments and design different policies and UX treatments for each.

---

## Part B – Fraud Detection (Supervised)

**Goal:** Use the **ETFD (Ethereum Transaction Fraud Detection)** dataset to build and explain transaction-level fraud models, and reason about how a crypto platform would use them in practice.

### 1. Dataset & Problem Framing

For supervised modeling, I used the **ETFD** dataset from Kaggle.

- **Size:** 85,003 labeled rows.  
- **Label:** `Fraud` (0 = normal, 1 = fraud), roughly **50/50** in this dataset.  
- **Features (14 numeric + label):**
  - Time / block: `Month`, `Day`, `Hour`, `blockNumber`, `confirmations`.  
  - Received behavior: `mean_value_received`, `variance_value_received`, `total_received`, `time_diff_first_last_received`.  
  - Sent behavior: `total_tx_sent`, `total_tx_sent_unique`.  
  - Malicious-interaction counters: `total_tx_sent_malicious`, `total_tx_sent_malicious_unique`, `total_tx_received_malicious_unique`.

Each row can be thought of as:

> “One transaction I want to classify, plus a snapshot of the address’s historical behavior and how often it has interacted with known malicious addresses.”

To avoid learning brittle time shortcuts, I **dropped** `blockNumber` and `confirmations` before modeling, since they mainly encode when synthetic fraud events were generated rather than intrinsic risk behavior.

Train / validation / test splits were stratified (70/15/15) on the balanced dataset.

### 2. Modeling Scenarios (A vs B)

I trained tree-based models (XGBoost and Random Forest) under two different feature scenarios:

- **Scenario A – Full “Blacklist” Context**  
  Uses all remaining features, including malicious counters:
  - `total_tx_sent_malicious`  
  - `total_tx_sent_malicious_unique`  
  - `total_tx_received_malicious_unique`

- **Scenario B – Behavior Only**  
  Drops the malicious counters and relies purely on broader behavior:
  - `Month`, `Day`, `Hour`  
  - `mean_value_received`, `variance_value_received`, `total_received`  
  - `time_diff_first_last_received`, `total_tx_sent`, `total_tx_sent_unique`

This gives two complementary questions:

1. “If I already know a lot about who the malicious neighbors are, how well can I operationalize that?”  
2. “Without that explicit prior knowledge, how far can I get just from behavioral patterns?”

**Validation performance (balanced set):**

- **Scenario A (XGBoost):**
  - ROC–AUC ≈ **0.9993**.  
  - At recall ≈ 0.90 (threshold ≈ 0.975):  
    - Very high precision (≈ 0.998).  
    - ~13 false positives, ~639 false negatives in the validation split.

- **Scenario B (XGBoost):**
  - ROC–AUC ≈ **0.9799**.  
  - At recall ≈ 0.80 (threshold ≈ 0.81):  
    - Precision ≈ 0.97.  
    - ~175 false positives, ~1,277 false negatives in the validation split.

Random Forest behaved similarly, with slightly lower AUC in both scenarios.

Interpretation:

- Scenario A is almost trivial: if you know how much an address has interacted with already-labeled malicious addresses, the model can nearly separate fraud from non-fraud.  
- Scenario B is harder and closer to a “behavior-only” model, showing more realistic tradeoffs once you drop the malicious counters.

An interactive ROC overlay (Asset 1) in the repo shows Scenario A vs Scenario B on the same plot.

### 3. Precision–Recall & Financial Impact

ROC–AUC is useful, but risk teams care about operational tradeoffs: how many real frauds they catch, how many legitimate users they annoy, and what that costs.

To capture this, I:

- Computed precision–recall curves for each model.  
- Picked operating points by **target recall** (e.g., ~80% recall in Scenario B).  
- Defined a simple **cost model**:

  - Cost per **false positive (FP)**: \$50  
    - Blocking or heavily frictioning a legitimate user → churn, support tickets, lost future revenue.  
  - Cost per **false negative (FN)**: \$500  
    - Missing a fraud → chargebacks, direct financial loss, remediation.

#### Realistic 5% Fraud Test

ETFD is ~50/50 fraud vs normal, which is convenient for training but not realistic. Real fraud rates are closer to 1–5%. To approximate this:

- Took the held-out test split (originally 50/50).  
- Kept all normal transactions.  
- Downsampled fraud transactions to create a test set with **~5% fraud / 95% normal**.

On this **TEST_REALISTIC (5% Fraud)** set:

- **Do-nothing baseline (never flag anything):**

  - FP = 0, FN ≈ 330  
  - Estimated cost ≈ 330 × \$500 = **\$165,000**

- **Scenario B – Behavior-only XGBoost (Threshold ≈ 0.81, chosen on validation):**

  - Precision ≈ **0.98**  
  - Recall ≈ **0.61**  
  - FP = 166, FN = 65  
  - Estimated cost ≈ 166 × \$50 + 65 × \$500 ≈ **\$40,800**

Under this simple cost model, the behavior-only model reduces expected loss by roughly **\$124,000 (~75% reduction)** compared to doing nothing, while keeping precision extremely high. A stacked bar chart (Asset 6) in the repo visualizes the baseline vs model costs and highlights net savings.

I also evaluated Scenario A and Random Forest variants on the same test, which show even lower cost when malicious-context features are allowed.

### 4. Explainability & Error Analysis

To understand why the model makes the decisions it does—and where it fails—I focused on **Scenario B (XGBoost)**, since it uses only behavior features.

#### Global drivers (Scenario B)

Using tree-based feature importance and SHAP, the top global drivers were:

1. `total_tx_sent` – number of transactions sent.  
2. `time_diff_first_last_received` – how long the address has been receiving funds.  
3. `total_received` – total value received.  
4. `total_tx_sent_unique` – number of distinct recipients.  
5. Calendar features (`Month`, `Day`, `Hour`) and value statistics.

In general, the model tends to flag as higher risk:

- Addresses with **short activity windows** (new/short-lived).  
- Addresses with **few, bursty sends**.  
- Addresses with **little or no prior received value**.

This matches the intuition of “fresh accounts quickly pushing funds out.”

#### Local SHAP examples

For representative cases:

- A **true positive (caught fraud)** transaction:
  - The address had a single outgoing transaction, essentially no prior inflows, and a very short activity window.  
  - SHAP showed strong positive contributions from `total_tx_sent`, `Day`, and `time_diff_first_last_received` (close to zero), pushing the score towards fraud.

- A **false positive (legitimate but flagged)** transaction:
  - Very similar pattern: new address, short-lived activity, one outgoing transaction.  
  - Behavior overlaps heavily with the fraud pattern above, so the model reasonably flags it, but in ground truth it is normal.

From a product/risk perspective, these are perfect candidates for **step-up verification flows** (e.g., extra checks or 2FA) rather than immediate hard blocks.

#### Outcome-wise distributions

I also split the validation set into TP / FP / FN / TN and examined key features:

- **TN (true normals):**
  - Much **longer activity windows** (median `time_diff_first_last_received` ≫ 0).  
  - Much higher `total_tx_sent` and `total_received` on average.

- **TP, FP, FN:**
  - Often have `time_diff_first_last_received` ≈ 0.  
  - Lower `total_tx_sent` and `total_received`, clustering in the “short-lived, low-history” regime.

This confirms:

- The model is very good at distinguishing long-lived, high-activity normals from suspicious patterns.  
- The hard part is separating early-stage fraud from early-stage legitimate users, which is exactly where step-up flows and rule-based overrides are useful.

An interactive Plotly figure (Asset 5) lets you toggle between distributions of `time_diff_first_last_received`, `total_tx_sent`, and `total_received` by outcome class.

---

## How This Fits a Real Risk Engine

Putting Parts A and B together:

- **Part A (wallet segments):**
  - Provides coarse-grained behavior clusters.
  - Suggests baseline policies by segment (e.g., which groups are likely high value vs noisy long tail).

- **Part B (transaction scores):**
  - Provides fine-grained, transaction-level risk scores.  
  - Can be used with multiple thresholds:
    - Low score → auto-approve.  
    - Medium score → step-up verification (extra checks).  
    - Very high score and/or high-risk cluster → manual review or block.

In a real exchange or brokerage:

1. A new wallet is first **assigned to a behavior cluster** from Part A, which sets its default risk tier and limits.  
2. As it starts transacting, **each transaction receives a score** from a model like the one in Part B.  
3. A policy engine combines segment + score + simple rules (e.g., “if value > X always review”) to decide how much friction and monitoring to apply.

This project is a small, end-to-end prototype of that workflow on a single chain and a single fraud dataset.

---

## Tech Stack

- **Data Warehouse:** Google BigQuery (Ethereum public dataset)  
- **Labeled Fraud Data:** ETFD Ethereum Transaction Fraud Detection dataset (Kaggle)  
- **Language & Libraries:** Python, pandas, NumPy, scikit‑learn, XGBoost, RandomForest  
- **Explainability:** SHAP (tree explainer)  
- **Visualization:**
  - Matplotlib, Seaborn (exploratory plots)
  - Plotly (interactive ROC/PR curves, confusion matrices, feature importance, error analysis, cost charts)
  - Google Looker Studio (Part A segmentation dashboard)
- **Environment:** Jupyter Notebooks