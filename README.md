# Mule Account EDA – RBIH × IITD Tryst Hackathon (Phase 1)

This repository contains my Phase‑1 Exploratory Data Analysis (EDA) for the RBI Innovation Hub × IIT Delhi Tryst Hackathon mule‑account detection challenge. The goal is not to build a model yet, but to deeply understand the banking dataset, identify behavioural patterns of mule accounts, and design a principled feature‑engineering plan for Phase 2.

## 1. Problem Overview

Banks face a persistent challenge with mule accounts – accounts controlled or rented by fraudsters for moving illicit funds. These accounts often look similar to legitimate customers on the surface, but show characteristic patterns in their transactions, balances, and relationships over time.

In Phase 1, the organisers ask us to:

- Perform structured EDA across multiple linked tables (customers, accounts, products, transactions, labels).
- Identify statistical patterns that distinguish mule accounts (is_mule = 1) from legitimate ones (is_mule = 0).
- Design a concrete, evidence‑backed feature engineering plan for Phase 2 modelling.

The emphasis is on how we think: data understanding, analytical depth, pattern identification, awareness of pitfalls (class imbalance, leakage, noisy labels), and clarity of reasoning.

## 2. Dataset Description

All files are CSVs provided by RBIH for Phase 1 (20% representative sample of the full dataset).

- **customers.csv** – One row per customer: demographics, KYC status, and channel adoption (mobile/internet banking, cards, FASTag, etc.).
- **accounts.csv** – One row per account: product family (savings / K‑family / overdraft), balance metrics, branch, KYC timestamps, rural/urban flag, freeze/unfreeze dates.
- **customer_account_linkage.csv** – Bridge table mapping customers ↔ accounts (a customer can hold multiple accounts).
- **product_details.csv** – Aggregated exposures per customer: loan sums/counts, card exposures, overdrafts, savings and K‑family balances and counts.
- **train_labels.csv** – Training labels at account level: is_mule (1 = mule, 0 = legitimate), plus auxiliary label‑side fields (mule_flag_date, alert_reason, flagged_by_branch).
- **test_accounts.csv** – Unlabelled accounts to be scored in Phase 2.
- **transactions_part_0.csv … transactions_part_5.csv** – ~7.4M transaction records over Jul 2020 – Jun 2025, split across six files with a shared schema: timestamps, channel, amount, debit/credit type, counterparty, MCC, etc.

The official README also lists 12 real‑world money‑laundering behaviour patterns (dormant activation, structuring, rapid pass‑through, fan‑in/fan‑out, geographic anomalies, post‑mobile‑change spikes, round amounts, salary‑cycle exploitation, branch‑level collusion, etc.). These patterns guide the EDA and feature design.

## 3. EDA Objectives

The notebook is structured exactly along the organisers' Phase‑1 expectations:

### Data Understanding
- Validate the relational schema and keys.
- Explore shapes, dtypes, and missingness across all tables.
- Build a consistent account‑centric training view by joining labels, accounts, customers, linkage, and product details.

### Statistical Analysis
- Quantify class imbalance (~1% mule rate).
- Compare distributions by class (mule vs legitimate), not just overall.
- Use robust statistics (medians, non‑parametric tests) and correlations rather than relying on .describe() only.

### Pattern Identification
- Map observed behaviours in the data to the known mule patterns from the README (structuring, dormant activation, fan‑in/fan‑out, post‑mobile‑change spike, round‑amount usage, branch‑level concentration, etc.).

### Feature Engineering Plan (Required)
Propose a concrete list of Phase‑2 features, for each stating:
- What it measures and how it's computed.
- Why it should help separate mule vs legitimate accounts, backed by EDA evidence.
- Which tables/columns it comes from.

### Data Quality & Critical Reasoning
- Inspect missing‑value structure, label noise, and leakage risk.
- Call out fields that are forbidden for modelling (e.g., mule_flag_date, alert_reason, flagged_by_branch).
- Discuss alternative explanations and limitations of the sample.

## 4. Key Analyses and Insights

### 4.1 Data Understanding & Class Imbalance
- Verified the join graph: customers ↔ customer_account_linkage ↔ accounts ↔ transactions, plus product_details at customer level and train_labels/test_accounts at account level.
- Constructed a unified train_view with one row per labelled account, enriched with customer demographics, KYC, branch, and product‐holding information.
- Confirmed strong class imbalance: only about 1% of labelled accounts are tagged as mules (is_mule = 1), making label‑stratified analysis essential.

### 4.2 Account‑ and Customer‑Level Behaviour
- Compared balance metrics (avg_balance, monthly/quarterly/daily averages), tenure (account_age_days, rel_tenure_days) and customer age_years between mules and legitimate accounts.
- Observed systematic distributional differences (e.g., young accounts with high turnover, newer relationships over‑represented among mules), confirmed using Mann–Whitney U tests and point–biserial correlations.

### 4.3 Transaction‑Level Behaviour and Mule Patterns
Using aggregated features from the 7.4M‑row transaction table:

- **Structuring:** Defined structuring_share as the share of an account's transactions in the [₹45k, ₹50k) band. Mule accounts show higher structuring_share than legitimate ones, indicating repeated transfers just below the common threshold.
- **Dormant Activation / New Account High Value:** Combined account_age_days, active_days, txn_velocity_per_day and days_since_last_txn to show clusters of mule accounts that are both young and hyper‑active, consistent with newly activated mules.
- **Fan‑In / Fan‑Out:** Used unique_counterparties, credit_amount, debit_amount, and credit_debit_ratio to capture asymmetries and dense counterparty networks, which are more extreme for mule accounts.
- **Round‑Amount Patterns:** Computed round_amount_share (fraction of transactions that are clean multiples of 1k/5k/10k/50k) and observed higher use of exact round amounts among mules.
- **Temporal Patterns:** Daily time series of transaction counts and amounts, split by class, highlight bursts and differing volatility profiles. Day‑of‑week × hour‑of‑day heatmaps show non‑uniform activity concentration for mules (e.g., off‑peak times and patterns around typical salary cycles).

### 4.4 Branch‑Level and Geographic Patterns
- Estimated mule rates per branch_code and found a few branches with disproportionately high mule concentration among branches with sufficient sample size, consistent with localised risk or possible collusion.
- Compared mule rates across rural_branch vs non‑rural branches to examine geographic skews.

### 4.5 Data Quality and Leakage
- Produced a ranked missing‑rate table to identify columns with heavy nulls and biased availability across segments.
- Verified that label‑side columns (mule_flag_date, alert_reason, flagged_by_branch) are populated almost exclusively for known mules, making them pure leakage for predictive modelling; these are explicitly excluded from the feature plan and treated only as explanatory EDA fields.
- Noted that labels are stated to be noisy and incomplete, so any Phase‑2 model must be robust to imperfect supervision and avoid overfitting idiosyncratic branches or product codes.

## 5. Feature Engineering Plan (Phase 2 Preview)

The notebook ends with a univariate screening table and a curated feature list. Representative examples:

- **structuring_share** (account–transaction) – Fraction of transactions with amounts in [₹45k, ₹50k); captures Structuring pattern and shows significantly higher medians for mules.
- **txn_velocity_per_day** (account–transaction) – txn_count / active_days; highlights Dormant Activation and New Account High Value when combined with account age.
- **unique_counterparties** (account–transaction) – Number of distinct counterparties per account; supports Fan‑In/Fan‑Out and network‑like laundering.
- **credit_debit_ratio** (account–transaction) – Magnitude ratio of total credits to total debits; extreme ratios mark asymmetric pass‑through flows.
- **round_amount_share** (account–transaction) – Fraction of exact‑round transactions; proxies Round Amount Patterns.
- **rel_tenure_days** and **account_age_days** (customer/account) – Capture how recently the relationship or account was opened; important for new/dormant activation patterns.
- **post_mobile_change_flag** (account) – Indicates recent last_mobile_update_date; combined with high velocity, helps detect post‑mobile‑change spikes and possible account takeover.

Each proposed feature in the notebook is accompanied by:
- Precise definition and computation formula.
- Justification grounded in observed class‑wise statistics and plots.
- Source tables/columns and any preprocessing assumptions.