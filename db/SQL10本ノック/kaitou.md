## 問題 1 常連顧客の特定

SQL

```
SELECT Customers.CustomerID, count(Customers.CustomerID) as orderCount FROM [Orders]
join Customers on Orders.CustomerID = Customers.CustomerID
group by Customers.CustomerID
having orderCount >= 3
order by orderCount DESC
;
```

一番よく注文したのは ID20

## 最大の注文詳細

```
SELECT Orders.OrderID, count(OrderDetails.OrderID) as orderDetailCount FROM [OrderDetails]
join Orders on Orders.OrderID = OrderDetails.OrderID
group by OrderDetails.OrderID
order by orderDetailCount DESC
LIMIT 1
;
```

## Shipper の特定

```
SELECT ShipperID, count(ShipperID) as shipperCount FROM [Orders]
group by ShipperID
order by shipperCount DESC
LIMIT 1
;
```

## 売れてる country

```
SELECT Customers.Country, sum(OrderDetails.Quantity * Products.Price) as sales
FROM [OrderDetails]
join Products on Products.ProductID = OrderDetails.ProductID
join Orders on Orders.OrderID = OrderDetails.OrderID
join Customers on Customers.CustomerID = Orders.CustomerID
group by Customers.Country
order by sales DESC
;
```

## Country ごとの売り上げを年毎に集計

```
SELECT Customers.Country, strftime('%Y', Orders.OrderDate) as OrderYear, sum(OrderDetails.Quantity * Products.Price) as sales
FROM [OrderDetails]
join Products on Products.ProductID = OrderDetails.ProductID
join Orders on Orders.OrderID = OrderDetails.OrderID
join Customers on Customers.CustomerID = Orders.CustomerID
group by OrderYear, Customers.Country
order by Country
;
```

## Junior の追加

```
ALTER TABLE Employees
ADD Junior Boolean default false NOT NULL;
```

更新クエリ

```
update Employees
set Junior = true
where BirthDate >= '1961-01-01'
;
```

## long_relation の追加

```
ALTER TABLE Shipper
ADD long_relation Boolean default false NOT NULL;
```

更新クエリ

```
update Shipper
set long_relation = true
where ShipperId in (
  select ShipperID from Orders
  group by ShipperID
  having count(ShipperID) >= 70
)
```

## 最後に担当した Order

```
select Orders.EmployeeID, Orders2.LatestOrderDate from Orders
join (
	select EmployeeID, max(OrderDate) as LatestOrderDate
    from Orders
    group by EmployeeID
) as Orders2
on Orders2.EmployeeID = Orders.EmployeeID
where Orders.OrderDate = Orders2.LatestOrderDate
order by Orders.EmployeeID ASC
;
```

## NULL の扱いに慣れる

- 原則 NULL と比較演算をすると NULL になるため。`CustomerName IS NULL`と記述すれば OK

## 従業員の削除

### 削除

```
delete from Employees
where EmployeeID = 1
;
```

### 表示しない

inner join

```
select Orders.* from Orders
inner join Employees on Employees.EmployeeID = Orders.EmployeeID
;
```

### 表示する

left outer join

```
select Orders.* from Orders
left outer join Employees on Employees.EmployeeID = Orders.EmployeeID
;
```

# 課題 2

## WHERE と HAVING の違い

実行順序の違い
WHERE 句は GROUPBY の前に実行されるので、それ以降では適用できない
HAVING 句は GROUPBY の後に実行されるので、それ以降でも適用できる

SQL の実行順序

```
FROM → WHERE → GROUPBY → HAVING → SELECT → ORDERBY
```

## データ構造言語の違い

### DDL

Data Definition Language = データ定義言語
CREATE, DROP, ALTER などデータベースの生成や削除変更を行うコマンド

### DML

Data Manipulation Language = データ操作言語
SELECT、INSERT、UPDATE、DELETE などテーブルに対するデータの取得などを行うコマンド

### DCL

Data Control Language = データ制御言語
GRANT, REVOKE など DML や DDL の利用に関する許可や禁止を設定するコマンド

### TCL

Transaction Control Language = トランザクション制御言語
BEGIN, COMMIT, ROLLBACK などトランザクションの制御を行うコマンド

# 課題 3

-
