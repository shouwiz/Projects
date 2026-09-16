# Customer Segmentation for an Indian E-Commerce Platform

K-Means segmentation of 40,000 customers across 250,000 orders, built on PySpark and scikit-learn, with PCA-based validation and a revenue-weighted read of each segment.

The headline finding: **42% of customers generate 66% of revenue, and that group has not ordered in 90 days on average.**

---

## Contents

- [Business problem](#business-problem)
- [Data](#data)
- [Approach](#approach)
- [Choosing K](#choosing-k)
- [Segment profiles](#segment-profiles)
- [Business insights](#business-insights)
- [Recommended actions](#recommended-actions)
- [Model validation](#model-validation)
- [Known limitations](#known-limitations)
- [Running this yourself](#running-this-yourself)

---

## Business problem

The platform treats its customer base as one undifferentiated audience. Marketing spend, retention campaigns and discounting are applied uniformly, which means budget flows to customers who will never be profitable while the customers who fund the business receive no special attention.

This project answers three questions:

1. How many economically distinct customer groups actually exist in the transaction data?
2. How much revenue does each group represent?
3. Which group should receive the next marketing rupee?

---

## Data

Three Delta tables in Databricks, read via PySpark:

| Table | Rows | Columns | Key fields |
|---|---|---|---|
| `customers` | 40,000 | 15 | `Customer_ID`, `Age`, `Gender`, `State`, `Customer_Tier`, `Total_Orders`, `Total_Spent` |
| `products` | 2,000 | 12 | `Product_ID`, `Category`, `Brand`, `Selling_Price`, `Discount_Percent` |
| `sales` | 250,000 | 21 | `Order_ID`, `Customer_ID`, `Product_ID`, `Order_Date`, `Quantity`, `Total_Amount`, `Order_Status`, `Rating` |

Analysis reference date: **2026-06-30** (the most recent `Order_Date` in the sales table).

**Completeness.** `customers` and `products` have no nulls. In `sales`, two columns are heavily sparse by design rather than by error: `Coupon_Code` is null for 199,815 rows (80% of orders used no coupon) and `Rating` / `Review_Text` are null for 129,970 rows (52% of orders went unreviewed). Both are handled explicitly rather than imputed.

---

## Approach

### Feature engineering

Segmentation features are aggregated from **delivered orders only**, so that cancellations and returns do not inflate a customer's apparent value:

| Feature | Definition |
|---|---|
| `Recency` | Days between the customer's last order and 2026-06-30 |
| `Frequency` | Distinct delivered orders |
| `Monetary` | Sum of `Total_Amount` |
| `Total_Units` | Sum of `Quantity` |
| `Unique_Products` | Distinct products purchased |
| `Average_Order_Value` | `Monetary / Frequency` |

### Handling skew

Raw features are right-skewed, which distorts Euclidean distance and lets a handful of whales dominate the centroids:

```
Recency               1.49
Frequency             0.51
Monetary              1.49
Total_Units           0.62
Unique_Products       0.51
Average_Order_Value   2.21
```

All monetary and count features are `log1p`-transformed; `Recency` is left untransformed since its skew is moderate and its raw scale is directly interpretable. All six features are then standardised with `StandardScaler`, which K-Means requires because it minimises unweighted squared distance.

---

## Choosing K

Two criteria were evaluated across K = 2 to 11: within-cluster sum of squares (elbow) and silhouette score.

**K=2 scored highest on silhouette but was rejected.** It splits the base into "high spenders" and "low spenders" — mathematically clean, commercially useless, since it reproduces what a single sort on `Total_Spent` already tells you.

**K=4 was selected.** Its silhouette score is only marginally below K=3, and the fourth cluster is not noise: it isolates genuinely lapsed customers from merely infrequent ones, which is the distinction that decides whether a retention campaign is worth running. Cluster quality was traded for cluster actionability, deliberately.

---

## Segment profiles

| | **C0 — Loyal High-Value** | **C2 — Premium Occasional** | **C1 — Low-Value Dormant** | **C3 — Lapsed** |
|---|---|---|---|---|
| Customers | 16,469 (41.5%) | 11,326 (28.5%) | 7,889 (19.9%) | 4,042 (10.2%) |
| Avg recency (days) | 90 | 185 | 113 | 336 |
| Avg frequency | 6.94 | 3.56 | 4.81 | 1.88 |
| Avg monetary | ₹189,793 | ₹116,614 | ₹28,603 | ₹17,178 |
| Avg order value | ₹28,119 | **₹34,290** | ₹6,016 | ₹10,992 |
| Avg units | 8.82 | 4.41 | 5.81 | 2.18 |
| Unique products | 6.92 | 3.55 | 4.79 | 1.88 |

### Revenue contribution

Segment revenue = cluster size × average monetary value.

| Segment | Revenue | Share of revenue | Share of customers |
|---|---|---|---|
| C0 — Loyal High-Value | ₹3.13B | **65.9%** | 41.5% |
| C2 — Premium Occasional | ₹1.32B | **27.9%** | 28.5% |
| C1 — Low-Value Dormant | ₹0.23B | 4.8% | 19.9% |
| C3 — Lapsed | ₹0.07B | 1.5% | 10.2% |
| **Total** | **₹4.74B** | 100% | 100% |

C0 and C2 together are 70% of the customer base and **93.8% of revenue**. C1 and C3 are 30% of the base and 6.2% of revenue.

---

## Business insights

### 1. The largest revenue risk is inside the best segment, not the worst one

Cluster 0 customers average 6.94 orders and ₹189,793 in lifetime value, but their average recency is **90 days**. These are not new customers still finding their footing — they are established buyers who have quietly gone quiet. ₹3.13 billion in historic revenue sits with a cohort one quarter from the conventional churn threshold.

The asymmetry matters for budget allocation: a 5-point retention improvement in C0 is worth roughly ₹156M, more than the *entire* combined revenue of Clusters 1 and 3 (₹296M) and far cheaper to achieve than reactivating them.

### 2. Cluster 2 has a frequency problem, not a value problem

C2 records the **highest average order value in the entire base — ₹34,290**, above even the loyal segment. They are not weak customers; they are seasonal, high-ticket buyers (the category-level seasonality in the sales trend supports appliance- and electronics-type purchase cycles) who simply do not have a reason to return between major purchases.

Discounting this segment would be a direct margin transfer to people already demonstrating high willingness to pay. Moving C2 from 3.56 to 4.5 orders per customer at unchanged AOV would add approximately **₹365M in revenue** — achieved through accessories, service plans and replenishment timing rather than price cuts.

### 3. Cluster 1 is the segment to stop spending on

C1 orders reasonably often (4.81 times) so the barrier is not platform friction or trust — they know how to buy. Their AOV of ₹6,016 is roughly one-fifth of every other segment's, and at 113 days their recency is no better than the premium segment's. This is a structurally low-margin cohort of discount-item buyers. Retention investment here has a low ceiling regardless of execution quality.

### 4. Cluster 3 is effectively already gone

At 336 days recency and 1.88 orders, C3 are near-single-purchase customers who have been absent for almost a year. At 1.5% of revenue, the correct treatment is exclusion from paid campaigns and retention to a low-cost email list only. Their main analytical value is as a control group and as a source of first-purchase churn signal.

### 5. Tier labels do not match behaviour

The `Customer_Tier` field concentrates the majority of customers in Platinum, while behavioural clustering shows only 41.5% of customers exhibit genuine high-value patterns and 30% contribute almost nothing. Whatever rule assigns tiers is not tracking economic value, which means any tier-driven perk or discount programme is mispricing a large portion of the base.

### 6. Spending is far more unequal than ordering

Order counts are near-symmetric (skew 0.46, mean 5.0, max 17) while spending is strongly right-skewed (skew 1.49, median ₹90,060 against a mean of ₹118,539, max ₹890,132). Customers differ far more in *what* they buy than in *how often*. Basket composition, not visit frequency, is the primary driver of customer value — which argues for merchandising and recommendation work ahead of pure re-engagement volume.

---

## Recommended actions

| Priority | Segment | Action | Rationale |
|---|---|---|---|
| **1** | C0 (₹3.13B) | Win-back before day 120: early access, loyalty tier upgrade, personal outreach for the top decile | Highest absolute revenue at risk; these customers are still warm |
| **2** | C2 (₹1.32B) | Cross-sell accessories and service plans; time outreach to category purchase cycles. **No discounting** | Highest AOV in the base — frequency is the gap, price is not |
| **3** | C1 (₹0.23B) | Low-cost automated email only; test bundling to raise AOV before any further spend | Reasonable engagement but structurally small baskets |
| **4** | C3 (₹0.07B) | Exclude from paid acquisition and retention; retain as control group | Nearly a year dormant, 1.5% of revenue |

**Cross-cutting:** rebuild `Customer_Tier` on behavioural cluster membership rather than the current rule, and recompute segments monthly so that C0 members drifting toward C2's recency profile are flagged before they lapse.

---

## Model validation

PCA on the six standardised features shows the structure is genuinely low-dimensional:

| Component | Explained variance | Cumulative |
|---|---|---|
| PC1 | 62.24% | 62.24% |
| PC2 | 24.25% | 86.49% |
| PC3 | 11.95% | **98.44%** |

Three components capture 98.4% of total variance, meaning the six behavioural features are largely expressing three underlying dimensions — broadly a value axis, a basket-size axis and a recency axis.

Plotted in 2D PCA space (86.5% of variance), the four clusters show clean Voronoi-style boundaries with limited overlap, confirming that the log transformation and scaling did the work required. Clusters 0, 1 and 3 are tightly packed and internally homogeneous; Cluster 2 is visibly more dispersed, consistent with it containing a mix of buyers united by high AOV but varied in category and cadence.

---

## Known limitations

- **Rating is null for 52% of orders.** `Average_Rating` is carried in the aggregation but is biased toward customers who choose to review, and should not be read as a satisfaction measure for the segment as a whole.
- **`Total_Spent` on the customers table does not reconcile with delivered-order monetary totals.** The two fields measure different things (the former appears to include non-delivered orders). All segmentation uses the recomputed delivered-order figure; the customers-table column is used for descriptive EDA only.
- **Segment revenue figures are size × mean.** Because monetary values are right-skewed, cluster means sit above cluster medians, so segment totals are a reasonable estimate of the pool but individual customers within a segment vary widely.
- **K-Means assumes spherical, similarly-sized clusters.** Cluster 2's greater dispersion suggests it may be better modelled by a density- or mixture-based method; worth testing as a follow-up.
- **No forward validation.** Segments describe historic behaviour; they have not yet been tested against subsequent-period purchasing, which is the real test of whether the C0 churn signal holds.
- **Single snapshot.** Customers are assigned to one cluster at one point in time. Migration between segments is where most of the operational value lives and is not yet modelled.

---

## Running this yourself

**Environment:** Databricks (PySpark 3.x) with Python 3.12.

```bash
pip install pandas numpy matplotlib seaborn scikit-learn
```

The notebook expects three tables at `workspace.default.{customers, products, sales}`. To run outside Databricks, replace the `spark.table(...)` calls at the top with your own loaders and drop the `display()` calls in favour of `print()`.

```
python_customer_seg.ipynb     # EDA → feature engineering → K-Means → PCA validation
```

**Pipeline order:** load and profile → EDA on demographics, tiers and seasonality → delivered-order feature aggregation in Spark → skew correction and scaling → elbow and silhouette sweep → K=4 fit → cluster profiling → PCA validation.

---

## Tech stack

PySpark · pandas · NumPy · scikit-learn (KMeans, StandardScaler, PCA, silhouette_score) · Matplotlib · Seaborn · Databricks
