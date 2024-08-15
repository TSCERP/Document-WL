# "UF_UpdateOrCreateLogQC"
create procedure "UF_UpdateOrCreateLogQC"( in type nvarchar(10),
IN Code nvarchar(254), 
in ItemCode nvarchar(254),
 in  Qty float,
 in WhsCode nvarchar(254),
 in  OpenQty float,
in LLSX NVARCHAR(254),
 in TO nvarchar(254),
 in LL nvarchar(254),
 in HXL nvarchar(254),
 in source nvarchar(254),
in ItemHC nvarchar(254),
in cmtQC nvarchar(254),
IN TCV nvarchar(254))
AS BEGIN 
	if(type='I')
	THEN
	insert into "@V_LOGQC" (
		"Code","Name","U_ItemCode",
		"U_Quantity",
		"U_WhsCode",
		"U_OpenQty",
		"U_Type",
		"U_TO",
		"U_LL",
		"U_HXL", 
		"U_source",
		"U_ItemHC",
		"U_DocDate",
		"U_cmtQC","U_TCV")
		 values (Code,Code,ItemCode,Qty,WhsCode,OpenQty,LLSX,TO,LL,HXL, source,ItemHC,CURRENT_DATE,cmtQC,TCV) ;
	END IF;
	IF(type='U') then
	if exists (select top 1 1 from "@V_LOGQC" where "U_OpenQty">0 and "Code"=Code ) then
		update "@V_LOGQC" set "U_OpenQty"="U_OpenQty"-Qty where "Code"=Code;
	end if;
	End if; 
	

END;
# "USP_AllocateIssue"
CREATE procedure "USP_AllocateIssue" (IN ItemCode NVARCHAR(5000),
    IN WhsCode NVARCHAR(5000),
    IN Qty FLOAT,
    IN Type int)

as 

begin 
/*
0 khong quan ly batch/serial
1 batch
2 serial
*/
	if (Type = 0) then 			 
		select WhsCode "WhsCode",null "BatchSeri",Qty "Allocated" from dummy;
	else
		select * from U_Allocate_Batch_Serial(ItemCode,WhsCode,Qty,:type);
	end if;
end;
# "USP_ChiTietSLVCN_RONG" 
CREATE procedure "USP_ChiTietSLVCN_RONG" 
(IN Item nvarchar(255),TO nvarchar(255) ,version nvarchar(255))
as
begin
select T0."ItemFather",T0."FatherName",t0."ItemCode",t0."ItemName",
t0."SanLuong",t0."DaLam",t0."Loi",
(t0."SanLuong"-t0."DaLam"-t0."Loi") "ConLai",
t0."U_CDOAN","CDai","CRong","CDay" from(
select tf."ItemCode" "ItemFather","U_To",
t0."U_Version" "Version",TF."ItemName" "FatherName",T1."ItemCode",t5."ItemName",sum(t1."PlannedQty"*-1) "SanLuong", 
SUM(ifnull(T3."Quantity",0)) "DaLam", SUM(ifnull(T4."Quantity",0)) "Loi",
ifnull(T5."U_CDai",0) "CDai",ifnull(T5."U_CRong",0) "CRong",ifnull(T5."U_CDay",0) "CDay",
t0."U_CDOAN"
from OWOR T0 
JOIN WOR1 T1 ON T0."DocEntry"=t1."DocEntry" 
join (SELECT t0."DocEntry",t1."ItemCode",t1."ItemName"
from OWOR T0 
JOIN WOR1 T1 ON T0."DocEntry"=t1."DocEntry" and  T1."BaseQty" > 0
and t1."ItemType"=4 and t0."Status"='R' 
and T0."U_CDOAN"='RO') TF on t0."DocEntry"=tf."DocEntry"
LEFT JOIN IGN1 T3 ON T1."DocEntry"=t3."BaseEntry" and t3."BaseType"=202 and t3."TranType"='C' and t3."ItemCode"=t1."ItemCode"
LEFT JOIN IGN1 T4 ON T1."DocEntry"=t4."BaseEntry" and t4."BaseType"=202 and t4."TranType"='R' and t4."ItemCode"=t1."ItemCode"
inner join OITM t5 ON T1."ItemCode"=T5."ItemCode"
inner join OITM t6 ON T0."U_SPDICH"=T6."ItemCode"
wHERE t0."U_CDOAN"='RO' AND T0."Status"='R' and t1."ItemType"=4 and t1."PlannedQty"<0
and tf."ItemCode"=item and t0."U_To"=TO and ifnull(t0."U_Version",1)=version
group by tf."ItemCode",Tf."ItemName",t1."ItemCode",t0."U_CDOAN",t5."ItemName",t0."U_Version","U_To",
T5."U_CDai",T5."U_CRong" ,T5."U_CDay"
having sum(t1."PlannedQty"*-1)- SUM(ifnull(T3."Quantity",0))- SUM(ifnull(T4."Quantity",0)) >0) T0;
END;
# usp_issueAutoData
CREATE PROCEDURE usp_issueAutoData (IN inputString NVARCHAR(5000)) 
LANGUAGE SQLSCRIPT 
AS 
BEGIN 
    -- Declare variables to track current position and delimiters
    DECLARE currentPosition INT := 1;
    DECLARE nextSemicolon INT;
    DECLARE nextDash INT;
    DECLARE item NVARCHAR(255);
    DECLARE docentry INT;
    DECLARE Qty float;
    DECLARE error INT := 0;
    DECLARE ItemError NVARCHAR(1000);
    -- Create temporary tables
    CREATE LOCAL TEMPORARY TABLE "#resultTable" (DocEntry INT, Qty float);
    
    -- Loop through the string to split elements
    WHILE currentPosition <= LENGTH(:inputString) DO 
        nextSemicolon := INSTR(:inputString, ';', currentPosition);
        
        IF nextSemicolon = 0 THEN 
            nextSemicolon := LENGTH(:inputString) + 1;
        END IF;
        
        item := SUBSTR(:inputString, currentPosition, nextSemicolon - currentPosition);
        nextDash := INSTR(:item, '-');
        
        IF nextDash > 0 THEN 
            docentry := TO_INTEGER(SUBSTR(:item, 1, nextDash - 1));
            Qty := cast(SUBSTR(:item, nextDash + 1) as float);
            INSERT INTO "#resultTable" (DocEntry, Qty) VALUES (:docentry, :Qty); 
        END IF;
        
        currentPosition := nextSemicolon + 1;
    END WHILE;

    -- Check for insufficient inventory
    IF EXISTS (SELECT 1 
               FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                     FROM "#resultTable" a 
                     JOIN OWOR b ON a.DocEntry = b."DocEntry"
                     JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                     AND c."IssueType" = 'M' 
                     AND c."BaseQty" > 0 
                     GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
               JOIN OITM b ON a."ItemCode" = b."ItemCode" 
               JOIN OITW c ON c."WhsCode" = a."wareHouse" 
               AND a."ItemCode" = c."ItemCode" 
               WHERE "Qty" > c."OnHand") THEN
        error := 1;
    END IF;

    -- Handle error case
    IF (:error = 1) THEN 
        SELECT  top 1 'Mã ' || a."ItemCode" || ' Không đủ tồn kho' 
        INTO ItemError 
        FROM ( select A.DocEntry "DocEntry",c."ItemCode",
			sum(c."BaseQty"*a.Qty) "Qty", c."wareHouse" 
              FROM "#resultTable" a 
              JOIN OWOR b ON a.DocEntry = b."DocEntry" 
              JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
              AND c."IssueType" = 'M' 
              AND c."BaseQty" > 0 
              GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
        JOIN OITM b ON a."ItemCode" = b."ItemCode" 
        JOIN OITW c ON c."WhsCode" = a."wareHouse" 
        AND a."ItemCode" = c."ItemCode" 
        WHERE a."Qty" > c."OnHand";
        
        SELECT ItemError AS "ItemError" FROM DUMMY;
    ELSE
    
            SELECT a."DocEntry", a."ItemCode", a."wareHouse" "WhsCode", a."Qty", 
                   CASE WHEN b."ManBtchNum" = 'Y' THEN 'B' 
                        WHEN b."ManSerNum" = 'Y' THEN 'S' 
                        ELSE 'N' END AS "Type" 
            FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                  FROM "#resultTable" a 
                  JOIN OWOR b ON a.DocEntry = b."DocEntry" 
                  JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                  AND c."IssueType" = 'M' 
                  AND c."BaseQty" > 0 
                  GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
            JOIN OITM b ON a."ItemCode" = b."ItemCode" 
            JOIN OITW c ON c."WhsCode" = a."wareHouse" 
            AND a."ItemCode" = c."ItemCode";

    END IF;

drop table "#resultTable";
    -- Temporary tables are automatically dropped at the end of the session
END;
# usp_issueAutoData_v2
CREATE PROCEDURE usp_issueAutoData_v2 (IN inputString NVARCHAR(5000)) 
LANGUAGE SQLSCRIPT 
AS 
BEGIN 
    -- Declare variables to track current position and delimiters
    DECLARE currentPosition INT := 1;
    DECLARE nextSemicolon INT;
    DECLARE nextDash INT;
    DECLARE item NVARCHAR(255);
    DECLARE docentry INT;
    DECLARE Qty float;
    DECLARE error INT := 0;
    DECLARE ItemError NVARCHAR(1000);
     DECLARE lv_docentry INTEGER;
    DECLARE lv_itemcode NVARCHAR(50);
    DECLARE lv_whscode NVARCHAR(50);
    DECLARE lv_qty DECIMAL(18,2);
    DECLARE lv_type NVARCHAR(1);

   
create local temporary table #allocationData
	("DocEntry" nvarchar(200),
	 "ItemCode" nvarchar(254),
	 "WhsCode" nvarchar(254),
	 "Qty" float,
	 "Batch" nvarchar(254) null,
	 "Serial" nvarchar(254) null);
create local temporary table #Stock(
	 "ItemCode" nvarchar(254),
	 "WhsCode" nvarchar(254),
	 "Qty" float,
	 "BatchSeri" nvarchar(254) null,
	 "Type" nvarchar(254) null,
	 "Status" int,
	 "DocEntry" nvarchar(200));	 
  create local temporary TABLE #runingtemp(
 	"BatchSeri" nvarchar(254) null,
	"Quantity" float,
	"RunningSum" float,
	"WhsCode" nvarchar(254),
	"ItemCode" nvarchar(254));
  create local temporary TABLE #data(
 		"DocEntry" nvarchar(254),
 		"ItemCode" nvarchar(254),
 		"WhsCode" nvarchar(254),
 		"Qty" float, 
         "Type" nvarchar(2) );
    -- Create temporary tables
    CREATE LOCAL TEMPORARY TABLE "#resultTable" (DocEntry INT, Qty float);
    
    -- Loop through the string to split elements
    WHILE currentPosition <= LENGTH(:inputString) DO 
        nextSemicolon := INSTR(:inputString, ';', currentPosition);
        
        IF nextSemicolon = 0 THEN 
            nextSemicolon := LENGTH(:inputString) + 1;
        END IF;
        
        item := SUBSTR(:inputString, currentPosition, nextSemicolon - currentPosition);
        nextDash := INSTR(:item, '-');
        
        IF nextDash > 0 THEN 
            docentry := TO_INTEGER(SUBSTR(:item, 1, nextDash - 1));
            Qty := cast(SUBSTR(:item, nextDash + 1) as float);
            INSERT INTO "#resultTable" (DocEntry, Qty) VALUES (:docentry, :Qty); 
        END IF;
        
        currentPosition := nextSemicolon + 1;
    END WHILE;

    -- Check for insufficient inventory
    IF EXISTS (SELECT 1 
               FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                     FROM "#resultTable" a 
                     JOIN OWOR b ON a.DocEntry = b."DocEntry"
                     JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                     AND c."IssueType" = 'M' 
                     AND c."BaseQty" > 0 
                     GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
               JOIN OITM b ON a."ItemCode" = b."ItemCode" 
               JOIN OITW c ON c."WhsCode" = a."wareHouse" 
               AND a."ItemCode" = c."ItemCode" 
               WHERE "Qty" > c."OnHand") THEN
        error := 1;
    END IF;

    -- Handle error case
    IF (:error = 1) THEN 
        SELECT  top 1 'Mã ' || a."ItemCode" || ' Không đủ tồn kho' 
        INTO ItemError 
        FROM ( select A.DocEntry "DocEntry",c."ItemCode",
			sum(c."BaseQty"*a.Qty) "Qty", c."wareHouse" 
              FROM "#resultTable" a 
              JOIN OWOR b ON a.DocEntry = b."DocEntry" 
              JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
              AND c."IssueType" = 'M' 
              AND c."BaseQty" > 0 
              GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
        JOIN OITM b ON a."ItemCode" = b."ItemCode" 
        JOIN OITW c ON c."WhsCode" = a."wareHouse" 
        AND a."ItemCode" = c."ItemCode" 
        WHERE a."Qty" > c."OnHand";
        
        SELECT ItemError AS "ItemError" FROM DUMMY;
    ELSE
    ---khởi tao data
          
	-----

	
 DECLARE CURSOR cur  FOR
        SELECT * FROM (	SELECT a."DocEntry", a."ItemCode", a."wareHouse" "WhsCode", a."Qty", 
                   CASE WHEN b."ManBtchNum" = 'Y' THEN 'B' 
                        WHEN b."ManSerNum" = 'Y' THEN 'S' 
                        ELSE 'N' END AS "Type" 
            FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                  FROM "#resultTable" a 
                  JOIN OWOR b ON a.DocEntry = b."DocEntry" 
                  JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                  AND c."IssueType" = 'M' 
                  AND c."BaseQty" > 0 
                  GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
            JOIN OITM b ON a."ItemCode" = b."ItemCode" 
            JOIN OITW c ON c."WhsCode" = a."wareHouse" 
            AND a."ItemCode" = c."ItemCode") where "Type"<>'N';
        
       insert into #data
			SELECT a."DocEntry", a."ItemCode", a."wareHouse" "WhsCode", a."Qty", 
                   CASE WHEN b."ManBtchNum" = 'Y' THEN 'B' 
                        WHEN b."ManSerNum" = 'Y' THEN 'S' 
                        ELSE 'N' END AS "Type" 
            FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                  FROM "#resultTable" a 
                  JOIN OWOR b ON a.DocEntry = b."DocEntry" 
                  JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                  AND c."IssueType" = 'M' 
                  AND c."BaseQty" > 0 
                  GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
            JOIN OITM b ON a."ItemCode" = b."ItemCode" 
            JOIN OITW c ON c."WhsCode" = a."wareHouse" 
            AND a."ItemCode" = c."ItemCode";
            
	 insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty")
	 select "DocEntry","ItemCode","WhsCode","Qty" from #data t0 where "Type"='N';
	  
	  
	  insert into #Stock ("ItemCode","WhsCode","Qty", "BatchSeri","Type","Status")
	 SELECT "ItemCode","WhsCode","Qty", "BatchSeri","Type",0 "status"
	 FROM uspweb_stock_batchserial where "ItemCode" in (
								 select distinct "ItemCode" 
								 from  #data where "Type"<>'N' )
	  and "WhsCode" in (
	  				select distinct "WhsCode"  
	  				from  #data where "Type"<>'N');
    OPEN cur;

    FETCH cur INTO lv_docentry, lv_itemcode, lv_whscode, lv_qty, lv_type;

    WHILE :lv_docentry IS NOT NULL DO
		if(:lv_type='S')
		THEN 
		UPDATE TOP :lv_qty  #Stock t0 SET  "Status" =1,"DocEntry"=:lv_docentry where "Status"=0;
		
		
		END IF;
      	if(:lv_type='B')
      	THEN
      	--- b1 tao data runing
      	insert into #runingtemp
      	select
 		 "BatchSeri",
        "Qty" "Quantity",
       
        SUM("Qty") OVER (ORDER BY "BatchSeri") AS "RunningSum",
         "WhsCode",
        "ItemCode"
        FROM #Stock
    	where "Type"='B' and "ItemCode"=lv_itemcode and "WhsCode"=lv_whscode and "Qty">0;
    	-- b2 allocate
		insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty","Batch")
		SELECT 
		lv_docentry,
		lv_itemcode,
		lv_whscode,
		t0."Allocated",
		 t0."BatchSeri"
		 from (select 
        "Quantity",
        CASE 
            WHEN  "RunningSum" <= lv_qty THEN "Quantity"
            WHEN  "RunningSum" - "Quantity" < lv_qty THEN lv_qty - ("RunningSum" - "Quantity")
            ELSE 0
        END AS "Allocated",
         "BatchSeri"
	    FROM  #runingtemp ) as t0 
	    where t0."Allocated" >0;
		---- cập nhật lại tồn của stock
	   update  #Stock t0 SET "Qty"="Qty"-t1."Allocated"  
		 from #Stock  t0
		  join 
		  ( SELECT 
		      "BatchSeri",
		       "Quantity",
		       "ItemCode",
		       "WhsCode",
		        CASE 
		            WHEN  "RunningSum" <= lv_qty THEN "Quantity"
		            WHEN  "RunningSum" - "Quantity" < lv_qty THEN lv_qty - ("RunningSum" - "Quantity")
		            ELSE 0
		        END AS "Allocated"
		    FROM #runingtemp
    )t1 on t0."BatchSeri"=t1."BatchSeri" and t0."ItemCode"=t1."ItemCode" and t0."WhsCode"=t1."WhsCode";
    DELETE from #Stock WHERE "Qty"=0;
 	DELETE from #runingtemp;
	    END if;
        FETCH cur INTO lv_docentry, lv_itemcode, lv_whscode, lv_qty, lv_type;
    END WHILE;

    CLOSE cur;
	insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty","Serial")
	 select "DocEntry","ItemCode","WhsCode","Qty","BatchSeri" from #Stock where "Status"=1;
    SELECT * FROM  #allocationData;
    END IF;
    DROP TABLE  #allocationData;
     DROP TABLE  #Stock;
     DROP TABLE #runingtemp;
	drop table "#resultTable";
	drop table #data;
    -- Temporary tables are automatically dropped at the end of the session
END;
# usp_webOpenProductionOrder
CREATE procedure usp_webOpenProductionOrder( in ItemCode nvarchar(254),in Team nvarchar(200),in ver int)
 as 
 begin 
 SELECT a."DocEntry",B."ItemCode",(b."PlannedQty"-b."IssuedQty") "Qty",0 "Allocated",b."LineNum" from 
OWOR A JOIN WOR1 B ON A."DocEntry"= b."DocEntry" 
and a."U_CDOAN"='RO' 
AND B."BaseQty" >0
 and a."Status"='R'
 and b."ItemType"=4
and   b."PlannedQty"> b."IssuedQty" 
where b."ItemCode"=ItemCode and a."U_Version"=ver
and a."U_To"=team
order by b."DocEntry" asc;
 end
 CREATE PROCEDURE USP_WEB_ALLOCATEBatchSerial()
LANGUAGE SQLSCRIPT
AS
BEGIN
	
    DECLARE lv_docentry INTEGER;
    DECLARE lv_itemcode NVARCHAR(50);
    DECLARE lv_whscode NVARCHAR(50);
    DECLARE lv_qty DECIMAL(18,2);
    DECLARE lv_type NVARCHAR(1);

    DECLARE CURSOR cur  FOR
        SELECT * FROM DATA where "Type"<>'N';
create local temporary table #allocationData
	("DocEntry" nvarchar(200),
	 "ItemCode" nvarchar(254),
	 "WhsCode" nvarchar(254),
	 "Qty" float,
	 "Batch" nvarchar(254) null,
	 "Serial" nvarchar(254) null);
create local temporary table #Stock(
	 "ItemCode" nvarchar(254),
	 "WhsCode" nvarchar(254),
	 "Qty" float,
	 "BatchSeri" nvarchar(254) null,
	 "Type" nvarchar(254) null,
	 "Status" int,
	 "DocEntry" nvarchar(200));	 
  create local temporary TABLE #runingtemp(
 		"BatchSeri" nvarchar(254) null,
	  "Quantity" float,
	 "RunningSum" float,
	 "WhsCode" nvarchar(254),
	  "ItemCode" nvarchar(254));
	 
	 insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty")
	 select "DocEntry","ItemCode","WhsCode","Qty" from Data where "Type"='N';
	  insert into #Stock ("ItemCode","WhsCode","Qty", "BatchSeri","Type","Status")
	 SELECT "ItemCode","WhsCode","Qty", "BatchSeri","Type",0 "status"
	 FROM uspweb_stock_batchserial where "ItemCode" in (select distinct "ItemCode" from  Data where "Type"<>'N' )
	  and "WhsCode" in (select distinct "WhsCode" from  Data where "Type"<>'N');
    OPEN cur;

    FETCH cur INTO lv_docentry, lv_itemcode, lv_whscode, lv_qty, lv_type;

    WHILE :lv_docentry IS NOT NULL DO
		if(:lv_type='S')
		THEN 
		UPDATE TOP :lv_qty  #Stock t0 SET  "Status" =1,"DocEntry"=:lv_docentry where "Status"=0;
		
		
		END IF;
      	if(:lv_type='B')
      	THEN
      	--- b1 tao data runing
      	insert into #runingtemp
      	select
 		 "BatchSeri",
        "Qty" "Quantity",
       
        SUM("Qty") OVER (ORDER BY "BatchSeri") AS "RunningSum",
         "WhsCode",
        "ItemCode"
        FROM #Stock
    	where "Type"='B' and "ItemCode"=lv_itemcode and "WhsCode"=lv_whscode and "Qty">0;
    	-- b2 allocate
		insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty","Batch")
		SELECT 
		lv_docentry,
		lv_itemcode,
		lv_whscode,
		t0."Allocated",
		 t0."BatchSeri"
		 from (select 
        "Quantity",
        CASE 
            WHEN  "RunningSum" <= lv_qty THEN "Quantity"
            WHEN  "RunningSum" - "Quantity" < lv_qty THEN lv_qty - ("RunningSum" - "Quantity")
            ELSE 0
        END AS "Allocated",
         "BatchSeri"
	    FROM  #runingtemp ) as t0 
	    where t0."Allocated" >0;
		---- cập nhật lại tồn của stock
	   update  #Stock t0 SET "Qty"="Qty"-t1."Allocated"  
		 from #Stock  t0
		  join 
		  ( SELECT 
		      "BatchSeri",
		       "Quantity",
		       "ItemCode",
		       "WhsCode",
		        CASE 
		            WHEN  "RunningSum" <= lv_qty THEN "Quantity"
		            WHEN  "RunningSum" - "Quantity" < lv_qty THEN lv_qty - ("RunningSum" - "Quantity")
		            ELSE 0
		        END AS "Allocated"
		    FROM runingtemp
    )t1 on t0."BatchSeri"=t1."BatchSeri" and t0."ItemCode"=t1."ItemCode" and t0."WhsCode"=t1."WhsCode";
    DELETE from #Stock WHERE "Qty"=0;
 	DELETE from #runingtemp;
	    END if;
        FETCH cur INTO lv_docentry, lv_itemcode, lv_whscode, lv_qty, lv_type;
    END WHILE;

    CLOSE cur;
     insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty","Serial")
	 select "DocEntry","ItemCode","WhsCode","Qty","BatchSeri" from #Stock where "Status"=1;
    SELECT * FROM  #allocationData;
    DROP TABLE  #allocationData;
     DROP TABLE  #Stock;
     DROP TABLE #runingtemp;
END;
# "usp_web_detailrong"
CREATE procedure "usp_web_detailrong" (
in sp nvarchar(254),
in item nvarchar(254),
in to nvarchar(254),
in ver nvarchar(254)
)
as 
begin 
select a."DocEntry", a."U_SPDICH" "SPDICH",a."U_Version" "Version",A."U_To" "TO",
 b."ItemCode",-1*(B."PlannedQty"-b."IssuedQty") "ConLai",b."LineNum",
  case when c."ManBtchNum" = 'Y' then 'B' when c."ManSerNum" = 'Y' then 'S' else 'N' end "BatchSeri"
from OWOR A JOIN WOR1 B 
join OITM c on b."ItemCode"=c."ItemCode"
ON A."DocEntry"=b."DocEntry" 
and a."U_CDOAN"='RO' 
AND b."BaseQty"< 0 
and b."ItemType"=4
and a."Status"='R'
and B."PlannedQty"<b."IssuedQty"
where a."U_SPDICH"=sp and a."U_Version"=ver and b."ItemCode"=item and a."U_To"=to
order by a."DocEntry" asc;
end;
# usp_web_issueAutoData
CREATE PROCEDURE usp_web_issueAutoData (IN inputString NVARCHAR(5000)) 
LANGUAGE SQLSCRIPT 
AS 
BEGIN 
    -- Declare variables to track current position and delimiters
    DECLARE currentPosition INT := 1;
    DECLARE nextSemicolon INT;
    DECLARE nextDash INT;
    DECLARE item NVARCHAR(255);
    DECLARE docentry INT;
    DECLARE Qty float;
    DECLARE error INT := 0;
    DECLARE ItemError NVARCHAR(1000);
     DECLARE lv_docentry INTEGER;
    DECLARE lv_itemcode NVARCHAR(50);
    DECLARE lv_whscode NVARCHAR(50);
    DECLARE lv_qty DECIMAL(18,2);
    DECLARE lv_type NVARCHAR(1);

   
create local temporary table #allocationData
	("DocEntry" nvarchar(200),
	 "ItemCode" nvarchar(254),
	 "WhsCode" nvarchar(254),
	 "Qty" float,
	 "Batch" nvarchar(254) null,
	 "Serial" nvarchar(254) null);
create local temporary table #Stock(
	 "ItemCode" nvarchar(254),
	 "WhsCode" nvarchar(254),
	 "Qty" float,
	 "BatchSeri" nvarchar(254) null,
	 "Type" nvarchar(254) null,
	 "Status" int,
	 "DocEntry" nvarchar(200));	 
  create local temporary TABLE #runingtemp(
 	"BatchSeri" nvarchar(254) null,
	"Quantity" float,
	"RunningSum" float,
	"WhsCode" nvarchar(254),
	"ItemCode" nvarchar(254));
  create local temporary TABLE #data(
 		"DocEntry" nvarchar(254),
 		"ItemCode" nvarchar(254),
 		"WhsCode" nvarchar(254),
 		"Qty" float, 
         "Type" nvarchar(2) );
    -- Create temporary tables
    CREATE LOCAL TEMPORARY TABLE "#resultTable" (DocEntry INT, Qty float);
    
    -- Loop through the string to split elements
    WHILE currentPosition <= LENGTH(:inputString) DO 
        nextSemicolon := INSTR(:inputString, ';', currentPosition);
        
        IF nextSemicolon = 0 THEN 
            nextSemicolon := LENGTH(:inputString) + 1;
        END IF;
        
        item := SUBSTR(:inputString, currentPosition, nextSemicolon - currentPosition);
        nextDash := INSTR(:item, '-');
        
        IF nextDash > 0 THEN 
            docentry := TO_INTEGER(SUBSTR(:item, 1, nextDash - 1));
            Qty := cast(SUBSTR(:item, nextDash + 1) as float);
            INSERT INTO "#resultTable" (DocEntry, Qty) VALUES (:docentry, :Qty); 
        END IF;
        
        currentPosition := nextSemicolon + 1;
    END WHILE;

    -- Check for insufficient inventory
    IF EXISTS (SELECT 1 
               FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                     FROM "#resultTable" a 
                     JOIN OWOR b ON a.DocEntry = b."DocEntry"
                     JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                     AND c."IssueType" = 'M' 
                     AND c."BaseQty" > 0 
                     GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
               JOIN OITM b ON a."ItemCode" = b."ItemCode" 
               JOIN OITW c ON c."WhsCode" = a."wareHouse" 
               AND a."ItemCode" = c."ItemCode" 
               WHERE "Qty" > c."OnHand") THEN
        error := 1;
    END IF;

    -- Handle error case
    IF (:error = 1) THEN 
        SELECT  top 1 'Mã ' || a."ItemCode" || ' Không đủ tồn kho, Tại kho ' || a."wareHouse" 
        INTO ItemError 
        FROM ( select A.DocEntry "DocEntry",c."ItemCode",
			sum(c."BaseQty"*a.Qty) "Qty", c."wareHouse" 
              FROM "#resultTable" a 
              JOIN OWOR b ON a.DocEntry = b."DocEntry" 
              JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
              AND c."IssueType" = 'M' 
              AND c."BaseQty" > 0 
              GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
        JOIN OITM b ON a."ItemCode" = b."ItemCode" 
        JOIN OITW c ON c."WhsCode" = a."wareHouse" 
        AND a."ItemCode" = c."ItemCode" 
        WHERE a."Qty" > c."OnHand";
        
        SELECT ItemError AS "ItemError" FROM DUMMY;
    ELSE
    ---khởi tao data
          
	-----

	
 DECLARE CURSOR cur  FOR
        SELECT * FROM (	SELECT a."DocEntry", a."ItemCode", a."wareHouse" "WhsCode", a."Qty", 
                   CASE WHEN b."ManBtchNum" = 'Y' THEN 'B' 
                        WHEN b."ManSerNum" = 'Y' THEN 'S' 
                        ELSE 'N' END AS "Type" 
            FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                  FROM "#resultTable" a 
                  JOIN OWOR b ON a.DocEntry = b."DocEntry" 
                  JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                  AND c."IssueType" = 'M' 
                  AND c."BaseQty" > 0 
                  GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
            JOIN OITM b ON a."ItemCode" = b."ItemCode" 
            JOIN OITW c ON c."WhsCode" = a."wareHouse" 
            AND a."ItemCode" = c."ItemCode") where "Type"<>'N';
        
       insert into #data
			SELECT a."DocEntry", a."ItemCode", a."wareHouse" "WhsCode", a."Qty", 
                   CASE WHEN b."ManBtchNum" = 'Y' THEN 'B' 
                        WHEN b."ManSerNum" = 'Y' THEN 'S' 
                        ELSE 'N' END AS "Type" 
            FROM (SELECT A.DocEntry "DocEntry", c."ItemCode", SUM(c."BaseQty" * a.Qty) AS "Qty", c."wareHouse" 
                  FROM "#resultTable" a 
                  JOIN OWOR b ON a.DocEntry = b."DocEntry" 
                  JOIN WOR1 c ON c."DocEntry" = b."DocEntry" 
                  AND c."IssueType" = 'M' 
                  AND c."BaseQty" > 0 
                  GROUP BY A.DocEntry, c."ItemCode", c."wareHouse") AS a 
            JOIN OITM b ON a."ItemCode" = b."ItemCode" 
            JOIN OITW c ON c."WhsCode" = a."wareHouse" 
            AND a."ItemCode" = c."ItemCode";
            
	 insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty")
	 select "DocEntry","ItemCode","WhsCode","Qty" from #data t0 where "Type"='N';
	  
	  
	  insert into #Stock ("ItemCode","WhsCode","Qty", "BatchSeri","Type","Status")
	 SELECT "ItemCode","WhsCode","Qty", "BatchSeri","Type",0 "status"
	 FROM uspweb_stock_batchserial where "ItemCode" in (
								 select distinct "ItemCode" 
								 from  #data where "Type"<>'N' )
	  and "WhsCode" in (
	  				select distinct "WhsCode"  
	  				from  #data where "Type"<>'N');
    OPEN cur;

    FETCH cur INTO lv_docentry, lv_itemcode, lv_whscode, lv_qty, lv_type;

    WHILE :lv_docentry IS NOT NULL DO
		if(:lv_type='S')
		THEN 
		UPDATE TOP :lv_qty  #Stock t0 SET  "Status" =1,"DocEntry"=:lv_docentry where "Status"=0;
		
		
		END IF;
      	if(:lv_type='B')
      	THEN
      	--- b1 tao data runing
      	insert into #runingtemp
      	select
 		 "BatchSeri",
        "Qty" "Quantity",
       
        SUM("Qty") OVER (ORDER BY "BatchSeri") AS "RunningSum",
         "WhsCode",
        "ItemCode"
        FROM #Stock
    	where "Type"='B' and "ItemCode"=lv_itemcode and "WhsCode"=lv_whscode and "Qty">0;
    	-- b2 allocate
		insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty","Batch")
		SELECT 
		lv_docentry,
		lv_itemcode,
		lv_whscode,
		t0."Allocated",
		 t0."BatchSeri"
		 from (select 
        "Quantity",
        CASE 
            WHEN  "RunningSum" <= lv_qty THEN "Quantity"
            WHEN  "RunningSum" - "Quantity" < lv_qty THEN lv_qty - ("RunningSum" - "Quantity")
            ELSE 0
        END AS "Allocated",
         "BatchSeri"
	    FROM  #runingtemp ) as t0 
	    where t0."Allocated" >0;
		---- cập nhật lại tồn của stock
	   update  #Stock t0 SET "Qty"="Qty"-t1."Allocated"  
		 from #Stock  t0
		  join 
		  ( SELECT 
		      "BatchSeri",
		       "Quantity",
		       "ItemCode",
		       "WhsCode",
		        CASE 
		            WHEN  "RunningSum" <= lv_qty THEN "Quantity"
		            WHEN  "RunningSum" - "Quantity" < lv_qty THEN lv_qty - ("RunningSum" - "Quantity")
		            ELSE 0
		        END AS "Allocated"
		    FROM #runingtemp
    )t1 on t0."BatchSeri"=t1."BatchSeri" and t0."ItemCode"=t1."ItemCode" and t0."WhsCode"=t1."WhsCode";
    DELETE from #Stock WHERE "Qty"=0;
 	DELETE from #runingtemp;
	    END if;
        FETCH cur INTO lv_docentry, lv_itemcode, lv_whscode, lv_qty, lv_type;
    END WHILE;

    CLOSE cur;
	insert into #allocationData ("DocEntry","ItemCode","WhsCode","Qty","Serial")
	 select "DocEntry","ItemCode","WhsCode","Qty","BatchSeri" from #Stock where "Status"=1;
    SELECT t0.*,T1."LineNum" as "BaseLine",SUM("Qty") OVER (PARTITION BY "LineNum" )"QtyTotal" FROM  #allocationData T0 join WOR1 T1 ON 
    T0."DocEntry"=t1."DocEntry" and t1."ItemCode"=t0."ItemCode" AND t1."IssueType"='M';
    END IF;
    DROP TABLE  #allocationData;
     DROP TABLE  #Stock;
     DROP TABLE #runingtemp;
	drop table "#resultTable";
	drop table #data;
    -- Temporary tables are automatically dropped at the end of the session
END;
# UV_WEB_StockRong
CREATE PROCEDURE UV_WEB_StockRong (in ItemCode nvarchar(254))
as
begin
SELECT 
A."ItemCode",SUM(c."OnHand") "Qty"
from 
(
select distinct b."ItemCode",b."wareHouse"
from OWOR A JOIN WOR1 B 
ON A."DocEntry"=b."DocEntry" 
and a."U_CDOAN"='RO' 
AND b."BaseQty">0 
and b."ItemType"=4
and a."Status"='R'
where  b."ItemCode"=ItemCode) a
join OITW C ON a."ItemCode"=c."ItemCode" and c."WhsCode"=a."wareHouse"
GROUP BY A."ItemCode";

end