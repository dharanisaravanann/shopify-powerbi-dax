# DAX Measures

All measures live in the `_Measures` table. Model: `Sales` (fact) with `Customers`, `Category` and `Date` dimensions, all one-to-many, single direction.

## Basic
```DAX
Total Revenue = SUM ( Sales[revenue] )
```
Sum of `revenue` (already discounted price x quantity). Validated against Python: 59,645,935.58.

```DAX
Total Orders = DISTINCTCOUNT ( Sales[order_id] )
```
Counts unique orders. 60,000.

```DAX
Total Units = SUM ( Sales[quantity] )
Total Profit = SUM ( Sales[profit] )
Average Order Value = DIVIDE ( [Total Revenue], [Total Orders] )
Active Customers = DISTINCTCOUNT ( Sales[customer_id] )
```
Units, profit, revenue per order (994.10) and unique customers in the current filter (31,154). DIVIDE avoids divide-by-zero errors.

## Time intelligence
```DAX
Revenue PY = CALCULATE ( [Total Revenue], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
Revenue YoY % = DIVIDE ( [Total Revenue] - [Revenue PY], [Revenue PY] )
Revenue YTD = TOTALYTD ( [Total Revenue], 'Date'[Date] )
Revenue YTD PY = CALCULATE ( [Revenue YTD], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
Revenue Rolling 3M =
CALCULATE ( [Total Revenue], DATESINPERIOD ( 'Date'[Date], MAX ( 'Date'[Date] ), -3, MONTH ) )
```
Prior-year revenue, growth, year-to-date and rolling three-month revenue. All depend on the marked Date table. Data ends 18 June 2025, so full-year 2025 comparisons are not like for like; YTD vs YTD PY is the fair comparison.

## Customers
```DAX
Repeat Customers = COUNTROWS ( FILTER ( VALUES ( Sales[customer_id] ), [Total Orders] > 1 ) )
Repeat Customer Rate = DIVIDE ( [Repeat Customers], [Active Customers] )
```
Customers with 2+ orders (17,832) and their share (57.24%). The `[Total Orders]` measure is evaluated per customer through context transition.

```DAX
Customer Revenue Rank = RANKX ( ALL ( Customers[customer_id] ), [Total Revenue], , DESC, SKIP )
Top 27% Customer Revenue =
VAR TotalCust = CALCULATE ( DISTINCTCOUNT ( Customers[customer_id] ), ALL ( Customers ) )
VAR Cutoff = ROUND ( TotalCust * 0.27, 0 )
RETURN SUMX ( ALL ( Customers[customer_id] ), IF ( [Customer Revenue Rank] <= Cutoff, [Total Revenue] ) )
Top 27% Revenue Share = DIVIDE ( [Top 27% Customer Revenue], CALCULATE ( [Total Revenue], ALL ( Customers ) ) )
```
Ranks customers by revenue, keeps the top 27% (8,412 customers) and divides by total revenue: **55.61%**. Note on ties: about 2,700 customers share an identical revenue total. With `DENSE` ranking the cutoff admitted 8,555 customers and gave 56.23%; `SKIP` matches an independent Python calculation (55.61%).

## Returns and discounts
```DAX
Returned Orders = CALCULATE ( [Total Orders], Sales[is_returned] = 1 )
Return Rate = DIVIDE ( [Returned Orders], [Total Orders] )
Gross Sales = SUMX ( Sales, Sales[product_price] * Sales[quantity] )
Discount Rate = DIVIDE ( [Gross Sales] - [Total Revenue], [Gross Sales] )
```
Returned orders (8,887), return rate (14.81%), pre-discount sales and the revenue-weighted discount rate (18.14%).

```DAX
Share of Total Revenue =
DIVIDE ( [Total Revenue], CALCULATE ( [Total Revenue], ALLSELECTED ( Category[product_category] ) ) )
```
Each category's share of revenue within the current slicer selection.

## Calculated column on `Sales`
```DAX
Discount Band =
SWITCH ( TRUE (),
    Sales[discount_percent] = 0, "1. No discount",
    Sales[discount_percent] <= 10, "2. 5-10%",
    Sales[discount_percent] <= 20, "3. 15-20%",
    "4. 25%+" )
```
A row-level column (not a measure) used to group orders by discount level. Return rate by band: 14.10%, 14.88%, 14.81%, 15.01%.
