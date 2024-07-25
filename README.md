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
- Sales Transactions Data: Details of each transaction (order ID, date, product ID, customer ID, quantity, price, 
   discount, total amount).
- Product Information Data: Product details (product ID, category).
- Customer Information Data: Customer details (customer ID, location).
The primary dataset used for this analysis is the "Orders.csv" file.

### Tools
- Database Management: MySQL
- Business Intelligence Tool: Tableau

### Objectives
- Analyze sales performance over different periods.
- Identify top-selling products and categories.
- Understand customer purchasing behavior.
- Evaluate the impact of promotions and discounts.
- Provide actionable insights for business improvement.

### Project Steps
- Data Collection: Gather and import data.
- Data Cleaning and Preprocessing: Handle missing values, duplicates, and standardize formats.
- Exploratory Data Analysis (EDA): Analyze sales trends, top products, customer segments, and visualize data.Y
- Advanced Analysis: Cohort analysis for customer retention.
- Insights and Recommendations: Summarize findings and provide actionable recommendations.
- Documentation and Reporting: Document the analysis process and results, create a comprehensive report.
- Project Deployment: Upload the project to GitHub with a detailed README file.

You can see the sql code 
[Uploading sales_analysis_s
select * from orders;

#Let's check for duplicates first
SELECT *
FROM (
SELECT *,
	ROW_NUMBER() OVER(PARTITION BY rowid, orderid, orderdate, customerid,segment, customername ) row_num
FROM Orders
)t WHERE row_num>1;
	
-- we don't have row number more than 1, so it means our data is uniqe and doesn't contain duplicates.
-- Next, let's take a look to our orderdate format

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

-- Total sales and profit for each province
SELECT
	province,
	productcategory,
        COUNT(*) OVER(PARTITION BY province),
	SUM(sales) OVER(PARTITION BY province) TotalSalesBYProvince,
	SUM(profit) OVER(PARTITION BY province) TotalProfitBYProvince
FROM Orders
ORDER BY TotalSalesBYProvince, TotalProfitBYProvince DESC;

-- Total number of all Orders and for each Product Name
SELECT
	Orderdate,
	productcategory,
	productname,
	COUNT(*) OVER() TotalOrders,
	COUNT(*) OVER(PARTITION BY productname) TotalOrdersByProductName
FROM Orders
ORDER BY TotalOrdersByProductName DESC;

-- Total number of orders for each customers
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

-- Total sales and profit by each customersegment
SELECT 
	customersegment,
	SUM(sales) OVER(PARTITION BY customersegment) TotalsalesBycustomersegment,
	SUM(profit) OVER(PARTITION BY customersegment) TotalProfitBycustomersegment
FROM Orders
ORDER BY TotalProfitBycustomersegment DESC
;

-- Total Sales by product category and customer segment
SELECT 
	productcategory,
	customersegment,
	SUM(sales) total_sales
FROM Orders
GROUP BY productcategory, customersegment
ORDER BY productcategory;

-- Total profit for each month
SELECT
	SUBSTRING(Orderdate, 1,7) Month, 
	SUM(profit) TotalProfit
FROM Orders
GROUP BY SUBSTRING(Orderdate, 1,7)
ORDER BY 1 ASC;

-- Percentage of contribution by each orders to the total sales
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

-- The deviation of each amount sales from minimum and maximum amount sales
SELECT
	productsubcategory,
	orderdate,
	MAX(sales) OVER() HighestSales,
	MIN(sales) OVER() LowestSales,
	sales - MIN(sales) OVER() DeviationFromMin,
	MAX(sales) OVER() - sales DeviationFromMax
FROM Orders;

-- Top 10 Orders based on their profit 
SELECT
	*,
	DENSE_RANK() OVER(ORDER BY profit DESC) ProfitRank_Dense
FROM Orders
LIMIT 10;

-- 10 Lowest Orders based on their profit 
SELECT
	*,
	DENSE_RANK() OVER(ORDER BY profit DESC) ProfitRank_Dense
FROM Orders
ORDER BY ProfitRank_Dense DESC
LIMIT 10;

-- Profit Ranking by year and product category
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

-- Products that fall within the highest 40% of the profit
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

-- Year-over-year profit performance between the current and previous year by finding the percentage change
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


-- Month-over-month sales and profit performance between the current and previous year by finding the percentage change
SELECT
*,
CurrentMonthSales - PreviousMonthSales AS MoM_Sales_Change,
ROUND((CurrentMonthSales - PreviousMonthSales)/PreviousMonthSales*100, 2) AS MoM_Sales_Percentage,
CurrentMonthProfit - PreviousMonthProfit AS MoM_Profit_Change,
ROUND((CurrentMonthProfit - PreviousMonthProfit)/PreviousMonthProfit*100, 2) AS MoM_Profit_Percentage
FROM (
SELECT
	SUBSTRING(Orderdate, 1,7) Month,
    SUM(sales) CurrentMonthSales,
    SUM(profit) CurrentMonthProfit,
    LAG(SUM(sales)) OVER (ORDER BY SUBSTRING(Orderdate, 1,7)) PreviousMonthSales,
    LAG(SUM(profit)) OVER (ORDER BY SUBSTRING(Orderdate, 1,7)) PreviousMonthProfit
FROM Orders
GROUP BY SUBSTRING(Orderdate, 1,7)
)t;


## Customer Retention
-- Customers return within the first 30 days
CREATE TEMPORARY TABLE customer_orders AS
SELECT
    customername,
    orderdate,
    DATE_FORMAT(orderdate, '%Y-%m') AS month,
    DATE_FORMAT(MIN(orderdate) OVER (PARTITION BY customername), '%Y-%m') AS cohort,
    DATEDIFF(orderdate, MIN(orderdate) OVER (PARTITION BY customername)) AS days_since_first_order
FROM orders;


SELECT
    cohort,
    COUNT(DISTINCT customername) AS num_returning_customers
FROM customer_orders
WHERE days_since_first_order > 0 AND days_since_first_order <= 30
GROUP BY cohort
ORDER BY cohort;

-- Calculating retention for customers who received a discount
SELECT
    cohort,
    month,
    COUNT(DISTINCT customername) AS num_customers_with_discount
FROM (
    SELECT
        customername,
        DATE_FORMAT(MIN(orderdate), '%Y-%m') AS cohort,
        DATE_FORMAT(orderdate, '%Y-%m') AS month,
        discount
    FROM orders
    GROUP BY customername, DATE_FORMAT(orderdate, '%Y-%m'), discount
) t
WHERE discount != 0
GROUP BY cohort, month
ORDER BY cohort, month;

-- Calculating retention for customers who doesn't received a discount
SELECT
    cohort,
    month,
    COUNT(DISTINCT customername) AS num_customers_without_discount
FROM (
    SELECT
        customername,
        DATE_FORMAT(MIN(orderdate), '%Y-%m') AS cohort,
        DATE_FORMAT(orderdate, '%Y-%m') AS month,
        discount
    FROM orders
    GROUP BY customername, DATE_FORMAT(orderdate, '%Y-%m'), discount
) t
WHERE discount = 0
GROUP BY cohort, month
ORDER BY cohort, month;


## Percentage Decline in Customer Retention
WITH
    discount_retention AS (
        SELECT
            cohort,
            month,
            COUNT(DISTINCT customername) AS num_customers_with_discount
        FROM (
            SELECT
                customername,
                DATE_FORMAT(MIN(orderdate), '%Y-%m') AS cohort,
                DATE_FORMAT(orderdate, '%Y-%m') AS month,
                discount
            FROM orders
            GROUP BY customername, DATE_FORMAT(orderdate, '%Y-%m'), discount
        ) t
        WHERE discount != 0
        GROUP BY cohort, month
    ),
    no_discount_retention AS (
        SELECT
            cohort,
            month,
            COUNT(DISTINCT customername) AS num_customers_without_discount
        FROM (
            SELECT
                customername,
                DATE_FORMAT(MIN(orderdate), '%Y-%m') AS cohort,
                DATE_FORMAT(orderdate, '%Y-%m') AS month,
                discount
            FROM orders
            GROUP BY customername, DATE_FORMAT(orderdate, '%Y-%m'), discount
        ) t
        WHERE discount = 0
        GROUP BY cohort, month
    )
SELECT
    d.cohort,
    d.month,
    d.num_customers_with_discount,
    n.num_customers_without_discount,
    IFNULL(n.num_customers_without_discount, 0) AS no_discount_count,
    IFNULL(d.num_customers_with_discount, 0) AS discount_count,
    CASE 
        WHEN d.num_customers_with_discount > 0 THEN
            ((d.num_customers_with_discount - IFNULL(n.num_customers_without_discount, 0)) / d.num_customers_with_discount) * 100
        ELSE
            NULL
    END AS percentage_decrease
FROM discount_retention d
LEFT JOIN no_discount_retention n
    ON d.cohort = n.cohort AND d.month = n.month
ORDER BY d.cohort, d.month;ql_queries.sql…]()

And interact with the sales dashboard [here](https://public.tableau.com/app/profile/rika3647/viz/E-commerceSalesPerforance/Dashboard1)

 
### Result/Findings

1. The company's sales and profit month-over-month performance is very unstable.
   The largest increase in sales was 100.72% and profits increased 121.32% which occurred between August 2010 to September 2010,
   while the largest sales loss, which decreased -48.08%, was in April 2009 to May 2009 and profits decreased -74.76%.
2.  Product Category **Tecnology** is the best performing category in terms of sales and profit with **Telephones and communication** is the highest subcategory sales
3.  Customer segment **corporate** should be targeted for marketing efforts
4.  There is an average **80%** retention rate difference between customers who use discounts and those who do not

### Recommendation

Based on the findings, here are some recommendations for stakeholders:

**1. Sales and Profit Stability**
   - Analyze the Causes of Fluctuations: Conduct a detailed analysis to understand the factors causing instability in month-over-month sales and profit performance. 
     Identify seasonal trends, market changes, or external factors contributing to the fluctuations.
   - Implement Stability Strategies: Develop strategies to reduce volatility, such as improved sales forecasting, product diversification, or more flexible pricing 
     adjustments.
   - Monitoring and Adjustment: Continuously monitor monthly performance and make periodic adjustments to strategies to ensure more stable results.

**2. Optimize Technology Product Category**
   - Focus on Key Category: Enhance marketing and sales efforts for the Technology category, particularly the Telephones and Communication subcategory, which shows 
     the best performance in terms of sales and profit.
   - Invest in Innovation: Invest in new product development and innovation within this category to maintain market position and meet evolving customer needs.
   - Expand Distribution Channels: Broaden distribution channels for technology products to reach a larger pool of potential customers.

**3. Target Corporate Customer Segment**
   - Personalized Marketing Strategies: Develop targeted marketing campaigns for the corporate customer segment, offering solutions and promotions tailored to their 
     needs.
   - Special Offers for Corporates: Create special offers or discounts for corporate customers to encourage repeat purchases and enhance loyalty.
   - Enhanced Customer Service: Provide better customer service and dedicated support for corporate clients to strengthen long-term relationships.

**4. Improve Customer Retention Through Discounts**
   - Optimize Discount Strategies: Utilize discount strategies to boost customer retention. Consider offering more frequent or attractive discounts to enhance appeal.
   - Segmented Discounts: Implement personalized discounts based on customer data to maximize impact and relevance.
   - Loyalty Programs: Develop loyalty programs that provide additional incentives for frequent buyers, whether they use discounts or not, to improve overall 
     retention.
     
By implementing these recommendations, the company can enhance performance stability, capitalize on key product categories, strengthen relationships with important customer segments, and improve overall customer retention.





