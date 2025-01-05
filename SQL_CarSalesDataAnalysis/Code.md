Background:
<br>
This project involves analyzing the `stores.db` SQLite database, which contains sales records for scale model cars. The database includes eight tables: Customers, Employees, Offices, Orders, OrderDetails, Payments, Products, and ProductLines. These tables are interconnected, providing a comprehensive dataset for analysis.

Goals:
1. Perform an initial exploration of the database to understand its structure and contents.
2. Extract meaningful insights from the data to support decision-making processes.
3. Identify key metrics and trends within the sales records.
4. Generate reports that summarize the findings and provide actionable recommendations.

Conclusions:
1. The initial exploration revealed the structure and size of each table, providing a foundation for further analysis.
2. Key metrics such as total sales, customer demographics, and product performance were identified.
3. The analysis highlighted trends in sales performance across different regions and product lines.
4. The findings can be used to inform strategic decisions, such as inventory management, marketing strategies, and sales forecasting.


```
//*
This is SQL code that contains all the queries used
to analyze the stores.db SQLite database
*//

-- loaded the database via DB Browser for SQLite --

//* 
The database contains eight tables:

Customers: customer data
Employees: all employee information
Offices: sales office information
Orders: customers' sales orders
OrderDetails: sales order line for each sales order
Payments: customers' payment records
Products: a list of scale model cars
ProductLines: a list of product line categories
*//

-- Initial Exploration --

SELECT 'Customers' AS table_name, 
       (SELECT COUNT(*) FROM pragma_table_info('Customers')) AS num_attributes ,
       COUNT(*) AS num_rows
  FROM Customers 
UNION ALL
SELECT 'Employees', 
       (SELECT COUNT(*) FROM pragma_table_info('Employees')),
       COUNT(*)
  FROM Employees
UNION ALL
SELECT 'Offices', 
       (SELECT COUNT(*) FROM pragma_table_info('Offices')),
       COUNT(*)
  FROM Offices
UNION ALL
SELECT 'Orders', 
       (SELECT COUNT(*) FROM pragma_table_info('Orders')),
       COUNT(*)
  FROM Orders
UNION ALL
SELECT 'OrderDetails', 
       (SELECT COUNT(*) FROM pragma_table_info('OrderDetails')),
       COUNT(*)
  FROM OrderDetails
UNION ALL
SELECT 'Payments', 
       (SELECT COUNT(*) FROM pragma_table_info('Payments')),
       COUNT(*)
  FROM Payments
UNION ALL
SELECT 'Products', 
       (SELECT COUNT(*) FROM pragma_table_info('Products')),
       COUNT(*)
  FROM Products
UNION ALL
SELECT 'ProductLines', 
       (SELECT COUNT(*) FROM pragma_table_info('ProductLines')),
       COUNT(*)
  FROM ProductLines;

//*
we can answer the first question: which products should we order more of or less 
of? This question refers to inventory reports, including low stock(i.e. product in 
demand) and product performance. This will optimize the supply and the user 
experience by preventing the best-selling products from going out-of-stock.

The low stock represents the quantity of the sum of each product ordered divided
by the quantity of product in stock. We can consider the ten highest rates. 
These will be the top ten products that are almost out-of-stock or completely 
out-of-stock.

The product performance represents the sum of sales per product.

Priority products for restocking are those with high product performance that 
are on the brink of being out of stock.

We'll need the following two tables to perform these calculations: 
low stock = SUM(quantityOrdered)/quantityInStock
&
product performance = SUM(quantityOrdered * priceEach)
*// 

-- Query 1: Top 10 products that are almost out of stock or completely out of stock --
SELECT p.productCode, 
       p.productName, 
       p.quantityInStock, 
       SUM(od.quantityOrdered) AS total_ordered, 
       SUM(od.quantityOrdered) / p.quantityInStock AS low_stock
  FROM Products p
       JOIN OrderDetails od
       ON p.productCode = od.productCode
 GROUP BY p.productCode
 ORDER BY low_stock DESC
 LIMIT 10;

-- Query 2: Top 10 products with the highest sales performance --
SELECT p.productCode, 
       p.productName, 
       SUM(od.quantityOrdered) AS total_ordered, 
       SUM(od.quantityOrdered * od.priceEach) AS product_performance
  FROM Products p
       JOIN OrderDetails od
       ON p.productCode = od.productCode
 GROUP BY p.productCode
 ORDER BY product_performance DESC
 LIMIT 10;

 -- Combine the previous queries using a Common Table Expression (CTE) to display priority products for restocking using the IN operator. --
 SELECT p.productCode, 
       p.productName, 
       p.quantityInStock, 
       p.productLine,
       SUM(od.quantityOrdered) AS total_ordered, 
       SUM(od.quantityOrdered) / p.quantityInStock AS low_stock, 
       SUM(od.quantityOrdered * od.priceEach) AS product_performance
  FROM Products p
       JOIN OrderDetails od
       ON p.productCode = od.productCode
    GROUP BY p.productCode
    HAVING p.productCode IN (
        SELECT p.productCode
          FROM Products p
               JOIN OrderDetails od
               ON p.productCode = od.productCode
         GROUP BY p.productCode
         ORDER BY SUM(od.quantityOrdered * od.priceEach) DESC
         LIMIT 10
    )
     ORDER BY low_stock DESC
 LIMIT 10;

//*
The second question is: which product lines are performing well? This question
refers to the performance of product lines in terms of sales. This will help
the company to focus on the best-selling product lines and to optimize the
supply chain.
*//

-- Query 3: Top 10 product lines with the highest sales performance --
SELECT pl.productLine, 
       SUM(od.quantityOrdered) AS total_ordered, 
       SUM(od.quantityOrdered * od.priceEach) AS product_performance
  FROM ProductLines pl
       JOIN Products p
       ON pl.productLine = p.productLine
       JOIN OrderDetails od
       ON p.productCode = od.productCode
 GROUP BY pl.productLine
 ORDER BY product_performance DESC
 LIMIT 10;

//* 
Based on Question1 and Question2:
Vintage cars and motorcycles are the priority for restocking. 
They sell frequently, and they are the highest-performance products.
*//


//*
The third question is: which employees are performing well? This question refers
to the performance of employees in terms of sales. This will help the company to
focus on the best-performing employees and to optimize the sales team.
*//

-- Query 4: Top 10 employees with the highest sales performance --
SELECT e.employeeNumber, 
       e.lastName, 
       e.firstName, 
       SUM(od.quantityOrdered * od.priceEach) AS sales_performance
  FROM Employees e
       JOIN Orders o
       ON e.employeeNumber = o.employeeNumber
       JOIN OrderDetails od
       ON o.orderNumber = od.orderNumber
 GROUP BY e.employeeNumber
 ORDER BY sales_performance DESC
 LIMIT 10;

//*
Another question: how should we match marketing and communication strategies to 
customer behaviors? This involves categorizing customers: 
finding the VIP (very  important person) customers and those who are less engaged.

-VIP customers bring in the most profit for the store.
-Less-engaged customers bring in less profit.

For example, we could organize some events to drive loyalty for the VIPs and 
launch a campaign for the less engaged.
*//

-- Query 5: VIP customers --
SELECT c.customerNumber, 
       c.customerName, 
       SUM(od.quantityOrdered * od.priceEach) AS total_sales ,
       SUM(od.quantityOrdered * (od.priceEach-p.buyPrice)) AS profit
  FROM Customers c
        JOIN Orders o
        ON c.customerNumber = o.customerNumber
        JOIN OrderDetails od
        ON o.orderNumber = od.orderNumber
        JOIN Products p
        ON od.productCode = p.productCode
 GROUP BY c.customerNumber
 ORDER BY profit DESC
 LIMIT 10;

-- Find out the following for VIP companies: Last Name, First Name, City, and Country for the Contacts --
WITH VIP_customers AS (
    SELECT c.customerNumber, 
           c.customerName, 
           SUM(od.quantityOrdered * od.priceEach) AS total_sales ,
           SUM(od.quantityOrdered * (od.priceEach-p.buyPrice)) AS profit
      FROM Customers c
           JOIN Orders o
           ON c.customerNumber = o.customerNumber
           JOIN OrderDetails od
           ON o.orderNumber = od.orderNumber
           JOIN Products p
           ON od.productCode = p.productCode
     GROUP BY c.customerNumber
     ORDER BY profit DESC
     LIMIT 10
)
SELECT v.customerName,
       c.contactLastName, 
       c.contactFirstName, 
       c.city, 
       c.country, 
       v.profit
  FROM Customers c
       JOIN VIP_customers v
       ON c.customerNumber = v.customerNumber;


-- Query 6: Less-engaged customers and their contact details --
WITH NoVIP_customers AS (
    SELECT c.customerNumber, 
           c.customerName, 
           SUM(od.quantityOrdered * od.priceEach) AS total_sales ,
           SUM(od.quantityOrdered * (od.priceEach-p.buyPrice)) AS profit
      FROM Customers c
           JOIN Orders o
           ON c.customerNumber = o.customerNumber
           JOIN OrderDetails od
           ON o.orderNumber = od.orderNumber
           JOIN Products p
           ON od.productCode = p.productCode
     GROUP BY c.customerNumber
     ORDER BY profit ASC
     LIMIT 10
)
SELECT nv.customerName,
       c.contactLastName, 
       c.contactFirstName, 
       c.city, 
       c.country, 
       nv.profit
  FROM Customers c
       JOIN NoVIP_customers nv
       ON c.customerNumber = nv.customerNumber;

//* 
So how much can we spend on marketing and communication strategies?
I.e. how much can we spend on acquiring new customers and retaining existing ones?
*//

-- Query 7: The number of new customers arriving each month. --
SELECT strftime('%Y-%m', orderDate) AS month, 
       COUNT(DISTINCT customerNumber) AS new_customers
  FROM Orders
 GROUP BY month;

-- Query 8: compute the average amount of money a customer generates. --
SELECT AVG(total_sales) AS avg_sales_per_customer
  FROM (
    SELECT c.customerNumber, 
           c.customerName, 
           SUM(od.quantityOrdered * od.priceEach) AS total_sales ,
           SUM(od.quantityOrdered * (od.priceEach-p.buyPrice)) AS profit
      FROM Customers c
           JOIN Orders o
           ON c.customerNumber = o.customerNumber
           JOIN OrderDetails od
           ON o.orderNumber = od.orderNumber
           JOIN Products p
           ON od.productCode = p.productCode
     GROUP BY c.customerNumber
  );

```
