# SQL_AussieRetailers
Understanding of warehousing and data mining fundamentals by using advanced SQL techniques 

A retail chain, "AussieRetailers," operates across multiple states in Australia. The company prides itself on delivering exceptional customer service, maintaining a diverse product range, and staying attuned to market demands. The company has been focusing on optimizing its operations to maximize profitability while maintaining high customer satisfaction.

The database has the following table overview: 
Sales: Records each sale, including sale ID, date, product ID, quantity, customer ID, and branch ID.
Customer Details: Stores customer information, including customer ID, name, contact details, and loyalty program status.
Product Details: Contains product information, such as product ID, name, category, price, and stock levels.
Staff Shift Details: Logs staff shifts, including staff ID, branch ID, shift date, start time, and end time.
Complaint Details: Records customer complaints, with complaint ID, customer ID, product ID, complaint date, and resolution status


MySQL transcript
List of products and their categories with sales greater than 100 units.
SELECT pd.product_id, pd.name AS product_name, pd.category, SUM(s.quantity) AS total_sales FROM productdetails pd JOIN sales s ON pd.product_id = s.product_id GROUP BY pd.product_id, pd.name, pd.category HAVING SUM(s.quantity) > 100;

•	The total revenue per branch
SELECT s.branch_id, SUM(s.quantity * pd.price) AS total_revenue FROM sales s JOIN productdetails pd ON s.product_id = pd.product_id GROUP BY s.branch_id;
•	Customers who purchase in more than 3 branches

SELECT customer_id FROM sales GROUP BY customer_id HAVING COUNT(DISTINCT branch_id) > 3;
•	The average sale quantity by category for sales made in the last quarter

SELECT pd.category AS product_category, AVG(s.quantity) AS avg_sale_quantity FROM sales s JOIN productdetails pd ON s.product_id = pd.product_id WHERE s.sale_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH) AND s.sale_date <= NOW() GROUP BY pd.category;

•	Ranking products based on the total sales quantity

SELECT product_id, name AS product_name, category AS product_category, total_sales, RANK() OVER (PARTITION BY category ORDER BY total_sales DESC) AS sales_rank FROM ( SELECT pd.product_id, pd.name, pd.category, COALESCE(SUM(s.quantity), 0) AS total_sales FROM productdetails pd LEFT JOIN sales s ON pd.product_id = s.product_id GROUP BY pd.product_id, pd.name, pd.category ) AS sales_summary;

•	The month-over-month percentage growth in sales for the top 5 products 

SELECT product_id, product_name, month_year, total_sales, LAG(total_sales) OVER (PARTITION BY product_id ORDER BY month_year) AS prev_month_sales, CASE WHEN LAG(total_sales) OVER (PARTITION BY product_id ORDER BY month_year) IS NOT NULL THEN (total_sales - LAG(total_sales) OVER (PARTITION BY product_id ORDER BY month_year)) / LAG(total_sales) OVER (PARTITION BY product_id ORDER BY month_year) * 100 ELSE NULL END AS sales_growth_percentage FROM ( SELECT s.product_id, pd.name AS product_name, DATE_FORMAT(s.sale_date, '%Y-%m') AS month_year, SUM(s.quantity) AS total_sales FROM sales s JOIN productdetails pd ON s.product_id = pd.product_id GROUP BY s.product_id, product_name, month_year ) AS monthly_sales ORDER BY sales_growth_percentage DESC LIMIT 5;


•	Products not sold in the last 6 months but have stock levels above 50

SELECT pd.product_id, pd.name AS product_name, pd.stock_level FROM productdetails pd LEFT JOIN sales s ON pd.product_id = s.product_id AND s.sale_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH) WHERE s.product_id IS NULL AND pd.stock_level > 50;

•	The total number of complaints lodged against per Product Category
SELECT pd.category AS product_category, COUNT(*) AS total_complaints FROM complaintdetails cd JOIN productdetails pd ON cd.product_id = pd.product_id GROUP BY pd.category;

•	The top 10 customers by total spending 
SELECT cd.customer_id, cd.name AS customer_name, SUM(s.quantity * pd.price) AS total_spending, ( SELECT pd.category FROM productdetails pd WHERE pd.product_id = s.product_id ORDER BY COUNT(*) DESC LIMIT 1 ) AS most_frequent_category FROM sales s JOIN customerdetails cd ON s.customer_id = cd.customer_id JOIN productdetails pd ON s.product_id = pd.product_id GROUP BY cd.customer_id, cd.name ORDER BY total_spending DESC LIMIT 10;
•	Days of the week with the highest sales transactions volume5
SELECT DAYNAME(sale_date) AS day_of_week, COUNT(*) AS transaction_volume FROM sales GROUP BY DAYOFWEEK(sale_date) ORDER BY transaction_volume DESC;

•	The correlation between loyalty program status and the average transaction value per customer
SELECT cd.loyalty_program_status, AVG(s.total_transaction_value) AS avg_transaction_value FROM customerdetails cd JOIN ( SELECT customer_id, SUM(quantity * price) AS total_transaction_value FROM sales s JOIN productdetails pd ON s.product_id = pd.product_id GROUP BY customer_id ) AS s ON cd.customer_id = s.customer_id GROUP BY cd.loyalty_program_status;

•	The average duration between complaint registration and resolution

SELECT AVG(DATEDIFF(CURRENT_DATE, complaint_date)) AS avg_duration_days FROM complaintdetails WHERE resolution_status = 'Closed';

•	Staff members with shifts longer than 8 hours 

SELECT s.staff_id, s.branch_id, s.shift_date, TIMEDIFF(s.end_time, s.start_time) AS shift_duration FROM staffshiftdetails s HAVING shift_duration > '08:00:00';

•	The branch with the lowest stock levels across all products

SELECT branch_id, SUM(stock_level) AS total_stock FROM productdetails JOIN sales ON productdetails.product_id = sales.product_id GROUP BY branch_id ORDER BY total_stock ASC LIMIT 1;

•	The product with the highest number of complaints and nature of complaints 
SELECT cd.product_id, pd.name AS product_name, COUNT(*) AS total_complaints, GROUP_CONCAT(cd.resolution_status ORDER BY cd.complaint_id) AS complaints FROM complaintdetails cd JOIN productdetails pd ON cd.product_id = pd.product_id GROUP BY cd.product_id, pd.name ORDER BY total_complaints DESC LIMIT 1;
Self-designed questions 
SELECT s.branch_id, b.branch_name, SUM(s.quantity) AS total_sales, SEC_TO_TIME(SUM(TIME_TO_SEC(TIMEDIFF(ss.end_time, ss.start_time)))) AS total_hours_worked, SUM(s.quantity) / TIME_TO_SEC(TIMEDIFF(ss.end_time, ss.start_time)) AS sales_per_hour FROM sales s JOIN staffshiftdetails ss ON s.branch_id = ss.branch_id JOIN branchdetails b ON s.branch_id = b.branch_id GROUP BY s.branch_id, b.branch_name ORDER BY sales_per_hour DESC;

How do sales vary by season for each product category? Calculate the total sales for each category in each season (e.g., summer, winter, spring, autumn).
SELECT pd.category AS product_category, CASE WHEN MONTH(sale_date) IN (12, 1, 2) THEN 'Winter' WHEN MONTH(sale_date) IN (3, 4, 5) THEN 'Spring' WHEN MONTH(sale_date) IN (6, 7, 8) THEN 'Summer' WHEN MONTH(sale_date) IN (9, 10, 11) THEN 'Autumn' ELSE 'Unknown' END AS season, SUM(s.quantity) AS total_sales FROM sales s JOIN productdetails pd ON s.product_id = pd.product_id GROUP BY pd.category, season ORDER BY pd.category, season;

What is the average amount spent per transaction?

SELECT AVG(s.quantity * pd.price) AS average_transaction_value FROM sales s JOIN productdetails pd ON s.product_id = pd.product_id;

How many complaints are currently pending resolution?
SELECT COUNT(*) AS num_pending_complaints FROM complaintdetails WHERE resolution_status = 'Pending';

What are the products with the highest and lowest prices?
SELECT MAX(price) AS highest_price_product, MIN(price) AS lowest_price_product FROM productdetails;


