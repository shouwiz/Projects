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
- [Trend and seasonality analysis](#trend-and-seasonality-analysis)
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
<img width="566" height="393" alt="image" src="https://github.com/user-attachments/assets/7fca4d18-6afa-420f-b554-437bed7949ff" />
<img width="545" height="393" alt="image" src="https://github.com/user-attachments/assets/569380dd-0e54-4c04-b693-027c39b0d301" /> 

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

## Trend and seasonality analysis

Transaction data spans **July 2024 to June 2026 — 24 months**, which means every calendar month appears exactly twice in the pooled monthly view. This is worth stating up front: it rules out the most common seasonality false positive, where an uneven number of years per month makes the early part of the calendar look artificially strong.

### The June revenue spike

| | Value |
|---|---|
| Baseline monthly order value (pooled, 2 yrs) | ~₹480M |
| June order value (pooled, 2 yrs) | **₹716M** |
| Uplift | **+49%** |
| Share of annual revenue falling in June | ~11.9% (vs 8.0% baseline) |

Every other month sits in a tight ₹445M–₹496M band. February is the weakest, June is the only genuine outlier in the year.

### The spike is driven by order value, not order volume

This is the central finding of the trend analysis. The state-level monthly charts track **quantity sold**, and they show no June effect whatsoever — Delhi's Electronics volume oscillates between roughly 237 and 310 units every month of the two years with no June peak, and Karnataka behaves identically.

Units flat, revenue up 49%. Customers are not shopping more often in June; they are buying **more expensive items**. Any capacity, logistics or inventory planning built off unit forecasts will therefore completely miss the June revenue concentration, and any marketing plan that treats June as a "more traffic" month is solving the wrong problem.

### Category decomposition of the June spike

| Category | Baseline | June | Change |
|---|---|---|---|
| Electronics | ~₹350M | **~₹520M** | +49% (~₹170M of the ₹236M total uplift) |
| Home | ~₹58M | ~₹87M | +50% |
| Fashion | ~₹28M | ~₹42M | +50% |
| Beauty, Books, Grocery, Sports | flat | flat | no seasonal signal |

Electronics dominates the platform year-round at roughly ₹350M per pooled month — several times larger than every other category combined — and it is also the category that produces the seasonal peak. Revenue concentration and seasonal risk sit in the same place.

### Why the business behaves this way

Electronics and Home rising together, in June, on flat unit volume, with high ticket values, is the signature of the **Indian summer cooling-appliance cycle**: air conditioners, coolers and refrigerators. These are few, expensive, planned, weather-triggered purchases — exactly the pattern of "fewer units, far more rupees." June also coincides with the close of financial-year Q1 and the summer sale events most Indian platforms run, both of which pull high-consideration purchases into the month.

**This is a well-supported hypothesis, not a proven cause.** Confirming it requires inspecting `Product_Name` and `Brand` within June Electronics orders to verify that cooling appliances specifically drive the uplift, which this analysis does not yet do. It is the highest-value next piece of work in the repo.

**Crucially, the seasonality and the segmentation describe the same customers.** Cluster 2 — premium occasional, highest AOV in the base at ₹34,290, only 3.56 orders — is behaviourally indistinguishable from a seasonal appliance buyer. The two analyses arrive at the same cohort from different directions, which substantially strengthens both.

### There is no underlying growth

Across all 24 months of state-level data, unit volume is **stationary**. Delhi Electronics runs ~266 units in July 2024 and ~237 in June 2026; Karnataka moves from ~282 to ~291. Two full years, no discernible trend in either direction, in any of the top categories or states examined.

This reframes the entire commercial problem. There is no acquisition momentum to ride. Revenue growth has to come from **basket value and retention within the existing base** — which is precisely what the segmentation is built to target, and why the Cluster 0 recency risk and the Cluster 2 frequency gap are the two levers that matter.

### Caution: "Customer Last Purchase Trend" is a censoring artifact

The last-purchase-by-month chart rises steadily from ~2,750 customers in January to ~9,250 in June, then collapses to ~600 in July. **This is not seasonality and must not be read as such.** The dataset ends 2026-06-30, so recent months mechanically accumulate more "most recent purchases," while July–December can only contain customers who last bought in 2024 or 2025.

Read correctly, it still says something useful: roughly **81% of customers last purchased in January–June**, and the ~7,500 customers whose final order falls in July–December are the genuinely lapsed pool — consistent in size with Cluster 3 plus the tail of Cluster 2.

To get a real churn signal, this should be replotted as last purchase against an **absolute time axis**, with the final 90 days excluded as not-yet-observable.

### Known chart defects to fix

- The "Seasonal Sales Trend by Category" chart plots months in **alphabetical order** (Apr, Aug, Dec, Feb...) because `strftime("%b")` produces strings. Convert to an ordered categorical before plotting.
- Two cells raise `KeyError` and produce no output: one references a non-existent `Order_Month` column, and one references `Month`/`Month_Name` after a re-merge has renamed columns. The monthly order-count trend is missing as a result.
- `sales_pd` is merged against `products_pd` more than once across the notebook, creating `Category_x`/`Category_y` collision risk. The merge should happen exactly once, early.

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

**Seasonal:** build the annual plan around a single June revenue peak in Electronics and Home that arrives on flat unit volume. Stock and merchandise for value, not traffic; begin C2 outreach in April–May while purchase intent is forming, not during the peak when the decision is already made. Recency measured against a 2026-06-30 cut-off is measured from the top of that peak, so the 90-day gap in Cluster 0 means the platform's best customers largely sat out the strongest month of the year.

**Cross-cutting:** rebuild `Customer_Tier` on behavioural cluster membership rather than the current rule, and recompute segments monthly so that C0 members drifting toward C2's recency profile are flagged before they lapse.

---

## Model validation


PCA on the six standardised features shows the structure is genuinely low-dimensional:

| Component | Explained variance | Cumulative |
|---|---|---|
| PC1 | 62.24% | 62.24% |
| PC2 | 24.25% | 86.49% |
| PC3 | 11.95% | **98.44%** |

<img width="700" height="470" alt="image" src="https://github.com/user-attachments/assets/c8dd40b5-2b5e-42a6-8011-17a3cb3be2b5" />


Three components capture 98.4% of total variance, meaning the six behavioural features are largely expressing three underlying dimensions — broadly a value axis, a basket-size axis and a recency axis.
<img width="1189" height="790" alt="image" src="https://github.com/user-attachments/assets/1df72e78-c9ec-4fa6-aa07-58544d27a1de" />\\
Plotted in 2D PCA space (86.5% of variance), the four clusters show clean Voronoi-style boundaries with limited overlap, confirming that the log transformation and scaling did the work required. Clusters 0, 1 and 3 are tightly packed and internally homogeneous; Cluster 2 is visibly more dispersed, consistent with it containing a mix of buyers united by high AOV but varied in category and cadence.

---

## Known limitations

- **Rating is null for 52% of orders.** `Average_Rating` is carried in the aggregation but is biased toward customers who choose to review, and should not be read as a satisfaction measure for the segment as a whole.
- **`Total_Spent` on the customers table does not reconcile with delivered-order monetary totals.** The two fields measure different things (the former appears to include non-delivered orders). All segmentation uses the recomputed delivered-order figure; the customers-table column is used for descriptive EDA only.
- **Segment revenue figures are size × mean.** Because monetary values are right-skewed, cluster means sit above cluster medians, so segment totals are a reasonable estimate of the pool but individual customers within a segment vary widely.
- **K-Means assumes spherical, similarly-sized clusters.** Cluster 2's greater dispersion suggests it may be better modelled by a density- or mixture-based method; worth testing as a follow-up.
- **No forward validation.** Segments describe historic behaviour; they have not yet been tested against subsequent-period purchasing, which is the real test of whether the C0 churn signal holds.
- **Two years is thin for a seasonality claim.** June appears twice. The uplift is large and consistent across categories, but two observations cannot separate a stable annual cycle from two unusual months. A third year, or a year-over-year comparison of the two Junes individually, is needed to confirm.
- **The June cause is inferred, not measured.** No product-level check has been run on June Electronics orders to confirm cooling appliances drive the uplift.
- **Recency is measured from the seasonal peak.** With a 2026-06-30 cut-off, every customer's `Recency` is anchored to the end of the strongest month of the year, which compresses recency for June buyers and inflates it for everyone else. Cluster boundaries are sensitive to this choice of analysis date.
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
