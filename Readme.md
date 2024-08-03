# Updated Requirements Documentation

1. **Overview of updated ERD:**

**New Entities:**

- Instalment
- Branch
- Shareholder
- Coupon

**New Relationships:**

- Instalment - Sales_operation : **One to many**, One sales operation could have many instalments or none.
- Coupon - Sale_operation : **One to one**, one sale operation may have only one coupon or none.
- Branch - Salesperson : **One to many**, A branch may have many Salespersons.
- Branch - Car : **One to many**, Each branch may have many cars.

1. **Overview of updated schema:**

 Sales_operation:

- Added **Instalment Plan** with default value of **Fales**.

Sales_person:

- Added **Branch_ID** with **FK** constraint.

Car:

- Added **Branch_ID** with **FK** constraint.

Coupon_Sale:

- Created composite PK of **Operation_ID** and **Coupon_ID**

which are both **Foreign keys**. Installments:

- Created table, then added **Operation_ID** with FK constraint the references **Sales_operation**.Coupon:
- Created Coupon table.

 Branch:

- Created Branch table.

Shareholder:

- Created Shareholder table.

1. **Triggers and constraints:**
- Trigger to automatically **calculate the price** in the **Sale_operation** table.

```sql
USE [Competition]
GO
/****** Object:  Trigger [dbo].[trg_CalculatePrice_Insert]    Script Date: 8/3/2024 7:18:47 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE TRIGGER [dbo].[trg_CalculatePrice_Insert]
ON [dbo].[Sales_operation]
AFTER INSERT
AS
BEGIN
    UPDATE so
    SET so.price = (cc.price * i.Quantity) + i.fees
    FROM Sales_operation so
    JOIN inserted i ON so.Operation_ID = i.Operation_ID
    JOIN Car c ON c.Car_ID = i.Car_ID
    JOIN Car_Category cc ON cc.Car_Category_ID = c.Category_ID
    WHERE so.Operation_ID = i.Operation_ID;
END;
```

- Trigger to check that both **Salesperson and Car** are from the same branch when inserting or updating to the **Sale_operation** table.

```sql
USE [Competition]
GO
/****** Object:  Trigger [dbo].[trg_CheckBranchID_Insert]    Script Date: 8/3/2024 7:19:47 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TRIGGER [dbo].[trg_CheckBranchID_Insert]
ON [dbo].[Sales_operation]
AFTER INSERT
AS
BEGIN
    -- Check if any new row violates the branch consistency rule
    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN Car c ON i.Car_ID = c.Car_ID
        JOIN Sales_Person sp ON i.SalesPerson_ID = sp.SalesPerson_ID
        WHERE c.Branch_num <> sp.Branch_ID
    )
    BEGIN
        -- Rollback the transaction if the rule is violated
        ROLLBACK TRANSACTION;
        -- Raise an error to notify the user
        RAISERROR ('Branch_ID of Car and Sales_Person must match.', 16, 1);
    END
END;

/****** Object:  Trigger [dbo].[trg_CheckBranchID_Update]    Script Date: 8/3/2024 7:20:22 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TRIGGER [dbo].[trg_CheckBranchID_Update]
ON [dbo].[Sales_operation]
AFTER UPDATE
AS
BEGIN
    -- Check if any updated row violates the branch consistency rule
    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN Car c ON i.Car_ID = c.Car_ID
        JOIN Sales_Person sp ON i.SalesPerson_ID = sp.SalesPerson_ID
        WHERE c.Branch_num <> sp.Branch_ID
    )
    BEGIN
        -- Rollback the transaction if the rule is violated
        ROLLBACK TRANSACTION;
        -- Raise an error to notify the user
        RAISERROR ('Branch_ID of Car and Sales_Person must match.', 16, 1);
    END
END;

```

- Trigger to prevent the price from being updated **without a coupon** in the **Sale_operation** table.

```sql
USE [Competition]
GO
/****** Object:  Trigger [dbo].[trg_PreventPriceUpdateWithoutCoupon]    Script Date: 8/3/2024 7:21:11 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TRIGGER [dbo].[trg_PreventPriceUpdateWithoutCoupon]
ON [dbo].[Sales_operation]
AFTER UPDATE
AS
BEGIN
    -- Check for NULL price
    IF EXISTS (
        SELECT 1
        FROM inserted i
        WHERE i.price IS NULL
    )
    BEGIN
        RAISERROR('Price cannot be updated to NULL.', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END

    -- Check for price update without coupon
    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN deleted d ON i.Operation_ID = d.Operation_ID
        WHERE i.price <> d.price
          AND NOT EXISTS (
              SELECT 1
              FROM Coupon_Sale cs
              JOIN Coupon c ON cs.Coupon_ID = c.Coupon_ID
              WHERE cs.Sale_Op_ID = i.Operation_ID
                AND i.price = d.price - d.price * c.discount
          )
    )
    BEGIN
        RAISERROR('Price cannot be updated directly, except through a valid coupon.', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END
END;

```

- Trigger to **apply the coupon** to sale operation automatically, after insertion in **Coupon_Sale** table.

```sql
USE [Competition]
GO
/****** Object:  Trigger [dbo].[trg_CheckCouponUsage]    Script Date: 8/3/2024 7:21:36 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

ALTER trigger [dbo].[trg_CheckCouponUsage]
on [dbo].[Coupon_Sale]
instead of insert 
as
begin

 declare @Coupon_ID int;
 declare @Sale_Op_ID int;
 declare @Discount float;

 select @Sale_Op_ID = Sale_Op_ID, @Coupon_ID = Coupon_ID from inserted

    -- Check if the coupon is already used
    IF EXISTS (select 1 from Coupon where Coupon_ID = @Coupon_ID AND used = 'True')
    BEGIN
        -- If the coupon is used, rollback the transaction
        RAISERROR('Coupon has already been used', 16, 1);
        ROLLBACK TRANSACTION;
    END
    ELSE
    BEGIN
        -- Start a transaction
        BEGIN TRANSACTION;

        -- 1. Insert into Coupon_Sale table
        insert into Coupon_Sale (Coupon_ID, Sale_Op_ID)
        values (@Coupon_ID, @Sale_Op_ID);

        -- 2. Set the used attribute in the coupon table to true
        UPDATE Coupon
        set Used = 'True'
        where Coupon_ID = @Coupon_ID;

        -- 3. Change the price in the Sales_operation table
        select @Discount = discount from Coupon where Coupon_ID = @Coupon_ID;

        UPDATE Sales_operation
        set price = price - price * @Discount
        where Operation_ID = @Sale_Op_ID;

        -- Commit the transaction
        COMMIT TRANSACTION;
    END
END;
```

1. **Queries**:
- Query 1

```sql
select sum(CAST(Quantity AS float)*CAST(cc.price AS float))/sum(CAST(Quantity AS float)) AS Average_price 
from Car c join Car_Category cc
on c.Category_ID = cc.Car_Category_ID where cc.Manufacturer='TOYOTA'
```

- Result
    
    ![Car_final_quereis.sql - DESKTOP-4LGCD5G_SQLEXPRESS.Competition (DESKTOP-4LGCD5G_Roaa (63)) - Microsoft SQL Server Management Studio 8_3_2024 7_25_02 PM.png](Updated%20Requirements%20Documentation%20ebc3926a05f9466fa2e7e83436dbd752/Car_final_quereis.sql_-_DESKTOP-4LGCD5G_SQLEXPRESS.Competition_(DESKTOP-4LGCD5G_Roaa_(63))_-_Microsoft_SQL_Server_Management_Studio_8_3_2024_7_25_02_PM.png)
    
- Query 2

```sql
DECLARE @Installment_number int = 1;
DECLARE @Due_Date DATE = DATEADD(MONTH, 1, GETDATE());

DECLARE @Total_Amount DECIMAL(18, 3);
select @Total_Amount = Price from Sales_operation where Operation_ID = 13;

DECLARE @Remaining_Amount DECIMAL(18, 3) = @Total_Amount - 10000;

WHILE (@Installment_Number <= 12)
BEGIN
    INSERT INTO installments(Installment_Due_Date, Paid, Amount, Deposit, Sale_op, Installment_number)
    VALUES (@Due_Date,'False', @Remaining_Amount/12, 10000, 13, @installment_number);

    SET @Installment_Number = @Installment_number + 1;
    SET @Due_Date = DATEADD(MONTH, 1, @Due_Date);
END

select sum(amount) from installments where Installment_number <= 5 
```

- Result
    
    ![Car_final_quereis.sql - DESKTOP-4LGCD5G_SQLEXPRESS.Competition (DESKTOP-4LGCD5G_Roaa (63)) - Microsoft SQL Server Management Studio 8_3_2024 7_33_02 PM.png](Updated%20Requirements%20Documentation%20ebc3926a05f9466fa2e7e83436dbd752/Car_final_quereis.sql_-_DESKTOP-4LGCD5G_SQLEXPRESS.Competition_(DESKTOP-4LGCD5G_Roaa_(63))_-_Microsoft_SQL_Server_Management_Studio_8_3_2024_7_33_02_PM.png)
    
- Query 3

```sql
select sum(price) as Cars_in_Mit_Ghamr_price from Car c join Branch b on c.Branch_num = b.Branch_ID
join Car_Category cc on c.Category_ID = cc.Car_Category_ID
where b.Location = 'Mit-Ghamr'
```

- Result
    
    ![Car_final_quereis.sql - DESKTOP-4LGCD5G_SQLEXPRESS.Competition (DESKTOP-4LGCD5G_Roaa (63))_ - Microsoft SQL Server Management Studio 8_3_2024 7_35_05 PM.png](Updated%20Requirements%20Documentation%20ebc3926a05f9466fa2e7e83436dbd752/Car_final_quereis.sql_-_DESKTOP-4LGCD5G_SQLEXPRESS.Competition_(DESKTOP-4LGCD5G_Roaa_(63))__-_Microsoft_SQL_Server_Management_Studio_8_3_2024_7_35_05_PM.png)
    
- Query 4

```sql
select share_percent *(select sum(price) from Sales_operation where Operation_Date >= DATEADD(MONTH, -2, GETDATE())
AND Operation_Date < GETDATE()) AS McMan_Profit from Shareholder where Fname = 'McMan'
```

- Query 5

```sql
select b.Branch_ID, b.Name AS Branch_Name, count(Operation_ID) AS Sales from Branch b Left join Car c 
on Branch_ID = c.Branch_num left join Sales_operation so on c.car_ID = so.car_ID
group by b.Branch_ID, b.Name order by Sales DESC, sum(so.price) DES
```

- Query 6

```sql
-- Trigger [trg_CheckCouponUsage] will handle all the needed steps.
insert into Coupon_Sale (Sale_Op_ID, Coupon_ID) 
values(18,3) -- this coupon has been used before so the query will be rejected

insert into Coupon_Sale (Sale_Op_ID, Coupon_ID) 
values(13,5) -- This coupon hasn't been used so the query will run successfully
```

- Query 7

```sql
--1. Manager of the Agency
select Top 1 * from Shareholder order by share_percent DESC

--2. Manager in case madbouli got 20% from Abdelfattah
begin transaction

update Shareholder 
set share_percent = share_percent - 0.2
where Fname = 'Abdelfattah' and Shareholder_ID = 1

update Shareholder
set share_percent = share_percent + 0.2
where Fname = 'Madboli' and Shareholder_ID = 2

select Top 1 * from Shareholder order by share_percent DESC

rollback transaction
```

- Results for (Query 4, Query 5, Query 7)
    
    ![Car_final_quereis.sql - DESKTOP-4LGCD5G_SQLEXPRESS.Competition (DESKTOP-4LGCD5G_Roaa (63))_ - Microsoft SQL Server Management Studio 8_3_2024 7_37_22 PM.png](Updated%20Requirements%20Documentation%20ebc3926a05f9466fa2e7e83436dbd752/Car_final_quereis.sql_-_DESKTOP-4LGCD5G_SQLEXPRESS.Competition_(DESKTOP-4LGCD5G_Roaa_(63))__-_Microsoft_SQL_Server_Management_Studio_8_3_2024_7_37_22_PM.png)