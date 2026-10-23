# Power BI Practice Project: Customers, Products, Orders

Follow the parts in order. Every step starts with a **[WINDOW]** tag that tells you where to do it.

## The windows you will use

| Tag | Where it is | How to open it |
|---|---|---|
| **[Report view]** | Main canvas where visuals live | Left sidebar, 1st icon (bar chart) |
| **[Table view]** | Grid of loaded data | Left sidebar, 2nd icon (table) |
| **[Model view]** | Relationship diagram | Left sidebar, 3rd icon (three linked boxes) |
| **[Power Query Editor]** | Separate window for cleaning data | Home tab > **Transform data** |

Short cuts used throughout: **Ribbon** = the tabs at the top (Home, Transform, Add Column, View...). **Applied Steps** = right-hand pane inside Power Query Editor.

## The files

| File | Rows | Columns |
|---|---|---|
| `customers.csv` | 1,015 (1,000 unique + 15 duplicates) | CustomerID, CustomerName, Email, Phone, Gender, DateOfBirth, City, State, Segment, JoinDate |
| `products.csv` | 120 | ProductID, ProductName, Category, SubCategory, Brand, UnitPrice, UnitCost, Supplier, StockQty |
| `orders.csv` | 5,000 | OrderID, OrderDate, ShipDate, CustomerID, ProductID, Quantity, Discount, PaymentMode, OrderStatus, ShipMode, Rating |

Deliberate problems hidden in the data (you will fix them):

- **Nulls** in: Customers (Email, Phone, Gender, DateOfBirth, City, State, Segment), Products (Brand, UnitCost, Supplier, StockQty), Orders (ShipDate, Quantity, Discount, PaymentMode, Rating).
- **15 duplicate customer rows** (this would break the relationship if left).
- **Messy text**: some names are UPPERCASE, some lowercase with extra spaces, some cities have a trailing space.
- Dates are in `YYYY-MM-DD` format so they load correctly in any regional setting.

---

# PART 1: Load the data

1. **[Power BI Desktop]** Open a blank report. Save it as `Sales_Project.pbix`.
2. **[Report view]** Home tab > **Get data** > **Text/CSV** > pick `customers.csv` > **Open**.
3. In the preview dialog click **Transform Data** (not Load). This opens **Power Query Editor**.
4. **[Power Query Editor]** Ribbon: Home > **New Source** > **Text/CSV** > `products.csv` > **OK**. Repeat for `orders.csv`.
5. You should now see 3 queries in the left **Queries** pane: `customers`, `products`, `orders`.
6. Rename them to **Customers**, **Products**, **Orders**: right-click each query > **Rename**.

---

# PART 2: Power Query cleaning (all in Power Query Editor)

## 2.0 Turn on data profiling first (do this once)

**[Power Query Editor]** View tab > tick **Column quality**, **Column distribution**, **Column profile**. Then at the bottom-left of the window, click the "Column profiling based on top 1000 rows" text and change it to **Column profiling based on entire data set**.

Now every column shows a small bar: green = valid, grey = **empty (null)**, red = error. This is how you find nulls.

## 2.1 Customers query

Click **Customers** in the Queries pane. Do these in order.

**Step 1: Remove duplicates**
1. Click the header of **CustomerID**.
2. Home tab > **Remove Rows** > **Remove Duplicates**.
3. Check the status bar at the bottom: rows should drop from 1,015 to **1,000**.

**Step 2: Clean the text**
1. Click **CustomerName** header. Transform tab > **Format** > **Trim**.
2. Transform tab > **Format** > **Capitalize Each Word**. (Fixes UPPER and lower case.)
3. Click **City** header > Transform > Format > **Trim**.
4. Click **City** > Transform > Format > **Capitalize Each Word**.

**Step 3: Set data types**
Click the small type icon at the left of each column header and choose:

| Column | Type |
|---|---|
| CustomerID, CustomerName, Email, Phone, Gender, City, State, Segment | Text |
| DateOfBirth, JoinDate | Date |

Phone must be **Text**, not a number, so it is not shown as 9.8E+09.

**Step 4: Replace the null values**
Select the column > Transform tab > **Replace Values**. In **Value To Find** type `null` (lowercase), in **Replace With** type the value below > **OK**.

| Column | Replace null with | Why |
|---|---|---|
| Email | `Not Provided` | Text placeholder, keeps the row |
| Phone | `Not Provided` | Same |
| Gender | `Unknown` | Keeps the category visible in charts |
| City | `Unknown` | Avoids a blank bar in map and chart visuals |
| State | `Unknown` | Same |
| Segment | `Unassigned` | Same |
| **DateOfBirth** | **Do NOT replace** | A fake date would give wrong ages. We handle blanks in DAX later |

> **Tip:** If Replace Values reports no change, first confirm the column type is Text, then try again with `null` typed in lowercase. Empty cells in these CSVs load as true nulls, so this works.

**Step 5: Verify**
Look at the column quality bars: Email, Phone, Gender, City, State, Segment should show **0% empty**. Only DateOfBirth should still show about 3% empty. Good.

## 2.2 Products query

Click **Products**.

**Step 1: Set data types**

| Column | Type |
|---|---|
| ProductID, ProductName, Category, SubCategory, Brand, Supplier | Text |
| UnitPrice, UnitCost | **Fixed decimal number** (right-click type icon list) |
| StockQty | Whole number |

**Step 2: Replace nulls (simple ones)**

| Column | Replace null with |
|---|---|
| Brand | `Unbranded` |
| Supplier | `Unknown Supplier` |
| StockQty | `0` |

**Step 3: Replace UnitCost nulls with a calculated value (smarter fix)**
A cost of 0 would make profit look huge, so we estimate cost as 70% of price.

1. Add Column tab > **Custom Column**.
2. New column name: `UnitCost Clean`
3. Formula:
   ```
   if [UnitCost] = null then [UnitPrice] * 0.7 else [UnitCost]
   ```
4. Click **OK**.
5. Right-click the old **UnitCost** column header > **Remove**.
6. Right-click **UnitCost Clean** > **Rename** > `UnitCost`.
7. Set its type to **Fixed decimal number**.

## 2.3 Orders query

Click **Orders**.

**Step 1: Set data types**

| Column | Type |
|---|---|
| OrderID, CustomerID, ProductID, PaymentMode, OrderStatus, ShipMode | Text |
| OrderDate, ShipDate | Date |
| Quantity, Rating | Whole number |
| Discount | Decimal number |

**Step 2: Replace nulls**

| Column | Replace null with | Why |
|---|---|---|
| Quantity | `1` | An order must have at least 1 item |
| Discount | `0` | No discount recorded means no discount given |
| PaymentMode | `Unknown` | Keep the row, label it |
| **ShipDate** | **Do NOT replace** | Null is the truth: cancelled and pending orders never shipped |
| **Rating** | **Do NOT replace** | Replacing with 0 would drag the average down. Blank is ignored by AVERAGE |

> **Rule to remember:** replace nulls when a sensible default exists; leave them null when the blank itself carries meaning (not shipped, not rated, unknown birth date).

**Step 3: Check text values**
Click the **OrderStatus** header's filter arrow: you should see Delivered, Shipped, Cancelled, Returned, Pending only.

## 2.4 Optional extra Power Query practice

Do these in **Orders** (Add Column tab):

1. **Order Year**: select OrderDate > Add Column > **Date** > **Year** > **Year**.
2. **Order Month Name**: select OrderDate > Add Column > Date > **Month** > **Name of Month**.
3. **Conditional column**: Add Column > **Conditional Column** > name `Order Size`: if Quantity >= 5 then `Bulk`, else if Quantity >= 2 then `Medium`, else `Single`.

In **Customers**: Add Column > **Column From Examples** > type an example of first name from CustomerName (e.g. `Aarav`) and let Power Query guess the pattern.

## 2.5 Apply

**[Power Query Editor]** Home tab > **Close & Apply**. You return to the main window with all 3 tables loaded.

> You can reopen Power Query Editor any time: Home tab > **Transform data**.

---

# PART 3: Data modelling

## 3.1 Create relationships

1. **[Model view]** (left sidebar, 3rd icon). You will see 3 table boxes.
2. Drag **CustomerID** from **Customers** onto **CustomerID** in **Orders**.
3. Drag **ProductID** from **Products** onto **ProductID** in **Orders**.
4. Double-click each relationship line and confirm:
   - Cardinality: **One to many (1:\*)**, "one" side on Customers / Products
   - Cross filter direction: **Single**
   - Make this relationship active: **ticked**

If Power BI refuses with "many to many" or "duplicate values", it means duplicates survived in Part 2.1 Step 1. Go back and fix.

## 3.2 Create a Date (Calendar) table

1. **[Report view]** or **[Table view]** > Home tab > **New table**.
2. Paste this in the formula bar and press Enter:
   ```DAX
   Calendar =
   ADDCOLUMNS (
       CALENDAR ( DATE ( 2024, 1, 1 ), DATE ( 2026, 12, 31 ) ),
       "Year", YEAR ( [Date] ),
       "Month No", MONTH ( [Date] ),
       "Month", FORMAT ( [Date], "MMM" ),
       "Quarter", "Q" & QUARTER ( [Date] ),
       "Year-Month", FORMAT ( [Date], "YYYY-MM" ),
       "Weekday", FORMAT ( [Date], "ddd" )
   )
   ```
3. **[Table view]** > click the **Calendar** table in the Data pane > **Table tools** tab > **Mark as date table** > choose column **Date** > OK.
4. **Fix month sorting:** **[Table view]** click the **Month** column > **Column tools** tab > **Sort by column** > **Month No**.
5. **[Model view]** drag **Date** (Calendar) onto **OrderDate** (Orders). Also set cardinality 1:\*, Single.

Optional: create a second relationship from Calendar[Date] to Orders[ShipDate]. It will appear as a dotted (inactive) line; use `USERELATIONSHIP` in DAX to activate it (see Part 4.7).

## 3.3 Tidy the model

1. **[Model view]** Right-click **Orders[OrderID]** etc. and hide technical key columns from Report view: right-click > **Hide in report view** (ID columns on the "many" side).
2. Make the layout a **star schema**: Customers and Products on top, Orders in the middle, Calendar on top as well.
3. **[Table view]** select Orders[Discount] > **Column tools** > Format: **Percentage**.

---

# PART 4: DAX

**Where to write DAX:**
- *Calculated column* or *calculated table*: **[Table view]** or **[Report view]**, select the table in the Data pane > **Table tools** tab > **New column** / Home > **New table**.
- *Measure*: **[Report view]** > select the table in the Data pane > Home tab (or Table tools) > **New measure**.

Always type the formula in the formula bar and press **Enter**.

## 4.1 Create a home for measures

1. **[Report view]** Home tab > **Enter data**.
2. Name the table `_Measures`, leave the grid as is > **Load**.
3. You will place every measure in this table (select `_Measures` before clicking New measure).

## 4.2 Calculated columns

**[Table view]**, select the table named in each heading, then Table tools > **New column**.

**In Orders:**
```DAX
Sales Amount =
Orders[Quantity] * RELATED ( Products[UnitPrice] ) * ( 1 - Orders[Discount] )
```
```DAX
Cost Amount =
Orders[Quantity] * RELATED ( Products[UnitCost] )
```
```DAX
Profit = Orders[Sales Amount] - Orders[Cost Amount]
```
```DAX
Delivery Days =
IF (
    ISBLANK ( Orders[ShipDate] ),
    BLANK (),
    DATEDIFF ( Orders[OrderDate], Orders[ShipDate], DAY )
)
```

**In Customers:**
```DAX
Age =
IF (
    ISBLANK ( Customers[DateOfBirth] ),
    BLANK (),
    DATEDIFF ( Customers[DateOfBirth], TODAY (), YEAR )
)
```
```DAX
Age Group =
SWITCH (
    TRUE (),
    ISBLANK ( Customers[Age] ), "Unknown",
    Customers[Age] < 25, "Under 25",
    Customers[Age] < 35, "25-34",
    Customers[Age] < 45, "35-44",
    Customers[Age] < 55, "45-54",
    "55+"
)
```
```DAX
Customer Value Tier =
VAR CustomerSales = CALCULATE ( SUM ( Orders[Sales Amount] ) )
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( CustomerSales ), "No Orders",
        CustomerSales >= 450000, "Platinum",
        CustomerSales >= 230000, "Gold",
        CustomerSales >= 100000, "Silver",
        "Bronze"
    )
```
(`CALCULATE` inside a calculated column converts the current row into a filter. This is called *context transition*.)

**Format:** select each money column in Table view > Column tools > Format: Currency (INR) or Decimal with 2 places.

## 4.3 Basic measures

**[Report view]**, select `_Measures`, then Home > **New measure**.

```DAX
Total Sales = SUM ( Orders[Sales Amount] )
```
```DAX
Total Cost = SUM ( Orders[Cost Amount] )
```
```DAX
Total Profit = [Total Sales] - [Total Cost]
```
```DAX
Profit Margin % = DIVIDE ( [Total Profit], [Total Sales],0 )
```
```DAX
Total Orders = DISTINCTCOUNT ( Orders[OrderID] )
```
```DAX
Total Quantity = SUM ( Orders[Quantity] )
```
```DAX
Total Customers = DISTINCTCOUNT ( Orders[CustomerID] )
```
```DAX
Avg Order Value = DIVIDE ( [Total Sales], [Total Orders] )
```
```DAX
Avg Rating = AVERAGE ( Orders[Rating] )
```
```DAX
Avg Delivery Days = AVERAGE ( Orders[Delivery Days] )
```

Set formats: Measure tools tab > Format. Sales/Profit/AOV = Currency; Margin % = Percentage.

## 4.4 CALCULATE and filter measures

```DAX
Net Sales =
CALCULATE ( [Total Sales], Orders[OrderStatus] IN { "Delivered", "Shipped" } )
```
```DAX
Cancelled Orders =
CALCULATE ( [Total Orders], Orders[OrderStatus] = "Cancelled" )
```
```DAX
Cancel Rate % = DIVIDE ( [Cancelled Orders], [Total Orders] )
```
```DAX
Return Rate % =
DIVIDE ( CALCULATE ( [Total Orders], Orders[OrderStatus] = "Returned" ), [Total Orders] )
```
```DAX
Electronics Sales =
CALCULATE ( [Total Sales], Products[Category] = "Electronics" )
```
```DAX
UPI Sales =
CALCULATE ( [Total Sales], Orders[PaymentMode] = "UPI" )
```

## 4.5 Percent of total, ranking, ALL family

```DAX
% of All Sales =
DIVIDE ( [Total Sales], CALCULATE ( [Total Sales], ALL ( Orders ) ) )
```
```DAX
% of Category Sales =
DIVIDE ( [Total Sales], CALCULATE ( [Total Sales], ALLSELECTED ( Products ) ) )
```
```DAX
Product Rank =
RANKX ( ALL ( Products[ProductName] ), [Total Sales], , DESC, DENSE )
```
```DAX
Top 5 Product Sales =
CALCULATE (
    [Total Sales],
    TOPN ( 5, ALL ( Products[ProductName] ), [Total Sales], DESC )
)
```

## 4.6 Time intelligence (needs the Calendar table from Part 3.2)

```DAX
Sales YTD = TOTALYTD ( [Total Sales], 'Calendar'[Date] )
```
```DAX
Sales MTD = TOTALMTD ( [Total Sales], 'Calendar'[Date] )
```
```DAX
Sales LY = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Calendar'[Date] ) )
```
```DAX
Sales YoY % = DIVIDE ( [Total Sales] - [Sales LY], [Sales LY] )
```
```DAX
Sales Last Month = CALCULATE ( [Total Sales], DATEADD ( 'Calendar'[Date], -1, MONTH ) )
```
```DAX
Sales MoM % = DIVIDE ( [Total Sales] - [Sales Last Month], [Sales Last Month] )
```
```DAX
Running Total Sales =
CALCULATE (
    [Total Sales],
    FILTER ( ALL ( 'Calendar'[Date] ), 'Calendar'[Date] <= MAX ( 'Calendar'[Date] ) )
)
```
```DAX
Rolling 3M Sales =
CALCULATE (
    [Total Sales],
    DATESINPERIOD ( 'Calendar'[Date], MAX ( 'Calendar'[Date] ), -3, MONTH )
)
```

## 4.7 Advanced: variables, iterators, inactive relationships

```DAX
Margin Status =
VAR m = [Profit Margin %]
RETURN
    SWITCH ( TRUE (), m >= 0.25, "High", m >= 0.18, "Medium", "Low" )
```
```DAX
Repeat Customers =
COUNTROWS ( FILTER ( VALUES ( Customers[CustomerID] ), [Total Orders] > 1 ) )
```
```DAX
Repeat Customer % = DIVIDE ( [Repeat Customers], [Total Customers] )
```
```DAX
Customer Lifetime Value = DIVIDE ( [Total Sales], [Total Customers] )
```
```DAX
Sales by Ship Date =
CALCULATE ( [Total Sales], USERELATIONSHIP ( Orders[ShipDate], 'Calendar'[Date] ) )
```
(Only works if you created the optional second relationship in Part 3.2.)

```DAX
Selected Year Title =
"Sales Dashboard for " & SELECTEDVALUE ( 'Calendar'[Year], "All Years" )
```

---

# PART 5: Build the report

**[Report view]**. Add a new page for each of these, dragging fields from the Data pane onto the canvas.

**Page 1: Overview**
1. 4 **Card** visuals: Total Sales, Total Profit, Total Orders, Profit Margin %.
2. **Line chart**: X-axis = Calendar[Year-Month], Y-axis = Total Sales, second line = Sales LY.
3. **Clustered bar chart**: Axis = Products[Category], Values = Total Sales.
4. **Slicers** (Visualizations pane > Slicer): Calendar[Year], Customers[State], Orders[OrderStatus].

**Page 2: Products**
1. **Table**: ProductName, Total Sales, Total Profit, Profit Margin %, Product Rank.
2. **Donut chart**: Products[Category], Total Sales.
3. In the table, use **Filters pane** > ProductName > Filter type **Top N** > Top 10 by Total Sales.

**Page 3: Customers**
1. **Column chart**: Customers[Age Group], Total Customers.
2. **Matrix**: Rows = Customers[State] > Customers[City], Values = Total Sales, Avg Order Value.
3. **Stacked column**: Customer Value Tier by Total Customers.
4. **Map** (or filled map): Customers[City] with Total Sales as bubble size.

**Page 4: Operations**
1. **Gauge** or card: Cancel Rate %, Return Rate %, Avg Delivery Days.
2. **Bar chart**: Orders[ShipMode], Avg Delivery Days.
3. **Column chart**: Orders[PaymentMode], Total Orders.
4. **Card**: Avg Rating.

**Extras**
- Use the Margin Status measure for conditional formatting: select a visual > Format > **Conditional formatting** > Font color > **Rules**.
- Create a **Drillthrough** page: Page 2, drag Products[ProductName] into the *Drill through* well.
- Add a **Bookmark** and a **Button** for navigation between pages (Insert tab > Buttons > Navigator > Page navigator).

---

# PART 6: Checks (do not skip)

| # | Check | Where | Expected |
|---|---|---|---|
| 1 | Customers row count | Power Query Editor, status bar | 1,000 |
| 2 | Orders row count | Power Query Editor, status bar | 5,000 |
| 3 | Null cells in Email, Phone, Gender, City, State, Segment | Power Query column quality | 0% empty |
| 4 | ShipDate and Rating still have blanks | Power Query column quality | Yes (on purpose) |
| 5 | Relationships shown as 1:\* | Model view | 3 relationships (4 with optional ShipDate one) |
| 6 | Total Orders card | Report view | 5,000 |
| 7 | Total Customers card | Report view | 964 (36 customers never ordered) |
| 8 | Total Sales | Report view | About 188 million (the exact value depends on how you filled nulls) |
| 9 | Profit Margin % | Report view | About 21% |

---

# Cheat sheet: how to treat nulls

| Situation | Action |
|---|---|
| Text attribute missing (city, gender, brand) | Replace with `Unknown` / `Unassigned` / `Not Provided` |
| Number that must exist (quantity) | Replace with sensible minimum (1) |
| Number where "none" is a fact (discount) | Replace with 0 |
| Number that can be estimated (unit cost) | Calculate from another column with a Custom Column |
| Blank that carries meaning (ship date of cancelled order, rating, date of birth) | **Leave null**, handle with `ISBLANK`, `IF`, or let `AVERAGE` ignore it |

# Common errors and fixes

| Problem | Fix |
|---|---|
| "Can't create relationship: many-to-many" | Duplicates in a dimension table. Remove duplicates on the key column in Power Query |
| Month names sort alphabetically | Table view > select Month > Column tools > Sort by column > Month No |
| Phone number appears as 9.8E+09 | Change the column type to Text in Power Query |
| Time intelligence returns blank | Calendar not marked as date table, or relationship to OrderDate missing |
| Replace Values does nothing | Make sure you typed `null` in Value To Find, and the column type is set first |
| Measure shows the same total in every row | Missing relationship, or you used a column from the wrong table |
