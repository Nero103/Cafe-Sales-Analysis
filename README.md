# Cafe-Sales-Analysis

![quentin-rogeret-OqTf1qpXC0A-unsplash](https://github.com/user-attachments/assets/4caf37b7-d050-4d37-9a8d-f485c29e9439)
Photo by <a href="https://unsplash.com/@quentinrogeret?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Quentin Rogeret</a> on <a href="https://unsplash.com/photos/a-window-with-a-screen-and-a-clock-on-it-OqTf1qpXC0A?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>

## Overview
This analysis leverages Excel, SQL, and Power BI to explore a year's worth of café transaction data (10,000 purchases). The focus is on uncovering sales drivers, low performers, revenue trends, and inventory status. Insights are drawn to support data-driven decisions for marketing, product offerings, and operational supply chain improvements.

## Objective & Business Question
This project aims to simulate a real-world data analysis scenario by analyzing messy transaction data from a fictional café. The goal is to extract actionable insights that inform strategic decision-making in marketing, operations, and inventory control.

### Business Objective
**Primary Question:**
How can the café improve revenue, optimize inventory, and enhance product performance using sales data insights?

**Additional Questions:**

- Which items generate the most revenue?

- What are the most and least purchased items each month?

- Are certain products consistently underperforming?

- What are the busiest months for café transactions?

- Are there indicators of inventory risk?

## Data Cleaning Summary
Data cleaning required extensive use of SQL's data definition language (DDL), to update the table with cleaned attributes, and data manipulation language (DML) for feature engineering. Excel and Power BI were also used to standardize and model the data for specific Power BI functions. Below is the following done to clean the data.

- *cafe_sales* was saved as *dirty_cafe_sales* to retain the original raw data

- Used Excel to quickly change the date data attribute from MM/DD/YYYY to a standardized format

- Standardized date format in SQL to YYYY-MM-DD HH:MM:SS and extracted month values for time-based analysis.

- Replaced 'ERROR', 'UNKNOWN', and blank strings with null or 'Unspecified' across key columns. This step was especially taken in Power BI's Power Query using Replace

- Since the percent of null data for transaction dates was 4.6% (0.046) of the dataset, and a general rule of thumb is that less than 5% can be filtered out without impacting that data, the nulls in transaction date were filtered out; and *dirty_cafe_sales* was saved as *new_cafe_sales* for later analysis and visualization in Power BI

- Imputed missing numeric fields (Quantity, Price Per Unit) using the mean, after comparing to median and mean in SQL, as seen below.

```
--SQL Quantity Mean
SELECT coalesce(SUM(Quantity), NULL) / COUNT(Quantity) AS mean_amount,
		AVG(Quantity) AS mean_amount2
FROM new_cafe_sales;

--SQL Quantity Median
WITH Ordered AS (
SELECT 
	Quantity,
	ROW_NUMBER() OVER (ORDER BY Quantity) AS rn,
	COUNT(*) OVER () AS total_count
FROM cafe_sales
WHERE Quantity IS NOT NULL
),
Middle AS (
SELECT
	Quantity
FROM Ordered
WHERE rn = (total_count + 1) / 2
	OR rn = (total_count + 2 / 2) --for even or odd cases
)
SELECT
	AVG(Quantity * 1.0) AS median_amount
FROM Middle
```

## Data Maipulation Summary
After cleaning the data in SQL and Power Query, further data manipulation was conducted for use in exploratory anaylsis

- Recalculated Total_Spent where possible as Quantity × Price Per Unit in SQL and then in Power Query
  
- Extracted the Year, Month, and Day from the Transaction_Date column

- Created a Date Table in Power BI and joined the Date Table to new-cafe_sales table, with a cardinality of one-to-many and single cross filter, for more accurate date granularity and calculations

- Created Popular Products, Ranked Products, and Top Ranked Products by Month tables using DAX from prior built SQL queries for later analyses and visualizations

```
--Dax Table: Popular Products
Popular_Products = 
SUMMARIZE(
    FILTER(new_cafe_sales, new_cafe_sales[Item] <> "Unspecified"),
    Date_Table[Month_Number],
    new_cafe_sales[Item],
    "Num_Of_Purchases", COUNTROWS(new_cafe_sales)
)

--Dax Table: Ranked Products
Ranked_Products = 
ADDCOLUMNS(
    Popular_Products,
    "Rank",
    RANKX(
        FILTER(
            Popular_Products,
            Popular_Products[Month_Number] = EARLIER(Popular_Products[Month_Number])
        ),
        [Num_Of_Purchases],
        ,
        DESC,
        DENSE
    )
)

--Dax Table: Top Ranked Products By Month
Top_Ranked_Products_By_Month = 
FILTER(
    Ranked_Products,
    [Rank] = 1
)
```

## Key Insights & Visualizations
After exploring, manipulating, and analyzing the data in SQL to uncover interesting patterns, Power BI was used to run my queries as in the BI tool and creat charts. DAX coding was needed to specify the charts and match findings in SQL.

### Most Purchased Items by Month
Coffee and Sandwiches are the most purchased item in a month.

Juice are the most consistently purchased item throughout the year.

Juice, Coffee, and Cookie also showed strong monthly performance.

Across all 12 Months, purchases ranged from 94 to 115.

Purchasing behavior shifts slightly by season, indicating potential for seasonal menu curation.

![image](https://github.com/user-attachments/assets/4f632e81-f920-4c70-b2f9-ed89f8ea94ff)

```
--SQL Query equivalent to above DAX tables: Popular Products, Ranked Products, and Top Ranked Products By Month
WITH popular_products AS ( 
SELECT
	Item,
	Transaction_Month,
	COUNT(*) AS num_of_purchases
FROM new_cafe_sales
WHERE Item IS NOT NULL
GROUP BY Item, Transaction_Month
),
ranked_purchases AS (
SELECT
	Transaction_Month,
	Item,
	num_of_purchases,
	DENSE_RANK() OVER (PARTITION BY Transaction_Month ORDER BY num_of_purchases DESC) AS rnk
FROM popular_products
WHERE Transaction_Month IS NOT NULL
)
SELECT
	Transaction_Month,
	Item,
	num_of_purchases
FROM ranked_purchases
WHERE rnk = 1
ORDER BY Transaction_Month ASC, num_of_purchases DESC
```

### Revenue Contribution by Item
Salad was the top revenue generator, contributing ~21%  ($15.2K) of total revenue ($71.74K).

Sandwiches and Smoothies followed closely, each generating over $11K.

Cookies and Tea contributed the least to revenue despite being top transaction volumes in certain months.

![image](https://github.com/user-attachments/assets/e72f8a72-b1c7-47cb-9ff8-753fe07618b1)

```
--SQL Product Contribution to Total Revenue
WITH top_products AS (
SELECT
	Item,
	SUM(Total_Spent) AS total_revenue
FROM new_cafe_sales
GROUP BY Item
ORDER BY total_revenue DESC
)
SELECT
	 Item,
	 total_revenue,
	 ROUND(total_revenue / (SELECT SUM(total_Spent) FROM new_cafe_sales), 2) AS percent_of_total_revenue
FROM top_products
WHERE Item IS NOT NULL

```

### Total Revenue by Item
Top Performers: Salad ($15.2K), Sandwich ($12.0K), Smoothie ($11.9K)

Lowest Earners: Cookie ($3.0K), Tea ($4.4K)

Total Revenue: $71.74K

![image](https://github.com/user-attachments/assets/491dfc11-f646-4d0e-bcbd-aab9c4bb00a8)

```
--SQL Sales
SELECT
	Item,
	coalesce(SUM(Total_Spent), NULL) AS total_revenue
FROM new_cafe_sales
WHERE Item IS NOT NULL
GROUP BY Item
ORDER BY total_revenue DESC;

--SQL Total Sales
SELECT
	coalesce(SUM(Total_Spent), NULL) AS total_revenue
FROM new_cafe_sales
WHERE Item IS NOT NULL;
```

### Monthly Total Revenue
Revenue remained relatively stable around the mean ($6,605/month).

Notable peaks in March, June, and October, suggesting high-traffic months and likely candidates for promotional or seasonal factors.

![image](https://github.com/user-attachments/assets/7b36c370-2a70-4c41-bf34-3fe788dfe939)

```
--SQL Sales over time
SELECT
	Transaction_Month,
	coalesce(SUM(Total_Spent), NULL) AS total_revenue
FROM new_cafe_sales
WHERE Standardized_Date IS NOT NULL
GROUP BY Transaction_Month
ORDER BY Transaction_Month ASC
```

### Inventory Risk

Highest inventory risks flagged for Sandwich, Coffee, and Salad, each with over 37 inconsistencies.

In context of sales and purchase peaks by month, Cookies and Tea showed elevated levels of missing inventory records, signaling potential supply chain or data entry issues.

![image](https://github.com/user-attachments/assets/65744f68-afbb-4123-953f-1e158b0a4a89)

```
--SQL Inventory
WITH total_quantity AS (
SELECT
	Item,
	coalesce(SUM(Quantity), NULL) AS total_inventory
FROM new_cafe_sales
WHERE Item IS NOT NULL
GROUP BY Item
)
SELECT
	Item,
	total_inventory,
	ROUND(((SELECT COUNT(Item_MissingFlag) FROM new_cafe_sales WHERE Item_MissingFlag = 'yes') 
	/ total_inventory), 2) AS pct_missing_inventory
FROM total_quantity
ORDER BY pct_missing_inventory DESC

```

### Low-Performing Products

Tea, Smoothie, Cake, and Cookie had the highest volume of below-average revenue transactions with Tea being of concern.

Despite moderate sales frequency, Cookies and Tea underperformed in revenue and carried inventory risk, making them a target for repositioning or bundling promotions.
Despite its popularity, Cookie and Tea may benefit from repositioning or bundling promotions.

![image](https://github.com/user-attachments/assets/189acf32-cd82-4d2c-b567-1e9d9cde7674)

```
--SQL Find underperforming products to suggest promotions
WITH mean_value AS (
SELECT
	AVG(Total_Spent) AS mean_revenue
FROM new_cafe_sales
WHERE Total_Spent IS NOT NULL
),
Comparison_metrics AS (
SELECT
	Item,
	Total_Spent,
	(SELECT mean_revenue FROM mean_value) AS mean_spent,
	CASE
		WHEN Total_Spent > (SELECT mean_revenue FROM mean_value) THEN 'Above Average'
		WHEN Total_Spent < (SELECT mean_revenue FROM mean_value) THEN 'Below Average'
		ELSE 'Average'
		END AS Comparison
FROM new_cafe_sales
WHERE Total_Spent IS NOT NULL AND Item IS NOT NULL
)
SELECT
	Item,
	COUNT(*) AS num_below_average
FROM Comparison_metrics
WHERE Comparison = 'Below Average'
GROUP BY Item
ORDER BY num_below_average DESC
```

### Monthly Purchase Volume

October (highest) and March saw the greatest number of purchases (both above 830), indicating the need for seasonal staffing adjustments.

February had the lowest transaction volume, possibly due to seasonality or reduced foot traffic.

Mean monthly transactions were 795

![image](https://github.com/user-attachments/assets/bb3f9b10-bc2b-478a-ab36-b075e817ae08)

```
--SQL Busiest Months of the year
SELECT
	Transaction_Month,
	coalesce(SUM(Total_Spent), NULL) AS total_revenue,
	COUNT(Transaction_ID) AS num_purchases
FROM new_cafe_sales
GROUP BY Transaction_Month
ORDER BY num_purchases DESC
```

## Dashboard Metrics Summary

1. Which items generate the most revenue?

Salad: ~$15.2K (≈21% of total revenue)
Sandwich: ~$12.0K
Smoothie: ~$11.9K

These three items collectively account for a significant share of the café’s annual revenue and should be core offerings in the menu strategy.

2. What are the most and least purchased items each month?

Most purchased items by month include:

- Juice (most frequently appearing at the top across months)
- Sandwich, Coffee, and Cookie also rank high in several months.

Least purchased items (based on low bar representation in monthly purchase charts):

- Tea, Cake, and Smoothie appear least in top purchased products, and Cookies lag behind other items in specific months

3. Are certain products consistently underperforming?

Yes, the following items consistently generate below-average revenue per transaction:

- Tea
- Smoothie
- Cake
- Cookie

Notably, Cookies stand out for having both high transaction volume and a large number of low-revenue transactions, indicating poor profitability despite popularity. A clear opportunity for repositioning or bundling strategies, especially with Tea since teas has the most below average purchases to revenue.

4. What are the busiest months for café transactions?

October and March had the highest purchase volumes (above 830 transactions each).

These months are likely high-traffic periods, warranting increased staffing and stock level, and/or promotions.

February was the least busy month.

5. Are there indicators of inventory risk?

Yes. The Inventory Risk by Item visualization shows:

- Sandwich, Coffee, and Salad had the highest inventory discrepancies (over 37 flagged instances).
- Tea, Cookies, Juices, and Smoothies also showed elevated inventory inconsistencies.
- This points to potential data entry issues, supply inconsistencies, or operational tracking problems, especially concerning since some of these items are also top sellers such as Salad

![image](https://github.com/user-attachments/assets/3e9ea528-a60c-45e0-b088-9b30e910d48f)

## Recommendations
- Promote high-revenue items like Salad and Sandwich via combos, features, or seasonal specials.

- Reposition underperformers like Cookies and Tea through targeted campaigns or bundling, especially during July and November when Cookie purchases peak.

- Optimize staffing during March, June, and October to accommodate high transaction volumes.

- Audit inventory systems, especially for high-risk items (e.g., Sandwiches, Coffee, Cookies), to reduce inconsistencies and ensure stock alignment with demand.

- Use dashboard slicers (by quarter) to tailor promotional and operational plans per season.

## Final Takeaways
The dashboard reveals a strong opportunity to maximize café profitability through product focus, seasonal planning, and inventory control. Data visualizations support targeted actions that can directly impact customer satisfaction through item demand and revenue growth.
