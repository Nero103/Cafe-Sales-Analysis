# Cafe-Sales-Analysis

![quentin-rogeret-OqTf1qpXC0A-unsplash](https://github.com/user-attachments/assets/4caf37b7-d050-4d37-9a8d-f485c29e9439)
Photo by <a href="https://unsplash.com/@quentinrogeret?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Quentin Rogeret</a> on <a href="https://unsplash.com/photos/a-window-with-a-screen-and-a-clock-on-it-OqTf1qpXC0A?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>

## Overview
This project analyzes a year's worth of transaction data for a café to uncover trends in product sales, revenue contributions, inventory inconsistencies, and underperforming items. The dataset included 10,000 rows of customer transactions with messy or incomplete attributes, requiring significant data cleaning before insight generation for CPG business.

## Objective & Business Question
This project aims to simulate a real-world business intelligence scenario by analyzing messy transaction data from a fictional café. The goal is to extract actionable insights that inform strategic decision-making in marketing, operations, and inventory control.

### Central Business Question:
How can a café improve revenue, inventory accuracy, and product performance from a year of transaction data? This question guided the entire analytical workflow and led to the development of several sub-questions:

- Which products are driving the most revenue, and which are underperforming?

- Are inventory records consistent, or do gaps suggest operational risks?

- What seasonal or monthly trends affect customer purchasing behavior?

- How can product offerings and promotions be optimized to enhance profitability?

## Data Cleaning Summary
Data cleaning required extensive use of SQL's data definition language (DDL), to update the table with cleaned attributes, and data manipulation language (DML) for feature engineering. Below is the following done to clean the data.

- Used Excel to quickly change the date data attribute from MM/DD/YYYY to a standardized format

- Standardized date format to YYYY-MM-DD HH:MM:SS and extracted month values for time-based analysis.

- Replaced 'ERROR', 'UNKNOWN', and blank strings with NULL across key columns.

- Imputed missing numeric fields (Quantity, Price Per Unit) using the mean, after comparing to median.

```
--Quantity Mean
SELECT coalesce(SUM(Quantity), NULL) / COUNT(Quantity) AS mean_amount,
		AVG(Quantity) AS mean_amount2
FROM new_cafe_sales;

--Quantity Median
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

- Recalculated Total_Spent where possible as Quantity × Price Per Unit.

## Key Insights & Visualizations
After exploring and manipulating the data to uncover interesting patterns, I used ChatGPT to run my queries and creat charts from them. Prompt engineering was needed to specify the charts for consistency.

### Most Purchased Item by Month

Sandwiches, Coffee, and Juice frequently dominated monthly purchases.

Product preference varies monthly, indicating potential for seasonal menu targeting.

![Most Purchase Item by Month](https://github.com/user-attachments/assets/62ea245e-36ca-4cd0-ab32-9739ae05de30)

```
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

Salad and Sandwich generated over 34% of total café revenue.

Cookies and Tea were least profitable despite steady purchases.

![Revenue Contribution By Item](https://github.com/user-attachments/assets/d5d980aa-cbe5-473b-95c9-a87f9783d93e)

```
--Product Contributuin to Total Revenue
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

Overall revenue reached $80,546.96.

Salad led all products with $17.3k in revenue, suggesting strong performance along with its low count of below-average transactions.

![Total Revenue](https://github.com/user-attachments/assets/b90c986a-5398-4fed-b169-e652a5efba7d)

```
--Sales
SELECT
	Item,
	coalesce(SUM(Total_Spent), NULL) AS total_revenue
FROM new_cafe_sales
WHERE Item IS NOT NULL
GROUP BY Item
ORDER BY total_revenue DESC;

--Total Sales
SELECT
	coalesce(SUM(Total_Spent), NULL) AS total_revenue
FROM new_cafe_sales
WHERE Item IS NOT NULL;
```

### Monthly Total Revenue

Monthly revenue fluctuated slightly, with some months dipping below the mean ($7,074).

June and October peaked, suggesting high-traffic promotional periods.

![Monthly Total Revenue](https://github.com/user-attachments/assets/396a5aa5-8702-4c8d-8598-9551cebec853)

```
--Sales over time
SELECT
	Transaction_Month,
	coalesce(SUM(Total_Spent), NULL) AS total_revenue
FROM new_cafe_sales
WHERE Standardized_Date IS NOT NULL
GROUP BY Transaction_Month
ORDER BY Transaction_Month ASC
```

### Inventory Risk

Cookie and Smoothie showed the highest proportion of missing inventory data (30%), indicating supply chain or entry to inventory issues.

All items showed 28–31% inconsistency, flagging need for tighter inventory systems.

![Inventory Risk](https://github.com/user-attachments/assets/32febfab-9d08-454f-ae57-6fe88d43634b)

```
--Inventory
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

Cookie, Tea, and Coffee most often generated below-average sales.

Despite its popularity, Cookie may benefit from repositioning or bundling promotions.

![Low Performing Products](https://github.com/user-attachments/assets/b26abef9-9a86-40f5-9e4a-a31db4ad3bdb)

```
--Find underperforming products to suggest promotions
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

October and March had the highest purchase counts (838 and 827), suggesting a need for staffing at these months.

February lagged behind, likely due to seasonal dips.

![Monthly Purchase Volume](https://github.com/user-attachments/assets/979d5928-9884-4832-9c06-dd45e8d3394a)

```
--Busiest Months of the year
SELECT
	Transaction_Month,
	coalesce(SUM(Total_Spent), NULL) AS total_revenue,
	COUNT(Transaction_ID) AS num_purchases
FROM new_cafe_sales
GROUP BY Transaction_Month
ORDER BY num_purchases DESC
```

## Answers and Conclusions
1. What are the biggest sales contributors?

Salad, Sandwich, and Smoothie accounted for the highest revenue. These items should remain prominent in the café’s menu and promotions.

2. What are the most frequently purchased items?

Sandwiches, Juice, and Coffee consistently led purchases by month.

3. What trends did I find in purchase behavior?

Clear monthly shifts in product popularity, with peak purchase months in October, March, June. Low-performing products like Cookie and Tea may need repositioning.

4. Were there any risks or gaps in the data?

High inventory data inconsistency (30% missing) across items, mostly Cookies and Smoothies. However, Cookies are especially at risk due to both low performance and missing records.

## Recommendations
Based on purchasing trends month-to-month, I suggest that the café should increase staffing around March, June, and October as those seem to be the busiest months. Sandwiches show strong demand not only in being the most purchased items in some months but is a strong revenue driver alongside Salads. These items, especially Sandwiches, would benefit form specials or potential product line extentions. Cookies and Teas would benefit a lot from extra promotions to reposition the tems in the conumer's minds. Moreso for Cookies as they have a high risk to inventory and have the most transactions selling below the benchmarked average. Campaigns to promote cookies would best be suited for July and November, where it is the top selling item for those months.

## Final Takeaways
Promote top earners like Salad and Sandwich while experimenting with bundling for underperformers like Cookies. Improve inventory tracking to ensure reliability and supply for item demand by monthly popularity. Consider seasonal marketing during high-traffic months.
