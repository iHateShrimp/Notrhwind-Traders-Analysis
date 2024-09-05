## Data Exploration: ##

> In the first part of the analysis, to gain some basic information about each table in a database, I do some simple queries such as the following:

### --1. List all product categories currently being distributed. ###
```sql
SELECT CategoryID, CategoryName, Description FROM [dbo].[Categories];
```
![image](https://github.com/user-attachments/assets/691425fc-5b37-43eb-bf21-a340bd52bf00)
### --2. Check the number of products by category ###
```sql
SELECT CategoryName, COUNT(ProductName) AS Number_of_Products
FROM [dbo].[Products] P
JOIN [dbo].[Categories] C
ON P.CategoryID = C.CategoryID
GROUP BY CategoryName
ORDER BY Number_of_Products;
```
![image](https://github.com/user-attachments/assets/4e6b5c9e-0b0a-4279-adc1-bdfa32ef6a68)

From the two queries above, we can see that there are 8 categories, along with a description table for each category. The category "Produce" has the lowest number of products, with 5 products, while "Confections" has the highest, with 13 products.

### --3.  Check the Total Quantity of Products Still Distributed and Discontinued: ###

**Method 1: Declare two variables, with X representing the number of products currently being distributed and Y representing those discontinued. Discontinued is defined as 0 for currently distributed and 1 for discontinued.**
```sql
DECLARE @X INT, @Y INT 
SELECT @X = (SELECT COUNT(ProductName) FROM [dbo].[Products]
				WHERE Discontinued = 0),
		@Y = (SELECT COUNT(ProductName) FROM [dbo].[Products]
				WHERE Discontinued = 1)
PRINT @X;
PRINT @Y;
```
**Method 2: Using PIVOT and CASE WHEN to Create Crosstab-Reports.**
```sql
SELECT 
		COUNT(CASE Discontinued WHEN 0 THEN 1 END) AS Active_Product,
		COUNT(CASE Discontinued WHEN 1 THEN 1  END) AS Discontinued_Product 
FROM [dbo].[Products]
```
![image](https://github.com/user-attachments/assets/1fc08f3a-9f9d-4e52-91ca-02f04cd68c5a)

Using DECLARE to assign variables and CASE WHEN in queries allows for easy classification of data based on specific conditions, improving reusability and simplifying complex calculations. This makes the query clearer, optimizes performance, and saves resources when processing data.

### --4. Total Quantity of Products Ordered: ###
```sql
SELECT COUNT(Quantity) AS Total_Ordered FROM [dbo].[Order Details]
```
![image](https://github.com/user-attachments/assets/3a2cf947-56f3-42b7-ac29-8b52b0e1eca9)

### --5. Check the number of products by supplier ###
```sql
SELECT CompanyName, COUNT(ProductName) AS Number_of_Products
FROM [dbo].[Suppliers] S
JOIN [dbo].[Products] P 
ON S.SupplierID = P.SupplierID
GROUP BY CompanyName
ORDER BY Number_of_Products DESC;
```

### --6. List the Total Number of Orders by Country in Descending Order: ###
```sql
SELECT ShipCountry, COUNT(*) AS Orders_of_each_Country
FROM [dbo].[Orders]
GROUP BY ShipCountry
ORDER BY Orders_of_each_Country DESC
```
**Conculsion:** Aiming to analyze product profitability, performing simple queries is crucial to gaining a preliminary understanding for more advanced queries. Northwind offers 8 product categories with a wide variety of products, including 69 active products and 8 discontinued ones. To date, 2,155 orders have been placed, with customers primarily located in North America, Europe, and some countries in South America. Among these, the USA and Germany lead the market with 122 orders each.

--Advanced Analyze to answer questions

--Tìm hiểu về doanh thu và khả năng gia tăng lợi nhuận của sản phẩm tính theo categories, theo thời gian trong từng quý, của từng năm:

--1. Top 10 sản phẩm mang lại doanh thu thuần cao nhất, xếp hạng theo doanh thu thuần?
/**Là doanh thu đã loại bỏ các chi phí chiết khấu từ tổng doanh thu, điều này sẽ biểu thị rõ hơn về doanh thu mà công ty thực sự nhận được**/
SELECT TOP 10 D.ProductID, ProductName, SUM(Quantity * (D.UnitPrice - (Discount * D.UnitPrice))) AS Net_Value,
RANK() OVER(ORDER BY SUM(Quantity * (D.UnitPrice - (Discount * D.UnitPrice))) DESC) AS Ranking
FROM [dbo].[Order Details] D
JOIN [dbo].[Products] P
ON D.ProductID = P.ProductID
GROUP BY ProductName, D.ProductID

/**2. Những sản phẩm nào bị ngừng kinh doanh và
doanh thu từ những sản phẩm đó là bao nhiêu trước khi ngừng kinh doanh?**/
SELECT D.ProductID, ProductName, SUM(Quantity * D.UnitPrice) AS Net_Value 
FROM [dbo].[Order Details] D
JOIN [dbo].[Products] P
ON D.ProductID = P.ProductID
WHERE Discontinued = 1
GROUP BY D.ProductID, ProductName
ORDER BY Net_Value DESC

--3. Xác định doanh thu mỗi quý, so sánh các quý theo năm và sự chênh lệch giữa các quý?
WITH QuarterRevenue AS (
					SELECT DATEPART(Year, O.OrderDate) AS Year, 
							DATEPART(Quarter, O.OrderDate) AS Quarter,
							SUM(D.UnitPrice * D.Quantity) AS Revenue 
							FROM [dbo].[Order Details] D

							JOIN [dbo].[Orders] O
							ON D.OrderID = O.OrderID

							GROUP BY DATEPART(Year, O.OrderDate), DATEPART(Quarter, O.OrderDate)
						)
SELECT *, LAG(Revenue) OVER(ORDER BY Year, Quarter) AS Previous_Revenue, Revenue - LAG(Revenue) OVER(ORDER BY Year, Quarter) AS Diff_Value FROM QuarterRevenue

--4. Liệt kê tổng doanh thu và doanh thu trung bình của từng category?
SELECT C.CategoryID, CategoryName, 
		SUM(D.UnitPrice * Quantity) AS Sum_of_Revenue,
		AVG(D.UnitPrice * Quantity) AS Avg_of_Revenue
FROM [dbo].[Order Details] D

JOIN [dbo].[Products] P
ON D.ProductID = P.ProductID

JOIN [dbo].[Categories] C
ON P.CategoryID = C.CategoryID

GROUP BY C.CategoryID, CategoryName

--Phân tích xếp hạng, giả sử trường hợp, các xu hướng tiêu dùng trong quá khứ để đưa ra các nhận định trong tương lai

--5. Tìm ra trong từng quý của năm, đâu là category được chọn nhiều nhất, điều này sẽ tìm hiểu xu hướng tiêu dùng theo mùa, đâu là loại sản phẩm trong mùa cần lượng hàng nhiều nhất để đáp ứng như cầu của thị trường 
WITH CategorySelected AS (
						SELECT C.CategoryID, C.CategoryName, SUM(D.Quantity) AS Total_Products,
							DATEPART(Quarter, O.OrderDate) As Quarter,
							DATEPART(Year, O.OrderDate) AS Year
						FROM [dbo].[Categories] C
					
						JOIN [dbo].[Products] P
						ON C.CategoryID = P.CategoryID

						JOIN [dbo].[Order Details] D
						ON P.ProductID = D.ProductID

						JOIN [dbo].[Orders] O
						ON D.OrderID = O.OrderID

						GROUP BY C.CategoryID, C.CategoryName, DATEPART(Quarter, O.OrderDate), DATEPART(Year, O.OrderDate)
						),
RankedCategory AS (
				SELECT *, RANK() OVER (PARTITION BY Year, Quarter ORDER BY Total_Products DESC) AS Ranked
				FROM CategorySelected
				)
----------
SELECT * FROM RankedCategory
WHERE Ranked = 1 

--6. Tìm ra top 5 sản phẩm của từng category có doanh thu cao nhất trong từng quý qua từng năm?
WITH ProductRevenue AS (
					SELECT C.CategoryID, C.CategoryName, P.ProductName,
						DATEPART(Quarter, O.OrderDate) AS Quarter,
						DATEPART(Year, O.OrderDate) AS Year,
						SUM( D.UnitPrice * Quantity) AS Revenue
					FROM [dbo].[Order Details] D

					JOIN [dbo].[Products] P 
						ON D.ProductID = P.ProductID

					JOIN [dbo].[Categories] C
						ON P.CategoryID = C.CategoryID

					JOIN [dbo].[Orders] O 
						ON D.OrderID = O.OrderID
					GROUP BY C.CategoryID, C.CategoryName, P.ProductName, DATEPART(Quarter, O.OrderDate), DATEPART(Year, O.OrderDate)
						),
RankedProducts AS (
				SELECT CategoryID, CategoryName, ProductName,
				Quarter, Year, Revenue,
				RANK() OVER(PARTITION BY  CategoryID, Year, Quarter ORDER BY Revenue DESC) AS RANK 
				FROM ProductRevenue
					)

SELECT * FROM  RankedProducts
WHERE RANK <= 5

--7. Tìm ra 3 sản phẩm được đặt hàng nhiều nhất cho mỗi category cùng với thông tin về nhà cung cấp của sản phẩm đó?
WITH ProductRankings AS ( 
					SELECT P.ProductID, P.ProductName,P.CategoryID, P.SupplierID, S.CompanyName, SUM(D.Quantity) AS Total_Orders,
							RANK() OVER(PARTITION BY P.CategoryID ORDER BY SUM(D.Quantity) DESC) AS ProductRank
					FROM [dbo].[Products] P

					JOIN [dbo].[Order Details] D
						ON P.ProductID = D.ProductID

					JOIN [dbo].[Orders] O
						ON D.OrderID = O.OrderID

					JOIN [dbo].[Suppliers] S
						ON P.SupplierID = S.SupplierID

					GROUP BY P.ProductID, P.ProductName,P.CategoryID, P.SupplierID, S.CompanyName
					)
							
SELECT * FROM ProductRankings
WHERE ProductRank <= 3

--8. Để theo dõi dễ dàng hơn về tình trạng hàng hóa, chúng ta cần lập một bảng để có thể cập nhật tình hình mỗi ngày về lượng hàng vào và ra**/
/** Cần dùng VIEW để đáp ứng việc cập nhật dữ liệu liên tục, CASE WHEN để viết điều kiện,
và LOẠI BỎ các sản phẩm có Discontinued = 1 (Sản phẩm đã dừng hoạt động)**/
--Tạo VIEW với Status của sản phẩm là Out of Stock
CREATE VIEW OutofStockStatus AS
SELECT * 
FROM (
    SELECT *, 
           CASE 
               WHEN UnitsInStock <= ReorderLevel AND UnitsOnOrder = 0 
                   THEN 'Out of Stock'
               ELSE 'Normal stock level'
           END AS Status 
    FROM [dbo].[Products]
    WHERE Discontinued = 0
) AS ProductView --Cần phải có tên cho bảng được tạo từ câu truy vấn con 
WHERE Status = 'Out of Stock';
--Chạy VIEW 
SELECT * FROM OutofStockStatus

/**9.1. Giả sử, ta cần xác định doanh thu của các sản phẩm bán chạy và không bán chạy qua từng quý của năm bằng cách chia tập dữ liệu ra làm 4 mức:
1 là mức doanh thu cao nhất và giảm dần, chúng ta sẽ so sánh nhóm bán chạy nhất và nhóm bán chạy nhất, khác biệt về doanh thu**/
WITH QuarterlyProductSales AS (
						SELECT C.CategoryID, C.CategoryName, P.ProductName,
							DATEPART(Quarter, O.OrderDate) AS Quarter,
							DATEPART(Year, O.OrderDate) AS Year,
							SUM( D.UnitPrice * Quantity) AS Revenue
						FROM [dbo].[Order Details] D

						JOIN [dbo].[Products] P 
						ON D.ProductID = P.ProductID

						JOIN [dbo].[Categories] C
						ON P.CategoryID = C.CategoryID

						JOIN [dbo].[Orders] O 
						ON D.OrderID = O.OrderID
						GROUP BY C.CategoryID, C.CategoryName, P.ProductName, DATEPART(Quarter, O.OrderDate), DATEPART(Year, O.OrderDate)
					),
				
Ranking AS (
		SELECT *, NTILE(4) OVER (ORDER BY REVENUE DESC) AS Rank
		FROM QuarterlyProductSales
			),

SalesComparision AS (
				SELECT Year, Quarter, 
				SUM( CASE WHEN Rank = 1 THEN Revenue ELSE 0 END) AS Top_Sales_Revenue,
				SUM( CASE WHEN Rank = 4 THEN Revenue ELSE 0 END) AS Bottom_Sales_Revenue
				FROM Ranking
				GROUP BY Year, Quarter
				)
----------
SELECT *, Top_Sales_Revenue - Bottom_Sales_Revenue AS Revenue_Difference
FROM SalesComparision
ORDER BY Year, Quarter

--9.2. Tương tự như vậy, thay vì hiển thị ra sự chênh lệch doanh thu, hiển thị ra tến sản phẩm bán chạy nhất qua từng quý của năm để có cái nhìn rõ hơn về các sản phẩm đó
WITH QuarterlyProduct AS (
						SELECT P.ProductID, P.ProductName, C.CategoryID, P.SupplierID,
							DATEPART(Quarter, O.OrderDate) AS Quarter,
							DATEPART(Year, O.OrderDate) AS Year,
							SUM( D.UnitPrice * Quantity) AS Revenue
						FROM [dbo].[Order Details] D

						JOIN [dbo].[Products] P 
						ON D.ProductID = P.ProductID

						JOIN [dbo].[Categories] C
						ON P.CategoryID = C.CategoryID

						JOIN [dbo].[Orders] O 
						ON D.OrderID = O.OrderID
						GROUP BY P.ProductID, P.ProductName, C.CategoryID, P.SupplierID, DATEPART(Quarter, O.OrderDate), DATEPART(Year, O.OrderDate)
						),
RankedProducts AS (
				SELECT *,
					NTILE(4) OVER(ORDER BY Revenue DESC) AS Ranked
					FROM QuarterlyProduct
					)
----------
SELECT ProductID, ProductName, CategoryID, SupplierID, Revenue, Year, Quarter
FROM RankedProducts 
WHERE Ranked = 1
ORDER BY  Year, Quarter

--9.3. Trong số 25% kể trên, hãy cho biết sản phẩm thuộc loại nào chiếm phần cao nhất 
WITH QuarterlyProduct AS (
						SELECT P.ProductID, P.ProductName, C.CategoryID, C.CategoryName, P.SupplierID,
							SUM( D.UnitPrice * Quantity) AS Revenue
						FROM [dbo].[Order Details] D

						JOIN [dbo].[Products] P 
						ON D.ProductID = P.ProductID

						JOIN [dbo].[Categories] C
						ON P.CategoryID = C.CategoryID

						JOIN [dbo].[Orders] O 
						ON D.OrderID = O.OrderID
						GROUP BY P.ProductID, P.ProductName, C.CategoryID,C.CategoryName, P.SupplierID
						),
RankedProducts AS (
				SELECT *,
					NTILE(4) OVER(ORDER BY Revenue DESC) AS Ranked
					FROM QuarterlyProduct
					)
----------
SELECT CategoryID, CategoryName, COUNT(CategoryID) Product_of_each_Category
FROM RankedProducts 
WHERE Ranked = 1 
GROUP BY CategoryID, CategoryName
ORDER BY COUNT(CategoryID) DESC

/**10. Giả sử, khách hàng có giá trị đơn hàng cao hơn 75% giá trị đơn hàng của các khách hàng khác thì được xem là High-Value Customers,
và ngược lại, ta cần in ra danh sách các customers có giá trị đơn hàng cao, gồm ID, tên, số lượng đơn đặt hàng, tổng giá trị đơn hàng**/
WITH TotalValues AS (
				SELECT C.CustomerID, C.CompanyName, C.ContactName, 
				SUM(D.UnitPrice * D.Quantity) AS Total_order_Value
				FROM [dbo].[Customers] C

				JOIN [dbo].[Orders] O
				ON C.CustomerID = O.CustomerID

				JOIN [dbo].[Order Details] D
				ON O.OrderID = D.OrderID

				GROUP BY C.CustomerID, C.CompanyName, C.ContactName
					),
HighValues AS (
			SELECT CustomerID, CompanyName, ContactName, Total_order_Value,
			PERCENT_RANK() OVER (ORDER BY Total_order_Value) AS Percentrank
			FROM TotalValues
				)
-------
SELECT CustomerID, CompanyName, ContactName, Total_order_Value, Percentrank,
		CASE 
			WHEN Percentrank > 0.75 THEN 'High Value Customer'
			ELSE 'Regular Customer'
		END AS Customer_Type
FROM  HighValues
WHERE Percentrank > 0.75
