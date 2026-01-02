# Wallet Risk & Behavior Analytics 🛡️
**End-to-end crypto risk engine combining unsupervised learning for user segmentation and supervised modeling for fraud detection.**

[![Live Dashboard](https://img.shields.io/badge/View_Live_Dashboard-Looker_Studio-blue?style=for-the-badge&logo=google-looker-studio)](https://lookerstudio.google.com/s/vub9BwmCKDU)

## 📋 Table of Contents
1. [Project Overview](#-project-overview)
2. [Part A: Behavioral Segmentation (Unsupervised)](#-part-a-behavioral-segmentation)
    - [Data Pipeline](#1-the-data-pipeline)
    - [Feature Engineering](#2-feature-engineering-the-secret-sauce)
    - [Key Insights & Personas](#3-key-insights--personas)
3. [Part B: Fraud Detection (Supervised)](#-part-b-fraud-detection-coming-soon)
4. [Tech Stack](#%EF%B8%8F-tech-stack)

---

## 📌 Project Overview
Crypto exchanges (like Coinbase, Gemini) and fintech apps face a massive "Cold Start" problem: How do you assess the risk of a new wallet address without historical labels?

This project builds a dual-layer risk engine:
1.  **Part A (Segmentation):** Analyzes raw blockchain behavior to cluster users into economic personas, automating risk policies for 94% of users.
2.  **Part B (Fraud Modeling):** Uses supervised learning on labeled data to predict illicit transactions with a focus on precision-recall tradeoffs.

---

## 🔍 Part A: Behavioral Segmentation
**Goal:** Identify hidden user archetypes and anomalous behavior patterns in unlabeled Ethereum data to automate compliance tiers.

### 1. The Data Pipeline
* **Source:** Google BigQuery Public Data (`crypto_ethereum`).
* **Scale:** Extracted and processed **1.2 Million** active wallets from Q4 2025.
* **Engineering:** Developed complex SQL CTEs to join `transactions` (native ETH) with `token_transfers` (ERC-20) to capture a complete view of user activity, overcoming the "Silent Whale" problem where users hold millions in tokens but little ETH.

### 2. Feature Engineering (The "Secret Sauce")
Standard metrics (volume, count) weren't enough to detect risk in a power-law distribution. I engineered specific risk proxies to capture behavioral intent:

| Feature | Logic / Proxy | Risk Signal |
| :--- | :--- | :--- |
| **Mule Score** | `1 / (1 + |log(In / Out)|)` | Detects **"Pass-Through"** wallets (Money Laundering behavior) where funds enter and exit almost immediately, leaving a near-zero balance. |
| **Fan-Out Ratio** | `Unique Counterparties / Total Tx` | High ratio (near 1.0) indicates a **"Burner Wallet"** or Sybil attacker who interacts once per peer and never returns. |
| **Gas Intensity** | `Avg Gas Price` & `Gas Used` | Differentiates **"Simple Users"** (~21k gas, low priority) from **"DeFi Bots"** (High gas price for speed + complex contract interactions). |

### 3. Key Insights & Personas
Using **K-Means Clustering** (optimized to k=4 via the Elbow Method) on Log-Transformed and Robust-Scaled data, I identified 4 distinct economic classes.

#### **🐋 The Institutions (Cluster 2) - 0.2% of Users**
* **Profile:** Massive volume (~4,700 ETH avg) and high-frequency algorithmic trading.
* **Risk Policy:** **White Glove.** Do not auto-block. Assign dedicated Account Manager and trigger manual Enhanced Due Diligence (EDD).

#### **🦄 DeFi Whales (Cluster 1) - 1.5% of Users**
* **Profile:** High net worth individuals interacting with complex Smart Contracts. Highly loyal (>9 days active).
* **Risk Policy:** **Retention & Security.** High risk of Account Takeover (ATO). Enforce hardware 2FA for large withdrawals but waive standard trading fees.

#### **🛒 Active Retail (Cluster 3) - 4% of Users**
* **Profile:** The "Core Customer." Consistent activity, moderate volume, low fraud signals.
* **Risk Policy:** **Growth Track.** Standard instant buy limits ($5k) that auto-scale after 90 days of benign history.

#### **🤖 The Long Tail (Cluster 0) - 94% of Users**
* **Profile:** "Dust" wallets with high Fan-Out ratios. Likely bots, airdrop farmers, or one-time users.
* **Risk Policy:** **Defensive Automation.** Strict KYC required. Block instantly if linked to suspicious IP ranges to reduce operational overhead.

---

## 🚀 Part B: Fraud Detection (Coming Soon)
*Currently in development.*
* **Goal:** Train an XGBoost classifier on the **ETFD (Ethereum Transaction Fraud Detection)** dataset.
* **Metric Focus:** Optimizing **Precision-Recall Tradeoffs** to minimize false positives (blocking legitimate high-value users) while catching real fraud.

---

## 🛠️ Tech Stack
* **Data Warehouse:** Google BigQuery (SQL)
* **Language:** Python (Pandas, Scikit-Learn, NumPy)
* **Visualization:** Matplotlib, Seaborn, Google Looker Studio
* **Environment:** Jupyter Notebook