### --1. List all product categories currently being distributed. ###
```sql
SELECT CategoryID, CategoryName, Description FROM [dbo].[Categories]
```
![image](https://github.com/user-attachments/assets/691425fc-5b37-43eb-bf21-a340bd52bf00)
### --2. Kiểm tra số lượng sản phẩm theo danh mục ###
SELECT CategoryName, COUNT(ProductName) AS Number_of_Products
FROM [dbo].[Products] P
JOIN [dbo].[Categories] C
ON P.CategoryID = C.CategoryID
GROUP BY CategoryName
ORDER BY Number_of_Products 
```
![image](https://github.com/user-attachments/assets/4e6b5c9e-0b0a-4279-adc1-bdfa32ef6a68)
