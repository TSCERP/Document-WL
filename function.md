CREATE OR REPLACE FUNCTION "Uf_AllocateIssue" (
    IN ItemCode NVARCHAR(5000),
    IN WhsCode NVARCHAR(5000),
    IN Qty FLOAT,
    IN Type INT
)
RETURNS TABLE (
    "WhsCode" NVARCHAR(5000),
    "BatchSeri" NVARCHAR(5000),
    "Allocated" FLOAT
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    IF :Type = 0 THEN
        RETURN
        SELECT
            :WhsCode AS "WhsCode",
            NULL AS "BatchSeri",
            :Qty AS "Allocated"
        FROM dummy;
    ELSE
        RETURN
        SELECT *
        FROM U_Allocate_Batch_Serial(:ItemCode, :WhsCode, :Qty, :Type);
    END IF;
END;

CREATE OR Replace FUNCTION "Uf_CheckStockByWhsCode_ManBy"(
    IN ItemCode NVARCHAR(5000),
    IN WhsCode NVARCHAR(5000)
)
RETURNS TABLE (
	"ItemType" int,
    "Stock" FLOAT
)
AS 
BEGIN 
    RETURN 
select case when  T0."ManBtchNum"='Y' AND t0."ManSerNum"='N' then 1
	            when  T0."ManBtchNum"='N' AND t0."ManSerNum"='Y' then 2
				else  0 end "ItemType" , t1."OnHand" "Stock"
				 from OITM T0 join OITW t1 on t0."ItemCode"=t1."ItemCode"
				 where t0."ItemCode"=ItemCode and t1."WhsCode"=WhsCode;
end;
CREATE or replace FUNCTION U_Allocate_Batch_Serial(
    IN ItemCode NVARCHAR(5000),
    IN WhsCode NVARCHAR(5000),
    IN Qty FLOAT,
    type int
)
RETURNS TABLE (
	"WhsCode" nvarchar(5000),
    "BatchSeri" NVARCHAR(5000),
    "Allocated" FLOAT
)
AS 
BEGIN 
	if(type=1) then 
    RETURN 
   WITH BatchStocks AS (
	SELECT T1."BatchNum", 
	T1."Quantity" as "Quantity"
	FROM OITW T0
	INNER JOIN OIBT T1 ON T0."WhsCode" = T1."WhsCode" and T0."ItemCode" = T1."ItemCode" 
	Inner join OITM T2 on T0."ItemCode" = T2."ItemCode"
	WHERE T0."ItemCode"=ItemCode
	and T1."WhsCode"=WhsCode
	and t1."Quantity">0
	order by t1."CreateDate" asc
),
RunningTotal AS (
    SELECT 
        "BatchNum",
        "Quantity",
        SUM("Quantity") OVER (ORDER BY "BatchNum") AS "RunningSum"
    FROM BatchStocks
),
AllocatedStock AS (
    SELECT 
         "BatchNum",
       "Quantity",
        CASE 
            WHEN  "RunningSum" <= Qty THEN "Quantity"
            WHEN  "RunningSum" - "Quantity" < Qty THEN Qty - ("RunningSum" - "Quantity")
            ELSE 0
        END AS "Allocated"
    FROM RunningTotal
)
SELECT WhsCode "WhsCode", "BatchNum" "BatchSeri", t0."Allocated"
from AllocatedStock t0 where t0."Allocated" >0;
end if;
if(type=2) then
return
select "WhsCode","SysSerial" "BatchSeri", 1 "Allocated" from OSRI 
where "ItemCode"=ItemCode and "Status"=0 and "WhsCode"=WhsCode and 
 "SysSerial" between 1 and Qty
order by "SysSerial" asc;
end if;
END;