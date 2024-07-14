# E-Commerce Sales Analysis

## Table of Contents

- [Project Overview](Project-overview)
- [Data Sources](#data-sources)
- [Recommendations](Recommendations)

### Project Overview

This data analysis data project aims to provide insights into the sales performance of
an e-commerce company over the past year. By analyzing various aspects of the sales data.
We seek to identify trends, make data-driven recommendation and gain a deeper understanding of the company's performance.

### Data Sources

Sales Data: The primary dataset used for this analysis is the "Orders.csv" file, containing information about each sale made by the company.

### Tools

--Mysql

### Exploratory Data Analysis

EDA involved the sales data to answear key question, such as:

- What is the overall sales trends?
- Which products are top sellers?
- What are the peak sales periods?

### Data Analysis

Include some interesting code/features worked with

SELECT * FROM Orders;

Let's check for duplicates first
SELECT *
FROM (
SELECT *,
	ROW_NUMBER() OVER(PARTITION BY rowid ) row_num
FROM Orders
)t WHERE row_num>1;

it look like the RowId is unique, so it doesn't contain duplicates
So, lets's go to explore the data

SELECT
	orderdate,
	shipdate
FROM Orders; 

SELECT
	orderdate,
	shipdate,
	STR_TO_DATE(Orderdate, '%m/%d/%Y'),
	STR_TO_DATE(Shipdate, '%m/%d/%Y')
FROM Orders; 

-- use str to date to update this field
UPDATE Orders
SET Orderdate = STR_TO_DATE(Orderdate, '%m/%d/%Y');
UPDATE Orders
SET Shipdate = STR_TO_DATE(Shipdate, '%m/%d/%Y'); 

-- convert the data type properly
ALTER TABLE Orders
MODIFY COLUMN Orderdate date;
ALTER TABLE Orders
MODIFY COLUMN Shipdate date;

SELECT 
	MAX(orderdate), 
	MIN(orderdate),
    MAX(shipdate), 
	MIN(shipdate)
FROM Orders; 

-- Find total sales and profit for each province
SELECT
	province,
	productcategory,
    COUNT(*) OVER(PARTITION BY province),
	SUM(sales) OVER(PARTITION BY province) TotalSalesBYProvince,
	SUM(profit) OVER(PARTITION BY province) TotalProfitBYProvince
FROM Orders
ORDER BY TotalSalesBYProvince, TotalProfitBYProvince; 

-- Find the total number of Orders
-- Find the total number of Orders for each Product Name
-- Additionally provide defauls such product name and order date
SELECT
	Orderdate,
	productcategory,
	productname,
	COUNT(*) OVER() TotalOrders,
	COUNT(*) OVER(PARTITION BY productname) TotalOrdersByProductName
FROM Orders
ORDER BY TotalOrdersByProductName DESC; 

-- Find total number of orders for each customers
SELECT
	orderid,
	orderdate,
	customername,
    productcategory,
	productname,
	COUNT(*) OVER(PARTITION BY customername) TotalOrderByCustomer,
	SUM(orderquantity) OVER(PARTITION BY customername) TotalOrderQuantityByCustomer
FROM Orders
ORDER BY TotalOrderQuantityByCustomer DESC;

SELECT
	productcategory,
    productsubcategory,
	ROUND(SUM(profit) OVER(),2) TotalProfit,
	ROUND(SUM(profit) OVER(PARTITION BY productsubcategory),2) TotalProfitByProductSubCategory
FROM Orders
ORDER BY TotalProfitByProductSubCategory DESC;

-- Find total profit for each month
SELECT
	SUBSTRING(Orderdate, 1,7) Month, 
	SUM(profit) TotalProfit
FROM Orders
GROUP BY SUBSTRING(Orderdate, 1,7)
ORDER BY 1 ASC;

-- Find the Percentage contribution of each orders to the total sales
SELECT
	orderid,
    orderdate,
	productcategory,
	productname,
    SUM(profit) OVER() TotalProfit,
	SUM(profit) OVER(PARTITION BY productcategory) TotalProfitByProductCategory,
	ROUND(SUM(profit) OVER(PARTITION BY productcategory) / SUM(profit) OVER() * 100,2) PercentageOfTotal
FROM Orders
ORDER BY PercentageOfTotal DESC;

SELECT
	customername,
	productname,
	COUNT(*) OVER() TotalOrders,
	SUM(sales) OVER() Totalsales,
	COUNT(*) OVER(PARTITION BY customername) TotalOrdersByCustomer,
	SUM(sales) OVER(PARTITION BY customername) TotalSalesByCustomer,
	COUNT(*) OVER(PARTITION BY customername) / COUNT(*) OVER() * 100 PercentageOfTotalOrders,
	ROUND(SUM(sales) OVER(PARTITION BY customername) / SUM(sales) OVER() * 100,2) PercentageOfTotalSales
FROM Orders
ORDER BY TotalOrdersByCustomer DESC;

-- And find the average sales for each productcategory
-- Additionally provide details such orderID, orderdate
SELECT
	orderid,
	orderdate,
    productcategory,
	sales,
	ROUND(AVG(sales) OVER(), 2) AvgSales,
	ROUND(AVG(sales) OVER(PARTITION BY productcategory), 2) AvgSalesByProductsCategory
FROM Orders;

-- Find all orders where sales amount are higher than the average sales across all orders
SELECT
*
FROM (
SELECT
	orderid,
	orderdate,
    productcategory,
    sales,
	ROUND(AVG(sales) OVER(), 2) AvgSales
FROM Orders
)t 
WHERE sales > AvgSales
ORDER BY sales DESC;

-- Find the deviation of each amount sales from minimum and maximum amount sales
SELECT
	productsubcategory,
	orderdate,
	MAX(sales) OVER() HighestSales,
	MIN(sales) OVER() LowestSales,
	sales - MIN(sales) OVER() DeviationFromMin,
	MAX(sales) OVER() - sales DeviationFromMax
FROM Orders;

-- Find top 10 Orders based on their profit 
SELECT
	*,
	DENSE_RANK() OVER(ORDER BY profit DESC) ProfitRank_Dense
FROM Orders
LIMIT 10;

-- Find 10 Lowest Orders based on their profit 
SELECT
	*,
	DENSE_RANK() OVER(ORDER BY profit DESC) ProfitRank_Dense
FROM Orders
ORDER BY ProfitRank_Dense DESC
LIMIT 10;

WITH Profit_Year (productcategory,years, profit) AS
(
SELECT 
	productcategory, 
	YEAR(orderdate), 
	SUM(profit)
FROM Orders
GROUP BY productcategory, YEAR(orderdate)
), Profit_Year_Rank AS

(SELECT *, 
	DENSE_RANK() OVER (PARTITION BY years ORDER BY profit DESC) AS Ranking
FROM Profit_Year
)
SELECT * FROM Profit_Year_Rank;

-- Find the products that fall within the highest 40% of the profit
SELECT
*
FROM (
SELECT
	Orderid,
	orderdate,
	productcategory,
    productsubcategory,
    productname,
	ROUND(CUME_DIST() OVER(PARTITION BY productcategory ORDER BY profit DESC),4) DistRank
FROM Orders
)t WHERE DistRank <= 0.4;

-- Analyze the year-over-year performance by finding the percentage change
-- in profit between the current and previous year
SELECT
*,
CurrentYearProfit - PreviousYearProfit AS YoY_Change,
ROUND((CurrentYearProfit - PreviousYearProfit)/PreviousYearProfit*100, 2) AS YoY_Percentage
FROM (
SELECT
	YEAR(orderdate) OrderYear,
	SUM(profit) CurrentYearProfit,
    LAG(SUM(profit)) OVER (ORDER BY YEAR(orderdate)) PreviousYearProfit
FROM Orders
GROUP BY
	YEAR(orderdate)
    )t;

-- Analyze the month-over-month performance by finding the percentage change
-- in profit between the current and previous month
SELECT
*,
CurrentMonthProfit - PreviousMonthProfit AS MoM_Change,
ROUND((CurrentMonthProfit - PreviousMonthProfit)/PreviousMonthProfit*100, 2) AS MoM_Percentage
FROM (
SELECT
	SUBSTRING(Orderdate, 1,7) Month, 
    SUM(profit) CurrentMonthProfit,
    LAG(SUM(profit)) OVER (ORDER BY SUBSTRING(Orderdate, 1,7)) PreviousMonthProfit
FROM Orders
GROUP BY SUBSTRING(Orderdate, 1,7)
)t;

### Result/Findings

1. The company's sales and profit have been steadily increasing over the past year
2.  Product Category Tecnology is the best performing category in terms of sales and profit
3.  Customer segment corporate should be targeted for marketing efforts

### Recommendation

Based on the analysis, we reccomend the following actions:
- Invest in marketing and promotions during peak sales season to maximize profit
- Focus on expanding and promoting products in Category Technology
- Implement a customer segmentation strategy to target corporate customers effectively








