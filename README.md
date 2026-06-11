# E-Commerce Sales Data Cleaning — SQL Server Project
## Project overview:
This project focuses on cleaning and preparing a raw e-commerce sales dataset using SQL Server (SSMS). The dataset was sourced from Kaggle and contains 103 rows of transactional data including customer information, product details, pricing, and order status.

The goal of this project is to transform a messy, inconsistent dataset into a clean, analysis-ready table by applying a series of structured data cleaning steps.

## Dataset Description:
the datsset contains 103 rows of sales records.

### Data Columns:
ID, Customer_Name, Order_ID, Order_Date, Product, Category, Quantity, Price, Payment_Method, Status, Total

## Tools used:
Microsoft SQL Server (SSMS)

## Problems Found in Raw Data:
1. Duplicate rows and duplicate Order_ID values.
2. Inconsistent Category casing ('electronics', 'ELECTRONICS', 'electronic').
3. Inconsistent Category for each product.
4. Blank strings instead of proper NULLs.
5. Price stored as VARCHAR with corrupt values ('abd', 'four hundred', '300$').
6. Missing Price for many products.
7. Negative values in Quantity and Price.
8. Total column inconsistent with Quantity * Price
9. Order_Date stored as text instead of DATE type

## SQL Concepts Practiced
1. ALTER TABLE / ALTER COLUMN.
2. UPDATE with ABS(), TRIM(), UPPER(), LOWER().
3. TRY_CAST.
4. ROW_NUMBER() for deduplication.
5. PARTITION BY with window functions.
6. CTEs (WITH clause).
7. NULL handling.
8. CASE statments.
9. Joins. 



## Cleaning Steps Performed
### 1. Remove duplicates rows
#### 1.1 create another table by the same data of the existing table and add a row_num column to distinguish the duplicates row where row_num > 2 

```sql
CREATE TABLE [dbo].[ecommerce_sales_data_new](
	[ID] [varchar](50) NULL,
	[ Customer_Name] [varchar](50) NULL,
	[Order_ID] [varchar](50) NULL,
	[Order_Date] [varchar](50) NULL,
	[Product] [varchar](50) NULL,
	[ Category] [varchar](50) NULL,
	[Quantity] [varchar](50) NULL,
	[Price] [varchar](50) NULL,
	[Payment_Method] [varchar](50) NULL,
	[Status] [varchar](50) NULL,
	[Total] [varchar](50) NULL,
	[row_num] [INT]
) ON [PRIMARY]
GO
;
``` 
#### 1.2 insert the same data into the new column in addition to row_num by partition by all columns to identify the row with all duplicate values.  this row is ID = 146
```sql
INSERT INTO ecommerce_sales_data_new
SELECT 
	*,
	ROW_NUMBER() 
	OVER(PARTITION BY ID, [ Customer_Name], Order_ID, Order_Date, Product, [ Category], Quantity, Price, Payment_Method, Status, Total ORDER BY ID) AS row_num   
FROM ecommerce_sales_data
;
```
#### 1.3 delete the rows whee the row num > 1 
```sql
DELETE FROM ecommerce_sales_data_new
WHERE row_num > 1
;
```


### 2. Removal of duplicate values in Order_id column 
#### 2.1 create a new table from the existing table to delete the duplicates
```sql
CREATE TABLE [dbo].[ecommerce_sales_data_new_new](
	[ID] [varchar](50) NULL,
	[ Customer_Name] [varchar](50) NULL,
	[Order_ID] [varchar](50) NULL,
	[Order_Date] [varchar](50) NULL,
	[Product] [varchar](50) NULL,
	[ Category] [varchar](50) NULL,
	[Quantity] [varchar](50) NULL,
	[Price] [varchar](50) NULL,
	[Payment_Method] [varchar](50) NULL,
	[Status] [varchar](50) NULL,
	[Total] [varchar](50) NULL,
	[row_num] [INT],
	[row_order_id] [INT]
) ON [PRIMARY]
GO
;
```
#### 2.2 insert the same data into the new column in addition to row_num by partition by Order_ID to identify the row the duplicate values.  these rows ID = 142 and ID = 175
```sql
INSERT INTO ecommerce_sales_data_new_new
SELECT 
	*,
	ROW_NUMBER() OVER(PARTITION BY Order_ID ORDER BY Order_ID) AS row_order_id
FROM ecommerce_sales_data_new
;
```

#### 2.3 delete the rows where the ID = 142 and ID = 175
```sql
DELETE FROM ecommerce_sales_data_new_new
WHERE (ID = '175' AND row_order_id = 2)
OR 
(ID = '142' AND row_order_id = 1)
;
```

### 3. Category standaraziation 
#### 3.1 Make sure the column category has no trailing spaces
```sql
UPDATE ecommerce_sales_data_new_new
SET [ Category] = TRIM([ Category])
;
```
#### 3.2 Update the category column in the table for the products with more than one category and one of them is wrong such as basketball, yoga mat, t-shirt
```sql
UPDATE ecommerce_sales_data_new_new
SET [ Category] = 'Sports'
WHERE [Product] = 'Basketball'
;

UPDATE ecommerce_sales_data_new_new
SET [ Category] = 'Sports'
WHERE [Product] = 'Yoga Mat'
;

UPDATE ecommerce_sales_data_new_new
SET [ Category] = 'Clothing'
WHERE Product = 'T-shirt' 
;
```
#### 3.3 Convert the balnk values and 'nan' text to null to ease the join on one value
```sql
UPDATE ecommerce_sales_data_new_new
SET [ Category] = NULL
WHERE [ Category] = 'nan'OR [ Category] =''
;
```
#### 3.4 Then join based on the common values of product name
```sql
UPDATE e1
SET e1.[ Category] = e2.[ Category]
FROM ecommerce_sales_data_new_new AS e1
JOIN ecommerce_sales_data_new_new AS e2
ON e1.Product = e2.Product
WHERE 
e1.[ Category] IS NULL AND e2.[ Category] IS NOT NULL 
;
```
#### 3.5 Make a table for the count of each time each product with each category appears for products with more than two categories (blednder, lamp, microwave, vacuum) and then seelct the category where it appears mostly then Join with the original table and set the values right
```sql
WITH cte AS
(
SELECT 

	[ Category],
	Product
FROM
(
	SELECT 
		 [ Category], Product, COUNT(*) AS num,
		 ROW_NUMBER() OVER (PARTITION BY Product ORDER BY COUNT(*) DESC) AS row
	FROM ecommerce_sales_data_new_new
	WHERE Product ='Blender' OR Product ='Lamp' OR Product ='Microwave' OR Product ='Vacuum'
	GROUP BY [ Category], Product
) AS new_table 
WHERE row = 1
	)

UPDATE e1
SET e1.[ Category] = e2.[ Category] 
FROM  ecommerce_sales_data_new_new AS e1
LEFT JOIN cte AS e2 
ON e1.Product = e2.Product
WHERE e1.Product ='Blender' OR e1.Product ='Lamp' OR e1.Product ='Microwave' OR e1.Product ='Vacuum'
;
```
### 4. filling the missing values and correcting values in the price column
#### 4.1 Convert blank values to null 
```sql
UPDATE ecommerce_sales_data_new_new
SET Price = NULL 
WHERE Price = ''
;
```
#### 4.2 Chnage the data rows where the values can not be converted to decimal
```sql
UPDATE ecommerce_sales_data_new_new
SET Price = 300 
WHERE Price = '300$'
;

UPDATE ecommerce_sales_data_new_new
SET Price = NULL 
WHERE Price = 'abd'
;

UPDATE ecommerce_sales_data_new_new
SET Price = 400
WHERE Price = 'four hundred '
;
```
#### 4.3 change the data type from text to decimal
```sql
ALTER TABLE ecommerce_sales_data_new_new
ALTER COLUMN Price DECIMAL(10, 2)
;
```
#### 4.4 change from negative to positive 
```sql
UPDATE ecommerce_sales_data_new_new
SET Price = ABS(Price)
WHERE Price < 0
;
```
#### 4.5 make the average as the data wanted for the missing value
```sql
UPDATE ecommerce_sales_data_new_new
SET Price = (
    SELECT AVG(Price)
    FROM ecommerce_sales_data_new_new AS e1
    WHERE e1.Product = ecommerce_sales_data_new_new.Product
      AND e1.Price IS NOT NULL
)
WHERE Price IS NULL;
;
```

### 5. Correcting the quantity column
#### 5.1 Change the only text and number value in quantity column '4a' to number only '4'
```sql
UPDATE ecommerce_sales_data_new_new
SET Quantity = 4
WHERE Quantity ='4a'
;
```
#### 5.2 Delete rows where quantity is blank
```sql
DELETE FROM ecommerce_sales_data_new_new
WHERE Quantity =''
;
```
#### 5.3 Change the data type to INT
```sql
ALTER TABLE ecommerce_sales_data_new_new
ALTER COLUMN Quantity INT
;
```
#### 5.4 drop the rows where quantity is null
```sql
DELETE FROM ecommerce_sales_data_new_new
WHERE Quantity IS NULL
;
```
#### 5.5 Change the negative values to positive in quantity 
```sql
UPDATE ecommerce_sales_data_new_new
SET Quantity = ABS(Quantity)
WHERE Quantity < 0
;
```

### 6. Correcting total values 
#### 6.1 Change the blank into null
```sql
UPDATE ecommerce_sales_data_new_new
SET Total = NULL 
WHERE Total = ''
;
```
#### 6.2 Change the data type to decimal 
```sql
ALTER TABLE ecommerce_sales_data_new_new
ALTER COLUMN Total DECIMAL(10,2)
;
```
#### 6.3 Check whther it is right or wrong
```sql
WITH cte AS 
(
SELECT 
	*,
	(Quantity * Price) AS total_valid
FROM ecommerce_sales_data_new_new
)

SELECT
	*,
	CASE WHEN Total = total_valid THEN 'Right' ELSE 'Wrong' END AS right_wrong
FROM cte 
;
```
#### 6.4 Delete the total column since it contains many wrong data 
```sql
ALTER TABLE ecommerce_sales_data_new_new
DROP COLUMN Total
;
```
#### 6.5 Create new column by multiplying the quantity by the price 
```sql
ALTER TABLE ecommerce_sales_data_new_new
ADD Total AS (Quantity * Price)
;
```

### 7. Category standaraization 
```sql
UPDATE ecommerce_sales_data_new_new
SET [ Category] = UPPER(LEFT([ Category], 1)) + LOWER(SUBSTRING([ Category], 2, LEN([ Category])))
;
```

### 8. Change the date format in the order_date
```sql
ALTER TABLE ecommerce_sales_data_new_new
ADD Order_Date_updated DATE;

UPDATE ecommerce_sales_data_new_new
SET Order_Date_updated = TRY_CAST(Order_Date AS DATE);

ALTER TABLE ecommerce_sales_data_new_new
DROP COLUMN Order_Date;
```

### 9. Text data triming and droping the columns row_num
```sql
SET 
[ Customer_Name] = TRIM([ Customer_Name]),
[Order_ID] = TRIM([Order_ID]),
[Product] = TRIM([Product]),
[Payment_Method] = TRIM([Payment_Method]),
[Status] = TRIM([Status])
;

ALTER TABLE ecommerce_sales_data_new_new
DROP COLUMN row_num
;

ALTER TABLE ecommerce_sales_data_new_new
DROP COLUMN row_order_id
;
```






