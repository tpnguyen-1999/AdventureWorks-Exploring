# [SQL] AdventureWorks Exploring
Utilizing Google BigQuery SQL to analyze data from multiple departments within a fictitious bicycle manufacturer (Adventure Works Cycles).

## 1. Introduction
The **adventureworks2019** dataset, stored in Google BigQuery, includes comprehensive data about the business from 2011 to 2014. Using this dataset, the author has developed exploratory questions for each department to be queried with SQL. The resulting data can then be used to create a strategic dashboard.

To interact with the dataset, the author utilizes Google BigQuery to write and execute SQL queries.

Dataset: **adventureworks2019** (public Google BigQuery dataset)

Dataset Dictionary: Please refer to the 'Data Dictionary' file attached above

## 2. Data Access
- Log in to your Google Cloud Platform account and create a new project.
- Navigate to the BigQuery console and select your newly created project.
- In the navigation panel, select "Add Data" and then "Star a project by name".
- Enter the project name "adventurework2019"

## 3. Sales Department
**3.1: Calculate Quantity of items, Sales value and Order quantity by each Subcategory in L12M**
- SQL code
~~~sql
SELECT format_datetime('%b %Y', a.ModifiedDate) month
      ,c.Name
      ,sum(a.OrderQty) qty_item
      ,sum(a.LineTotal) total_sales
      ,count(distinct a.SalesOrderID) order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` a 
LEFT JOIN `adventureworks2019.Production.Product` b
  ON a.ProductID = b.ProductID
LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c
  ON b.ProductSubcategoryID = cast(c.ProductSubcategoryID as string)  
WHERE date(a.ModifiedDate) >= date(2013,06,30)
GROUP BY 1,2
ORDER BY 2,1;
~~~
- Query results
  
|month| Name| qty_item| total_sales| order_cnt|
|-----|-----|---------|------------|----------|
|Apr 2014| Bib-Shorts| 4| 233.974| 1|
|Feb 2014| Bib-Shorts| 4| 233.972| 2|
|Jul 2013| Bib-Shorts| 2| 116.987| 1|
|Jun 2013| Bib-Shorts| 2| 116.987| 1|
|Apr 2014| Bike Racks| 45| 5400.0| 45|
|Aug 2013| Bike Racks| 22| 17387.184| 63|
|Dec 2013| Bike Racks| 162| 12582.288| 48|
|Feb 2014| Bike Racks| 27| 3240.0| 27|
|Jan 2014| Bike Racks| 161| 12840.0| 53|
|Jul 2013| Bike Racks| 422| 29802.3| 75|
|Jun 2013| Bike Racks| 363| 24684.0| 57|

**3.2: Calculate % YoY growth rate by SubCategory and release top 3 category with highest growth rate. Can use metric: quantity_item. Round results to 2 decimal**
- SQL code
~~~sql
WITH total_qty AS(
SELECT
  DISTINCT FORMAT_DATE('%Y', a.ModifiedDate) period,
  c.Name,
  SUM(OrderQty) qty_item,
FROM `adventureworks2019.Sales.SalesOrderDetail` a
LEFT JOIN `adventureworks2019.Production.Product` b on a.ProductID = b.ProductID
LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c on cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID
GROUP BY period, c.Name
ORDER BY c.Name desc, period)

,curr_prev_qty AS(
SELECT
  period,
  name,
  qty_item,
  LAG(qty_item) OVER(partition by name order by period) prv_qty
FROM total_qty
ORDER BY name desc, period)

,qty_difference AS(
SELECT
  period,
  name,
  qty_item,
  prv_qty,
  ROUND((qty_item/prv_qty -1),2) qty_diff
FROM curr_prev_qty)

,ranking_table AS(
SELECT
  name,
  qty_item,
  prv_qty,
  qty_diff,
  dense_rank() over(order by qty_diff desc) ranking
FROM qty_difference
WHERE qty_diff>0
ORDER BY ranking)

SELECT name, qty_item, prv_qty, qty_diff
FROM ranking_table
WHERE ranking <=3;
~~~
- Query results

|name| qty_item| prv_qty| qty_diff|
|----|---------|--------|---------|
|Mountain Frames| 3168| 510| 5.21|
|Socks| 2724| 523| 4.21|
|Road Frames| 5564| 1137| 3.89|

**3.3: Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number**
- SQL code
~~~sql
WITH total_order AS(
SELECT
  FORMAT_DATE('%Y',s1.ModifiedDate) year,
  s2.TerritoryID,
  sum(s1.OrderQty) order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` s1
LEFT JOIN `adventureworks2019.Sales.SalesOrderHeader` s2 ON s1.SalesOrderID = s2.SalesOrderID
GROUP BY year, s2.TerritoryID
ORDER BY year)

,ranking_table AS(
SELECT
  year,
  TerritoryID,
  order_cnt,
  dense_rank() over(partition by year order by order_cnt desc) ranking
FROM total_order)

SELECT *
FROM ranking_table
WHERE ranking <=3
ORDER BY year desc, ranking;
~~~
- Query results

|year| TerritoryID| order_cnt| ranking|
|----|------------|----------|--------|
|2014| 4| 11632| 1|
|2014| 6| 9711| 2|
|2014| 1| 8823| 3|
|2013| 4| 26682| 1|
|2013| 6| 22553| 2|
|2013| 1| 17452| 3|
|2012| 4| 17553| 1|
|2012| 6| 14412| 2|
|2012| 1| 8537| 3|
|2011| 4| 3238| 1|
|2011| 6| 2705| 2|
|2011| 1| 1964| 3|

**3.4: Calculate Total Discount Cost belongs to Seasonal Discount for each SubCategory**
- SQL code
~~~sql
SELECT 
    FORMAT_TIMESTAMP("%Y", ModifiedDate) year
    , Name
    , sum(disc_cost) as total_cost
FROM (
      SELECT a.ModifiedDate
      , c.Name
      , d.DiscountPct, d.Type
      , a.OrderQty * d.DiscountPct * UnitPrice as disc_cost 
      FROM `adventureworks2019.Sales.SalesOrderDetail` a
      LEFT JOIN `adventureworks2019.Production.Product` b on a.ProductID = b.ProductID
      LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c on cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID
      LEFT JOIN `adventureworks2019.Sales.SpecialOffer` d on a.SpecialOfferID = d.SpecialOfferID
      WHERE lower(d.Type) like '%seasonal discount%' 
)
GROUP BY 1,2;
~~~
- Query results

|year| Name| total_cost|
|----|-----|-----------|
|2012| Helmets| 827.647|
|2013| Helmets| 1606.041|

**3.5: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)**
- SQL code
~~~sql
WITH total_ord AS(
SELECT 
  extract(month from ModifiedDate) month,
  extract(year from ModifiedDate) year,
  CustomerID,
  count(SalesOrderID) order_cnt
FROM `adventureworks2019.Sales.SalesOrderHeader` 
WHERE extract(year from ModifiedDate) = 2014 
    AND Status = 5
GROUP BY 1,2,3)

, row_nb_table AS(
SELECT *
      , row_number() over(partition by CustomerID order by month) row_nb
FROM total_ord
ORDER BY CustomerID)

, first_month AS(
SELECT month as month_join
      , year
      , CustomerID
FROM row_nb_table
WHERE row_nb = 1)

, join_table AS(
SELECT 
  a.month,
  a.year,
  a.CustomerID,
  b.month_join,
  concat('M-',month - b.month_join) month_diff
FROM total_ord a
LEFT JOIN first_month b on a.CustomerID = b.CustomerID
ORDER BY CustomerID)

SELECT 
  month_join,
  month_diff,
  count(distinct CustomerID) customer_cnt
FROM join_table
GROUP BY 1,2
ORDER BY 1,2;
~~~
- Query results

|month_join| month_diff| customer_cnt|
|----------|-----------|-------------|
|1 |M-0| 2076|
|1 |M-1| 78|
|1 |M-2| 89|
|1 |M-3| 252|
|1 |M-4| 96|
|1 |M-5| 61|
|1 |M-6| 18|
|2 |M-0| 1805|
|2 |M-1| 51|
|2 |M-2| 61|
|2| M-3| 234|
|2| M-4| 58|
|2| M-5| 8|
|3| M-0| 1918|
|3| M-1| 43|

**3.6: Trend of Stock level & MoM diff % by all product in 2011. If % growth rate is null then 0. Round to 1 decimal**
- SQL code
~~~sql

~~~
- Query results

**3.7: Calculate Ratio of Stock / Sales in 2011 by product name, by month. Order results by month desc, ratio desc. Round Ratio to 1 decimal**
- SQL code
~~~sql

~~~
- Query results
