# Wallet Risk & Behavior Analytics

**End‑to‑end exploration of how to score and segment crypto wallets using on‑chain data, with unsupervised clustering (Part A) and a planned supervised fraud model (Part B).**


[![Live Dashboard](https://img.shields.io/badge/View_Live_Dashboard-Looker_Studio-blue?style=for-the-badge&logo=google-looker-studio)](https://lookerstudio.google.com/s/vub9BwmCKDU)

## Table of Contents
1. [Project Overview](#-project-overview)
2. [Part A: Behavioral Segmentation (Unsupervised)](#-part-a-behavioral-segmentation)
    - [Data Pipeline](#1-the-data-pipeline)
    - [Feature Engineering](#2-feature-engineering-the-secret-sauce)
    - [Key Insights & Personas](#3-key-insights--personas)
3. [Part B: Fraud Detection (Supervised)](#-part-b-fraud-detection-coming-soon)
4. [Tech Stack](#%EF%B8%8F-tech-stack)

---

## Project Overview

Crypto exchanges and fintech apps all face a similar problem: when a new wallet shows up, there is very little labeled history, but the platform still has to make risk decisions (limits, extra checks, monitoring).

This project is my attempt to build a small version of a risk engine around that idea:

1. **Part A – Behavioral Segmentation:** Use raw Ethereum transaction and token transfer data to group wallets into behavioral “types” and think through how a platform might treat each type differently from a risk and product point of view.
2. **Part B – Fraud Modeling:** Train a supervised model on a labeled Ethereum fraud dataset (ETFD) and study precision–recall tradeoffs and false positives in more depth.


---

## Part A: Behavioral Segmentation
**Goal:** Start from unlabeled Ethereum data and see what kinds of wallet behaviors exist, then map those behaviors to simple, practical risk policies a consumer crypto platform could use.

### 1. Data Pipeline
- **Source:** Google BigQuery public Ethereum dataset (`bigquery-public-data.crypto_ethereum`).
- **Scope:** For this first pass, I pulled roughly 1.2M active wallets over a recent ~1‑month window in Q4 2025. This keeps the data recent but still manageable.
- **Tables used:**
  - `transactions` for native ETH transfers.
  - `token_transfers` for ERC‑20 / ERC‑721 token movements.

Instead of downloading raw CSVs, I wrote SQL in BigQuery to aggregate up to the wallet level. The query:
- Unions `from_address` and `to_address` into a single `address` field.
- Computes basic statistics (tx counts, ETH sent/received, unique counterparties).
- Joins in simple token activity stats from `token_transfers` (e.g., how many token transfers, how many distinct token contracts).

This produces one row per wallet with features that describe its behavior over the time window.
The SQL query can be found in `data/query.md`.


### 2. Feature Engineering

Simple counts and sums are a good start, but they don’t fully capture risky or unusual behavior. I engineered a few extra features that are inspired by how fraud and abuse often show up in transaction patterns:

| Feature | Intuition | What it can indicate |
| :--- | :--- | :--- |
| **Flow ratio / “mule score”** | Compare total in vs total out for a wallet. Wallets that mostly pass funds straight through (in ≈ out, low residual balance) behave like intermediaries. | Potential “pass‑through” or money‑mule style behavior. |
| **Fan‑out ratio** | Number of unique counterparties relative to total transactions. | Wallets that send small amounts to many unique addresses (high fan‑out) can look like spam, airdrops, or Sybil activity. |
| **Gas usage and gas price** | Average gas used and gas price for outgoing transactions. | Distinguishes simple transfers from more complex contract interactions and high‑priority / bot‑like activity. |
| **Activity window / “tenure”** | Difference between first and last transaction timestamps in the window. | A high transaction count in a very short window can look bursty or automated. Longer, steady histories tend to be less suspicious. |
| **Token activity** | Number of token transfers and number of distinct tokens moved. | Whether the wallet behaves more like an ETH‑only user, a DeFi/NFT power user, or something in between. |


### 3. Clusters and Wallet “Personas”

I applied K‑Means clustering on log‑scaled and robust‑scaled features and, after trying several values of k, landed on **k = 4** as a reasonable balance between simplicity and expressiveness.

The clusters are not ground truth, but they give a useful mental model for different wallet behaviors:

#### 🐋 Cluster 2 – Large, High‑Volume Wallets (~0.2%)

- High ETH volume and frequent activity.
- Often interact with many counterparties.
- Likely to include institutional, exchange, or sophisticated trading wallets.

**Risk/Policy idea:**  
Avoid automatic blocks. Treat these wallets as “high‑touch”: stronger ongoing monitoring and enhanced due diligence, but careful about adding friction that might push them away.

---

#### 🦄 Cluster 1 – DeFi‑Heavy / Contract Users (~1–2%)

- Lower raw volumes than Cluster 2 but more complex behaviors:
  - Higher gas usage.
  - More token transfers and more distinct token contracts.
- Likely to be DeFi/NFT users or bots interacting with contracts frequently.

**Risk/Policy idea:**  
Focus on account protection (e.g., strong authentication for withdrawals) and clear UX around contract permissions, rather than aggressive blocking.

---

#### 🛒 Cluster 3 – Active Retail Users (a few %)

- Moderate ETH volumes and transaction counts.
- Some token activity but not too many different tokens.
- Fairly regular activity across the time window.

**Risk/Policy idea:**  
These look like typical “core” retail customers. Reasonable buy/fund limits that increase with clean history, and standard KYC/monitoring.

---

#### 🤖 Cluster 0 – Long Tail / One‑off / Noisy Wallets (majority)

- Many wallets with very small balances or a handful of transactions.
- Often higher fan‑out ratios and limited activity span.
- This group likely includes:
  - Bots, airdrop participants, experimental wallets, and one‑time users.

**Risk/Policy idea:**  
Automate as much as possible: strict KYC and simple rules for this group. The goal is to keep operational overhead low while still catching obviously problematic behavior.


### 4. Operational Dashboard (Looker Studio)

To make the clusters and features easier to explore, I built a simple dashboard in **Looker Studio** (link at the top of this README).

- I downsampled the very large “long tail” cluster while keeping all high‑volume clusters to keep the dashboard responsive.
- Plots use log scales (e.g., for volume vs activity) to reflect the power‑law nature of crypto activity.
- For each cluster, the dashboard shows typical ranges for key features and a few example wallets.

The goal is not to build a production tool, but to show how analytics and product/risk teams could explore wallet segments and design different policies and UX treatments for each.


---

## 🚀 Part B: Fraud Detection
* **Goal:** Train an XGBoost classifier on the **ETFD (Ethereum Transaction Fraud Detection)** dataset.
* **Metric Focus:** Optimizing **Precision-Recall Tradeoffs** to minimize false positives (blocking legitimate high-value users) while catching real fraud.

---

## 🛠️ Tech Stack
* **Data Warehouse:** Google BigQuery (SQL)
* **Language:** Python (Pandas, Scikit-Learn, NumPy)
* **Visualization:** Matplotlib, Seaborn, Google Looker Studio
* **Environment:** Jupyter Notebook