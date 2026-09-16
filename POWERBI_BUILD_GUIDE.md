# Power BI Build Guide — Customer Analytics

One report, two pages, a shared left navigation rail. Page 1 answers *who the customers are*, Page 2 answers *what the segments are worth*.

Everything below is buildable with native Power BI visuals — nothing here needs a custom visual from AppSource.

---

## 1. Export the data first

Power BI cannot see your K-Means results, because `K4_Cluster` only exists in a pandas dataframe inside the notebook. **This is the step people skip, and it's the reason the segmentation page can't be built.** Add this cell at the end of the notebook:

```python
# --- Export the segment assignment table for Power BI -------------------
segment_names = {
    0: "Loyal High-Value",
    1: "Low-Value Dormant",
    2: "Premium Occasional",
    3: "Lapsed",
}
segment_order = {0: 1, 2: 2, 1: 3, 3: 4}   # sort by revenue contribution

dim_segment = customer_pd[[
    "Customer_ID", "K4_Cluster",
    "Recency", "Frequency", "Monetary",
    "Total_Units", "Unique_Products", "Average_Order_Value",
]].copy()

dim_segment["Segment_Name"]  = dim_segment["K4_Cluster"].map(segment_names)
dim_segment["Segment_Order"] = dim_segment["K4_Cluster"].map(segment_order)

(spark.createDataFrame(dim_segment)
      .write.mode("overwrite")
      .saveAsTable("workspace.default.customer_segments"))
```

Then connect Power BI Desktop with **Get Data → Azure Databricks**, supplying your server hostname, HTTP path and a personal access token. Import mode, not DirectQuery — the tables are small and import gives you far better visual performance.

Load four tables: `customers`, `products`, `sales`, `customer_segments`.

---

## 2. Data model

A star schema. In **Model view**, delete any relationships Power BI auto-detects incorrectly and build these:

| From | To | Cardinality | Direction |
|---|---|---|---|
| `sales[Customer_ID]` | `customers[Customer_ID]` | Many → 1 | Single |
| `sales[Product_ID]` | `products[Product_ID]` | Many → 1 | Single |
| `customer_segments[Customer_ID]` | `customers[Customer_ID]` | 1 → 1 | **Both** |
| `sales[Order_Date]` | `DimDate[Date]` | Many → 1 | Single |

`customer_segments` to `customers` is bidirectional so that a segment slicer filters the demographics page too. That's the only bidirectional relationship in the model — keep it that way, or you'll get ambiguous filter paths.

### Date table

Mark this as the official date table (**Table tools → Mark as date table**), or time intelligence will silently misbehave:

```dax
DimDate =
VAR MinDate = MIN ( sales[Order_Date] )
VAR MaxDate = MAX ( sales[Order_Date] )
RETURN
ADDCOLUMNS (
    CALENDAR ( MinDate, MaxDate ),
    "Year",        YEAR ( [Date] ),
    "Month No",    MONTH ( [Date] ),
    "Month",       FORMAT ( [Date], "MMM" ),
    "Year Month",  FORMAT ( [Date], "YYYY-MM" ),
    "Quarter",     "Q" & QUARTER ( [Date] )
)
```

Sort `Month` by `Month No` (**Column tools → Sort by column**), otherwise your months plot alphabetically — the exact bug already present in the notebook's category chart.

### Segment sort order

Select `customer_segments[Segment_Name]` → **Sort by column** → `Segment_Order`. Every visual then shows segments ordered by revenue contribution rather than alphabetically.

---

## 3. Measures

Create an empty table called `_Measures` (**Enter data**, no rows) and put everything in it, so measures don't clutter your fact table.

### Shared

```dax
Total Customers = DISTINCTCOUNT ( customers[Customer_ID] )

Delivered Sales =
CALCULATE (
    SUM ( sales[Total_Amount] ),
    sales[Order_Status] = "Delivered"
)

Total Orders =
CALCULATE (
    DISTINCTCOUNT ( sales[Order_ID] ),
    sales[Order_Status] = "Delivered"
)

Products Sold =
CALCULATE (
    SUM ( sales[Quantity] ),
    sales[Order_Status] = "Delivered"
)

Orders per Customer =
DIVIDE ( [Total Orders], [Total Customers] )

Analysis Date = MAX ( sales[Order_Date] )
```

### Page 1 — demographics

```dax
Median Age = MEDIAN ( customers[Age] )

Active Customers =
CALCULATE (
    DISTINCTCOUNT ( sales[Customer_ID] ),
    sales[Order_Status] = "Delivered"
)

% of Customers =
DIVIDE (
    [Total Customers],
    CALCULATE ( [Total Customers], ALLSELECTED ( customers ) )
)

Products Sold Rank by State =
RANKX (
    ALLSELECTED ( customers[State] ),
    [Products Sold],
    ,
    DESC
)
```

### Page 2 — segmentation

Recency, frequency and monetary are per-customer attributes, so averaging them needs `AVERAGE` over the dimension, not over the fact table:

```dax
Segmented Customers = DISTINCTCOUNT ( customer_segments[Customer_ID] )

Avg Recency = AVERAGE ( customer_segments[Recency] )

Avg Frequency = AVERAGE ( customer_segments[Frequency] )

Avg Monetary = AVERAGE ( customer_segments[Monetary] )

Segment Sales = SUM ( customer_segments[Monetary] )

Avg Order Value = AVERAGE ( customer_segments[Average_Order_Value] )

% of Revenue =
DIVIDE (
    [Segment Sales],
    CALCULATE ( [Segment Sales], ALL ( customer_segments ) )
)

% of Base =
DIVIDE (
    [Segmented Customers],
    CALCULATE ( [Segmented Customers], ALL ( customer_segments ) )
)
```

Two worth adding, because they turn the dashboard from descriptive into actionable:

```dax
Revenue at Risk =
CALCULATE (
    [Segment Sales],
    customer_segments[Recency] > 90
)

Revenue Concentration =
VAR TopTwo =
    CALCULATE (
        [Segment Sales],
        customer_segments[Segment_Name] IN { "Loyal High-Value", "Premium Occasional" }
    )
RETURN
    DIVIDE ( TopTwo, CALCULATE ( [Segment Sales], ALL ( customer_segments ) ) )
```

### Formatting

Set these once on each measure (**Measure tools**) rather than per visual:

- `Delivered Sales`, `Segment Sales`, `Revenue at Risk` → Currency, ₹ English (India), 2 decimals, display units **Billions**
- `Avg Monetary`, `Avg Order Value` → Currency ₹, 0 decimals, display units Thousands
- `% of Revenue`, `% of Base`, `Revenue Concentration` → Percentage, 1 decimal
- `Avg Frequency` → 2 decimals; `Avg Recency` → 0 decimals

---

## 4. The India map

Power BI's built-in maps are the only part of this that needs care, because they geocode from text and Indian state names are ambiguous.

**Add a geo-typed column.** In `customers`, select `State` → **Column tools → Data category → State or Province**. Without this, Power BI will scatter your bubbles across the wrong continents.

**Clean the state names before they reach Power BI.** Your data contains abbreviations like `UP` which will not geocode. Fix it in the notebook, not in Power BI:

```python
state_fix = {
    "UP": "Uttar Pradesh", "MP": "Madhya Pradesh",
    "AP": "Andhra Pradesh", "TN": "Tamil Nadu",
    "WB": "West Bengal",   "HR": "Haryana",
    "PB": "Punjab",        "RJ": "Rajasthan",
    "KA": "Karnataka",     "MH": "Maharashtra",
    "TS": "Telangana",     "GJ": "Gujarat",
    "DL": "Delhi",
}
customers_pd["State"] = customers_pd["State"].replace(state_fix)
```

Add a country column too — `customers_pd["Country"] = "India"` — and put it above `State` in the map's Location hierarchy. This stops Power BI resolving "Punjab" to Pakistan, which it will otherwise do.

**Which visual to use.** Use the **Map** (bubble) visual rather than Filled map:

| Field well | Put this in |
|---|---|
| Location | `customers[State]` |
| Bubble size | `[Products Sold]` |
| Tooltips | `[Total Customers]`, `[Delivered Sales]`, `[Orders per Customer]` |

Bubble area encodes magnitude honestly; a filled choropleth shades Rajasthan's huge landmass the same as tiny Delhi and misleads the reader about where volume actually is. Set **Map styles → Theme → Grayscale** so the basemap recedes.

If your tenant has map visuals disabled by admin policy (common in Indian enterprises), the fallback is a bar chart of `State` by `[Products Sold]` sorted descending. It's less striking but reads more precisely, and given your state volumes are nearly uniform it may genuinely be the better visual.

---

## 5. Page layout

Both pages: **16:9, 1280 × 720**, page background `#EDF0F4` at 0% transparency. Gutters of 12px between every visual, 18px page margin.

### Shared navigation rail — 186px wide, both pages

1. Insert → Shapes → Rectangle, 186 × 684, fill `#FFFFFF`, border `#DFE4EC`, rounded corners 8.
2. Two **Buttons** (Insert → Buttons → Blank), each 162 × 34, text left-aligned 10pt.
3. For each: **Action → Type: Page navigation → Destination**.
4. Style the current page's button with a `#2C5F8A` left border and 12%-tint fill so the active page is obvious.
5. Select both shapes and buttons → right-click → **Group**, then copy the group to the second page so the rail sits in identical pixel positions. Misaligned rails between pages produce a visible jump when navigating and are the most common polish failure in multi-page reports.

### Page 1 — Customer demographics

```
┌──────┬──────────────────────────────────────────────────────────┐
│      │  Who the customers are                    As of 30 Jun   │
│ RAIL ├────────┬────────┬────────┬────────┬────────┬─────────────┤
│      │ Total  │Products│ Total  │ Median │ Orders │   5 cards   │
│ ·Dem │ custs  │  sold  │revenue │  age   │ /cust  │   h = 84    │
│ ·Seg ├────────┴────────┴────────┼────────┴────────┴─────────────┤
│      │                          │  Gender — donut               │
│ slic │   India bubble map       ├───────────────────────────────┤
│      │   Products sold          │  Customer tier — bar          │
│      │   by state               ├───────────────────────────────┤
│      │                          │  Age group — column           │
└──────┴──────────────────────────┴───────────────────────────────┘
```

| Visual | Type | Fields |
|---|---|---|
| KPI ×5 | Card | `[Total Customers]`, `[Products Sold]`, `[Delivered Sales]`, `[Median Age]`, `[Orders per Customer]` |
| Map | Map (bubble) | Location `State`, Size `[Products Sold]` |
| Gender | Donut | Legend `Gender`, Values `[Total Customers]` |
| Tier | Clustered bar | Y `Customer_Tier`, X `[Total Customers]` |
| Age group | Clustered column | X `Age_Group`, Y `[Total Customers]` |

Sort `Age_Group` by a numeric sort column or it renders as 18–25, 26–35, 36–45, 46–55, 56–65, **65+** out of order — `65+` sorts before `18-25` alphabetically in some collations.

### Page 2 — Customer segmentation

```
┌──────┬──────────────────────────────────────────────────────────┐
│      │  What the segments are worth           K=4 · 30 Jun 2026 │
│ RAIL ├────────┬────────┬────────┬────────┬────────┬─────────────┤
│      │ Custs  │ Total  │  Avg   │  Avg   │  Avg   │   5 cards   │
│ ·Dem │segmntd │ sales  │recency │ freq   │monetary│             │
│ ·Seg ├────────┴───┬────┴───┬────┴───┬────┴────────┴─────────────┤
│      │  Loyal     │Premium │Low-val │ Lapsed │   4 segment      │
│ slic │  high-val  │occasnl │dormant │        │   cards  h=118   │
│      ├────────────┴────────┼────────┴──────────────────────────┤
│      │  Segment detail     │  Recency │ Frequency │ Monetary   │
│      │  table w/ data bars │  three small multiples            │
└──────┴─────────────────────┴───────────────────────────────────┘
```

| Visual | Type | Fields |
|---|---|---|
| KPI ×5 | Card | `[Segmented Customers]`, `[Segment Sales]`, `[Avg Recency]`, `[Avg Frequency]`, `[Avg Monetary]` |
| Segment cards ×4 | Multi-row card, filtered to one segment each | `[Segmented Customers]`, `[Segment Sales]`, `[Avg Recency]`, `[Avg Frequency]` |
| Segment detail | Table | `Segment_Name`, `[Segmented Customers]`, `[Segment Sales]`, `[% of Revenue]`, `[Avg Order Value]` |
| Recency / Frequency / Monetary | Three clustered bar charts | Y `Segment_Name`, X the matching measure |

On the detail table, apply **Conditional formatting → Data bars** to `% of Revenue`, and set the bar colour to `#2C5F8A`. Turn the numeric value on so the bar supplements the figure rather than replacing it.

Colour each segment consistently across all visuals — teal `#14505E` for Loyal High-Value, `#4A8E9C` for Premium Occasional, amber `#C08A2E` for Low-Value Dormant, muted red `#9B4A4A` for Lapsed. Set this once via the theme's `dataColors` order and it propagates. Consistent segment colour across nine visuals is what makes a dashboard feel designed rather than assembled.

---

## 6. Apply the theme

**View → Themes → Browse for themes** → select `customer-analytics-theme.json`.

It sets the page background, card borders and radius, removes shadows and gridlines, turns off axis titles, and fixes the segment colour order. Apply it *before* you start formatting individual visuals, otherwise your manual formatting overrides it and you'll be undoing work.

---

## 7. Interactions and polish

- **Cross-filtering.** Select the map → **Format → Edit interactions** → set the KPI cards to *Filter*, so clicking a state updates every headline number. Set the segment cards on page 2 to *None*, since each is already filtered to a fixed segment and cross-filtering would blank them.
- **Tooltips.** Build a hidden page-type tooltip showing recency, frequency, monetary and order count for whatever is hovered. Page size **Tooltip**, and set **Page information → Allow use as tooltip** on.
- **Sync slicers.** **View → Sync slicers**, tick both pages for the date and state slicers so a filter set on one page carries to the other. Leave the segment slicer unsynced if you want the demographics page to always show the full base.
- **Alignment.** Select multiple visuals → **Format → Align → Distribute horizontally**. Pixel-aligned edges are most of what "pristine" actually means in practice.
- **Accessibility.** Set tab order under **View → Selection → Tab order**, putting KPI cards first and decorative shapes last (or marking them hidden from tab order entirely).

---

## 8. What to check before you share it

- Does `[Delivered Sales]` total ₹4.74B? If it reads ~₹6.0B you've lost the `Order_Status = "Delivered"` filter somewhere.
- Do the four segments sum to 39,726 customers, not 40,000? The 274-customer gap is real — those customers never had a delivered order and are correctly absent from `customer_segments`.
- Do all months sort chronologically in every visual?
- Does every state bubble land inside India?

---

## Caveats worth putting on the report itself

Add a small text box in the footer of page 2:

> Segments reflect delivered-order behaviour to 30 Jun 2026. Recency is measured from the end of June, the strongest sales month of the year, which compresses recency for seasonal buyers. Segment revenue is the sum of customer lifetime value, not a period figure — it will not tie to a date-filtered sales total.

That last point catches people out constantly: `[Segment Sales]` sums a pre-computed lifetime value per customer, so it **will not respond correctly to a date slicer**. If you need date-responsive segment revenue, compute it from the `sales` fact instead:

```dax
Segment Sales (date aware) =
CALCULATE (
    SUM ( sales[Total_Amount] ),
    sales[Order_Status] = "Delivered"
)
```

with `customer_segments[Segment_Name]` on the axis — the bidirectional relationship makes this work. Use this version on any visual that sits behind a date slicer, and the lifetime version only on the unfiltered segment summary.
