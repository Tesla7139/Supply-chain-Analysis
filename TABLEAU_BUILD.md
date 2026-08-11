# Tableau Build Spec

Rebuild instructions for the Supply Chain Analytics dashboard, against the
**corrected** data in `Cleaned_Data/`. Follow top to bottom.

Tool: **Tableau Public Desktop** (free download). Publishing to Tableau Public
makes the workbook public — that is expected here.

---

## 1. Connect the data

Connect to **Text file** → `Cleaned_Data/orders_and_shipments_clean.csv`, then
add the other two to the canvas.

| File | Rows |
|---|---:|
| `orders_and_shipments_clean.csv` | 30,871 |
| `inventory_clean.csv` | 4,200 |
| `fulfillment_clean.csv` | 118 |

### Use Relationships, not Joins

Drag the second and third files onto the canvas so Tableau draws a **noodle**
(relationship), *not* a Venn-diagram join.

| From | To | Match on |
|---|---|---|
| orders | inventory | `Product Name` = `Product Name` **AND** `Order YearMonth` = `Year Month` |
| orders | fulfillment | `Product Name` = `Product Name` |

**This matters.** An inner join on `Product Name` alone fans each order line out
against every monthly inventory row for that product, multiplying `Gross Sales`
and `Profit` several times over. Relationships keep each table at its own grain
and aggregate correctly. If your profit totals look implausibly large, this is
why.

---

## 2. Calculated fields

Create these before building any sheet.

**Storage Cost**
```
[Warehouse Inventory] * [Inventory Cost Per Unit]
```

**Profit Margin %**
```
SUM([Profit]) / SUM([Gross Sales]) * 100
```

**Shipment Delay (days)** — actual processing time minus what was scheduled
```
[Order Processing Time] - [Shipment Days - Scheduled]
```

**Valid Shipment Delay** — the one that matters
```
IF [Valid Processing Time] THEN [Shipment Delay (days)] END
```

**Is Delayed**
```
IF [Valid Processing Time] THEN [Shipment Delay (days)] > 0 END
```

**Delayed Order Rate %**
```
SUM(IF [Is Delayed] THEN 1 ELSE 0 END) / SUM(IF NOT ISNULL([Is Delayed]) THEN 1 ELSE 0 END) * 100
```

**Inventory to Sales Delta**
```
SUM([Warehouse Inventory]) - SUM([Order Quantity])
```

**Stock Status**
```
IF [Inventory to Sales Delta] > 0 THEN 'Overstock' ELSE 'Understock' END
```

### The filter you must not skip

`Valid Processing Time` is a boolean flagging the **25,882 usable rows** out of
30,871. The other **4,989 (16.2%)** carry corrupt shipment dates — shipments
recorded up to **975 days before their own order**.

> On **every** sheet that measures shipment timing or delay, drag
> `Valid Processing Time` to Filters and keep **True** only.
>
> Do **not** apply it to profit, sales or inventory sheets — those rows have
> valid financial data, and excluding them would understate revenue.

`Valid Shipment Delay` and `Is Delayed` already return NULL for bad rows, so
they are safe unfiltered; the explicit filter is belt-and-braces and makes the
intent visible to anyone reading the workbook.

---

## 3. Dashboard 1 — Business Performance

| Sheet | Marks | Columns / Rows | Detail |
|---|---|---|---|
| Most Profitable Goods | Circle (packed bubbles) | `Product Name` | Size + Colour: `SUM(Profit)`. Filter: Top N by `SUM(Profit)` |
| Profit by Product Department | Area | `MONTH(Order Datetime)` / `SUM(Profit)` | Colour: `Product Department` |
| Total Profit | Line | `MONTH(Order Datetime)` / `SUM(Profit)` | Annotate the peak month |
| Highest Inventory Storage Cost | Horizontal bar | `SUM(Storage Cost)` / `Product Name` | Sort descending, Top N |
| Goods with Highest Profit Margin | Circle | `Product Name` | Colour: `Profit Margin %`. **Filter `SUM(Gross Sales)` ≥ 10000** |
| Storage Cost by Department | Area | `MONTH(Order Datetime)` / `SUM(Storage Cost)` | Colour: `Product Department` |

**Top N parameter:** create an integer parameter `Top N` (range 1–20, default 6)
and drive the Top N filters from it, so one control resizes every bubble/bar
sheet at once.

**Why the margin filter:** without a sales floor, a product with ₹100 of sales
and ₹90 profit shows a 90% margin and dominates the view. The floor keeps the
chart on products that move real volume.

---

## 4. Dashboard 2 — Inventory Management

| Sheet | Marks | Setup |
|---|---|---|
| Warehouse Inventory Over Time | Line | `MONTH(Order Datetime)` / `SUM(Warehouse Inventory)`, colour by `Product Department` |
| Supply vs Demand by Department | Side-by-side bar | `Product Department` / measures `SUM(Warehouse Inventory)` and `SUM(Order Quantity)` |
| Most Overstocked Products | Bar | `Product Name` / `Inventory to Sales Delta`, filter `Stock Status` = Overstock, sort desc |
| Most Understocked Products | Bar | `Product Name` / `Inventory to Sales Delta`, filter `Stock Status` = Understock, sort asc |
| Storage Cost by Category | Treemap | Size `SUM(Storage Cost)`, colour `Product Department`, detail `Product Category` |

Colour the over/understock bars divergently (positive one hue, negative another)
so the direction reads without consulting the legend.

---

## 5. Dashboard 3 — Shipment Investigation

**Apply the `Valid Processing Time` = True filter to every sheet here.**

| Sheet | Marks | Setup |
|---|---|---|
| % of Delayed Orders | BAN (big number) | `Delayed Order Rate %`, formatted to 1 dp |
| Delay Evolution | Line | `MONTH(Order Datetime)` / `AVG(Valid Shipment Delay)`, reference line at 0 |
| Delay by Shipment Mode | Bar | `Shipment Mode` / `AVG(Valid Shipment Delay)` |
| Delay by Customer Region | Map or bar | `Customer Country` / `AVG(Valid Shipment Delay)`, diverging colour centred on 0 |
| Most Delayed Products | Bar | `Product Name` / `AVG(Valid Shipment Delay)`, Top N, filter to products with ≥ 30 orders |

The ≥ 30 order floor prevents a product with two late shipments from topping the
"most delayed" list.

---

## 6. Dashboard 4 — Shipment Delay Details

| Sheet | Marks | Setup |
|---|---|---|
| Delay Distribution | Histogram | Bin `Valid Shipment Delay` at width 1 |
| Scheduled vs Actual | Scatter | `AVG(Shipment Days - Scheduled)` / `AVG(Order Processing Time)`, detail `Product Category`, 45° reference line |
| Delay Heatmap | Square | `Shipment Mode` / `Product Department`, colour `AVG(Valid Shipment Delay)` |
| Data Quality Note | Text | See below |

Add a text tile stating plainly:

> Shipment metrics exclude 4,989 of 30,871 order lines (16.2%) whose shipment
> dates precede their order date or exceed 30 days. Financial metrics use all
> rows.

Putting the exclusion on the dashboard rather than hiding it is the point. It is
the difference between a dashboard that can be trusted and one that cannot.

---

## 7. Dashboard 5 — Order Fulfillment Days

| Sheet | Marks | Setup |
|---|---|---|
| Fulfillment by Category | Bar | `Product Category` / `AVG(Warehouse Order Fulfillment (days))`, sorted desc |
| Fulfillment vs Delay | Scatter | `AVG(Warehouse Order Fulfillment (days))` / `AVG(Valid Shipment Delay)`, detail `Product Name`, trend line |
| Slowest Products | Bar | `Product Name` / `AVG(Warehouse Order Fulfillment (days))`, Top N |

The scatter answers a real question: do products slow to fulfill in the
warehouse also ship late? If the trend is flat, warehouse and transport are
independent problems and need separate fixes.

---

## 8. Assemble and publish

1. **Layout** — one dashboard per area, size *Automatic*. Add the five as tabs.
2. **Filter scope** — set date and department filters to *Apply to all using
   this data source* so tabs stay in sync.
3. **Tooltips** — include order count on every aggregate mark, so a viewer can
   see when an average rests on three rows.
4. **Title** — "Supply Chain Analytics".
5. Publish: **Server → Tableau Public → Save to Tableau Public As…**, sign in
   with your own account.
6. Copy the published URL into `README.md`.
7. Screenshot the first tab, save over `Tableau_Dashboard.png`, and commit.

---

## Numbers to sanity-check against

If the workbook disagrees with these, something is wrong — most likely a join
fanning out, or a missing validity filter.

| Metric | Expected |
|---|---|
| Order lines | 30,871 |
| Usable for shipment timing | 25,882 (83.8%) |
| Excluded as corrupt | 4,989 (16.2%) |
| Median processing time, valid rows | 3 days |
| Mean processing time, valid rows | 3.50 days |
| Scheduled shipment days | only 1, 2, 3 or 4 |
| Mean discount, nulls excluded | 0.1074 |
| Rows missing a discount | 1,749 (5.7%) |
| Product departments / categories | 11 / 49 |
| Date span | 2015-01-01 → 2017-12-31 |
