# "USPWEB_STOCK_BATCHSERIAL"
CREATE or replace VIEW "USPWEB_STOCK_BATCHSERIAL" ( "ItemCode",
	 "ItemName",
	 "WhsCode",
	 "BatchSeri",
	 "Qty",
	 "Type" ) AS ((SELECT
	 T0."ItemCode",
	 T2."ItemName",
	 T0."WhsCode",
	 T1."BatchNum" as "BatchSeri",
	 T1."Quantity" as "Qty",
	 'B' "Type" 
		FROM OITW T0 
		INNER JOIN OIBT T1 ON T0."WhsCode" = T1."WhsCode" 
		and T0."ItemCode" = T1."ItemCode" 
		Inner join OITM T2 on T0."ItemCode" = T2."ItemCode" 
		WHERE T0."OnHand" > 0 
		and T1."Quantity" >0 
		Group by T0."ItemCode",
	 T2."ItemName",
	 T0."WhsCode",
	 T1."BatchNum",
	 T1."Quantity",
	 T0."OnHand") 
	UNION ALL (SELECT
	 "A"."ItemCode" ,
	 "A"."ItemName" ,
	 "A"."WhsCode" ,
	 "A"."BatchSeri" ,
	 "A"."Qty" ,
	 "A"."Type" 
		from (select
	 "ItemCode",
	 "ItemName",
	 "WhsCode",
	 cast("SysSerial" as nvarchar(250)) as "BatchSeri",
	 1 "Qty",
	 'S' "Type" 
			from OSRI 
			where "Status"=0 
			order by "SysSerial" asc) a)) ;
# "UV_DETAILGHINHANSL" 
CREATE or replace VIEW "UV_DETAILGHINHANSL" ( "LSX",
	 "DocEntry",
	 "ItemChild",
	 "ChildName",
	 "SPDICH",
	 "NameSPDich",
	 "TO",
	 "TOTT",
	 "SanLuong",
	 "DaLam",
	 "Loi",
	 "CDay",
	 "CRong",
	 "CDai",
	 "ConLai",
	 "Allocate" ) AS SELECT
	 "A"."LSX" ,
	 "A"."DocEntry" ,
	 "A"."ItemChild" ,
	 "A"."ChildName" ,
	 "A"."SPDICH" ,
	 "A"."NameSPDich" ,
	 "A"."TO" ,
	 "A"."TOTT" ,
	 "A"."SanLuong" ,
	 "A"."DaLam" ,
	 "A"."Loi" ,
	 "A"."CDay" ,
	 "A"."CRong" ,
	 "A"."CDai" ,
	 a."SanLuong" - "DaLam"+ "Loi" "ConLai" ,
	 null "Allocate" 
from (select
	 a."U_GRID" "LSX",
	 a."DocEntry",
	 a."ItemCode" "ItemChild",
	 a."ProdName" "ChildName",
	 a."U_SPDICH" "SPDICH",
	 d."ItemName" "NameSPDich",
	 c."VisResCode" "TO",
	 a."U_Next" "TOTT",
	 sum(a."PlannedQty")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "SanLuong",
	 sum(a."U_QtyE")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "Loi",
	 sum(a."CmpltQty")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "DaLam",
 ifnull(xx."U_CDay",
	 '0') "CDay",
	 ifnull(xx."U_CRong",
	 '0') "CRong",
	 ifnull(xx."U_CDai",
	 '0') "CDai" 
	from OWOR A join WOR1 B ON A."DocEntry"=b."DocEntry" join ORSC c on b."ItemCode"=c."VisResCode" 
	and b."ItemType"=290 join OITM d on a."U_SPDICH"=d."ItemCode" join OITM xx on a."ItemCode"=xx."ItemCode"
	WHERE ifnull("U_GRID",
	 '')!='' 
	AND a."U_LLSX"='CBG' 
	and a."Status"='R') a 
where ( a."SanLuong" - "DaLam" +"Loi")>0 ;
# "UV_DETAILGHINHANSL_VCN"
CREATE or replace VIEW "UV_DETAILGHINHANSL_VCN" ( "LSX",
	 "ProType",
	 "Version",
	 "DocEntry",
	 "ItemChild",
	 "ChildName",
	 "SPDICH",
	 "NameSPDich",
	 "TO",
	 "TOTT",
	 "SanLuong",
	 "DaLam",
	 "Loi",
	 "CDay",
	 "CRong",
	 "CDai",
	 "ConLai",
	 "Allocate" ) AS SELECT
	 "A"."LSX" ,
	 "A"."ProType",
	 "A"."Version",
	 "A"."DocEntry" ,
	 "A"."ItemChild" ,
	 "A"."ChildName" ,
	 "A"."SPDICH" ,
	 "A"."NameSPDich" ,
	 "A"."TO" ,
	 "A"."TOTT" ,
	 "A"."SanLuong" ,
	 "A"."DaLam" ,
	 "A"."Loi" ,
	 "A"."CDay" ,
	 "A"."CRong" ,
	 "A"."CDai" ,
	 a."SanLuong" - "DaLam"- "Loi" "ConLai" ,
	 null "Allocate" 
from (select
	 a."U_GRID" "LSX",
	 a."U_IType" "ProType",
	 ifnull(a."U_Version",
	 1) "Version",
	 a."DocEntry",
	 a."ItemCode" "ItemChild",
	 a."ProdName" "ChildName",
	 a."U_SPDICH" "SPDICH",
	 d."ItemName" "NameSPDich",
	 c."VisResCode" "TO",
	 a."U_Next" "TOTT",
	 sum(a."PlannedQty")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "SanLuong",
	 sum(a."RjctQty")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "Loi",
	 sum(a."CmpltQty")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "DaLam",
	 ifnull(xx."U_CDay",
	 '0') "CDay",
	 ifnull(xx."U_CRong",
	 '0') "CRong",
	 ifnull(xx."U_CDai",
	 '0') "CDai" 
	from OWOR A join WOR1 B ON A."DocEntry"=b."DocEntry" join ORSC c on b."ItemCode"=c."VisResCode" 
	and b."ItemType"=290 join OITM d on a."U_SPDICH"=d."ItemCode" join OITM xx on a."ItemCode"=xx."ItemCode" 
	WHERE ifnull("U_GRID",
	 '')!='' 
	AND A."U_CDOAN"!='RO' 
	AND a."U_LLSX"='VCN' 
	and a."Status"='R') a 
where ( a."SanLuong" - "DaLam" -"Loi")>0 ;
# "UV_GHINHANSL"
CREATE or replace VIEW "UV_GHINHANSL" ( "LSX",
	 "ItemChild",
	 "ChildName",
	 "SPDICH",
	 "NameSPDich",
	 "MaThiTruong",
	 "TO",
	 "NameTO",
	 "TOTT",
	 "NameTOTT",
	 "SanLuong",
	 "DaLam",
	 "Loi",
	 "CDay",
	 "CRong",
	 "CDai",
	 "ConLai" ) AS SELECT
	 "A"."LSX" ,
	 "A"."ItemChild" ,
	 "A"."ChildName" ,
	 "A"."SPDICH" ,
	 "A"."NameSPDich" ,
	 "A"."MaThiTruong" ,
	 "A"."TO" ,
	 "A"."NameTO" ,
	 "A"."TOTT" ,
	 "A"."NameTOTT" ,
	 "A"."SanLuong" ,
	 "A"."DaLam" ,
	 "A"."Loi" ,
	 "A"."CDay" ,
	 "A"."CRong" ,
	 "A"."CDai" ,
	 a."SanLuong" - "DaLam" +"Loi" "ConLai" 
from (select
	 a."U_GRID" "LSX",
	 a."ItemCode" "ItemChild",
	 a."ProdName" "ChildName",
	 a."U_SPDICH" "SPDICH",
	 a."U_SKU" "MaThiTruong",
	 d."ItemName" "NameSPDich",
	 c."VisResCode" "TO",
	 c."ResName" "NameTO",
	 a."U_Next" "TOTT",
	 case when a."U_Next"='KHO' 
	then 'KHO' 
	else cc."ResName" 
	end "NameTOTT",
	 sum(a."PlannedQty")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "SanLuong",
	 sum(ifnull(a."CmpltQty",
	 0))over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "DaLam",
	 sum(ifnull(a."U_QtyE",
	 0))over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "Loi",
 ifnull(xx."U_CDay",
	 '0') "CDay",
	 ifnull(xx."U_CRong",
	 '0') "CRong",
	 ifnull(xx."U_CDai",
	 '0') "CDai" 
	from OWOR A join WOR1 B ON A."DocEntry"=b."DocEntry" join ORSC c on b."ItemCode"=c."VisResCode" 
	and b."ItemType"=290 join OITM d on a."U_SPDICH"=d."ItemCode" join OITM xx on a."ItemCode"=xx."ItemCode" 
	left join ORSC cc on a."U_Next"=cc."VisResCode" 

	WHERE ifnull("U_GRID",
	 '')!='' 
	and a."U_LLSX"='CBG' 
	and a."Status"='R') a 
where (a."SanLuong" - "DaLam" +"Loi") >0 ;
# "UV_GHINHANSLVCN"
CREATE or replace VIEW "UV_GHINHANSLVCN" ( "LSX",
	 "ProType",
	 "Version",
	 "ItemChild",
	 "ChildName",
	 "QuyCach2",
	 "SPDICH",
	 "NameSPDich",
	 "MaThiTruong",
	 "TO",
	 "NameTO",
	 "TOTT",
	 "NameTOTT",
	 "SanLuong",
	 "DaLam",
	 "Loi",
	 "CDay",
	 "CRong",
	 "CDai",
	 "CDOAN",
	 "ConLai" ) AS ((SELECT
	 "A"."LSX" ,
	 "A"."ProType",
	 a."Version",
	 a."ItemChild" ,
	 a."ChildName" ,
	 '' "QuyCach2",
	 a."SPDICH" ,
	 a."NameSPDich" ,
	 a."MaThiTruong" ,
	 a."TO" ,
	 a."NameTO" ,
	 a."TOTT" ,
	 a."NameTOTT" ,
	 a."SanLuong" ,
	 a."DaLam" ,
	 a."Loi" ,
	 a."CDay" ,
	 a."CRong" ,
	 a."CDai" ,
	 a."CDOAN",
	 a."SanLuong" - "DaLam" - "Loi" - "LoiCongDoanTruoc" "ConLai" 
		from (select
	 a."U_GRID" "LSX",
	 a."U_IType" "ProType",
	 ifnull(a."U_Version",
	 1) "Version",
	 a."ItemCode" "ItemChild",
	 a."ProdName" "ChildName",
	 '' "QuyCach2",
	 a."U_SPDICH" "SPDICH",
	 a."U_SKU" "MaThiTruong",
	 d."ItemName" "NameSPDich",
	 c."VisResCode" "TO",
	 c."ResName" "NameTO",
	 a."U_Next" "TOTT",
	 case when a."U_Next"='KHO' 
			then 'KHO' 
			else cc."ResName" 
			end "NameTOTT",
	 sum(a."PlannedQty")over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "SanLuong",
	 sum(ifnull(a."CmpltQty",
	 0))over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "DaLam",
	 sum(ifnull(a."RjctQty",
	 0))over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "Loi",
	 sum(ifnull(a."U_QtyE",
	 0))over (partition by a."U_GRID",
	 a."ItemCode",
	 c."VisResCode") "LoiCongDoanTruoc",
	 ifnull(xx."U_CDay",
	 '0') "CDay",
	 ifnull(xx."U_CRong",
	 '0') "CRong",
	 ifnull(xx."U_CDai",
	 '0') "CDai",
	 a."U_CDOAN" "CDOAN" 
			from OWOR A join WOR1 B ON A."DocEntry"=b."DocEntry" join ORSC c on b."ItemCode"=c."VisResCode" 
			and b."ItemType"=290 join OITM d on a."U_SPDICH"=d."ItemCode" join OITM xx on a."ItemCode"=xx."ItemCode" 
			left join ORSC cc on a."U_Next"=cc."VisResCode" 
			WHERE ifnull("U_GRID",
	 '')!='' 
			AND A."U_CDOAN"!='RO' 
			and a."U_LLSX"='VCN' 
			and a."Status"='R') a 
		where a."SanLuong" - "DaLam" - "Loi" - "LoiCongDoanTruoc" >0) 
	UNION ALL (SELECT
	 a."LSX" ,
	 a."ProType",
	 a."Version",
	 a."ItemChild" ,
	 a."ChildName" ,
	 a."QuyCach2" "QuyCach2",
	 a."SPDICH" ,
	 a."NameSPDich" ,
	 a."MaThiTruong" ,
	 a."TO" ,
	 a."NameTO" ,
	 a."TOTT" ,
	 a."NameTOTT" ,
	 a."SanLuong" ,
	 a."DaLam" ,
	 a."Loi" ,
	 a."CDay" ,
	 a."CRong" ,
	 a."CDai" ,
	 a."CDOAN",
	 a."SanLuong" - "DaLam" - "Loi" -"LoiCongDoanTruoc" "ConLai" 
		from (select
	 distinct a."U_GRID" "LSX",
	 a."U_IType" "ProType",
	 ifnull(a."U_Version",
	 1) "Version",
	 tf."ItemCode" "ItemChild",
	 tf."ItemName" "ChildName",
	 '(' || ff."QuyCach" || ')' as "QuyCach2",
	 a."U_SPDICH" "SPDICH",
	 a."U_SKU" "MaThiTruong",
	 d."ItemName" "NameSPDich",
	 c."VisResCode" "TO",
	 c."ResName" "NameTO",
	 a."U_Next" "TOTT",
	 case when a."U_Next"='KHO' 
			then 'KHO' 
			else cc."ResName" 
			end "NameTOTT",
	 0 "SanLuong",
	 0 "DaLam",
	 0 "Loi",
	 0 "LoiCongDoanTruoc",
	 ifnull(xx."U_CDay",
	 '0') "CDay",
	 ifnull(xx."U_CRong",
	 '0') "CRong",
	 ifnull(xx."U_CDai",
	 '0') "CDai",
	 a."U_CDOAN" "CDOAN" 
			from OWOR A join WOR1 B ON A."DocEntry"=b."DocEntry" join (SELECT
	 t0."DocEntry",
	 t1."ItemCode",
	 t1."ItemName" 
				from OWOR T0 JOIN WOR1 T1 ON T0."DocEntry"=t1."DocEntry" 
				and T1."BaseQty" > 0 
				and t1."ItemType"=4 
				and t0."Status"='R' 
				and T0."U_CDOAN"='RO') TF on a."DocEntry"=tf."DocEntry" join (select
	 b."DocEntry",
	 c."VisResCode",
	 c."ResName" 
				from wor1 b join ORSC c on b."ItemCode"=c."VisResCode" 
				and b."ItemType"=290 ) c on a."DocEntry"=c."DocEntry" join OITM d on a."U_SPDICH"=d."ItemCode" join OITM xx on a."ItemCode"=xx."ItemCode" join "UV_WEB_QUYCACHTPRONG" ff on a."DocEntry"=ff."DocEntry" 
			left join ORSC cc on a."U_Next"=cc."VisResCode" --LEFT JOIN -- COLLECT SL DA LAM
 
			WHERE ifnull("U_GRID",
	 '')!='' 
			AND A."U_CDOAN"='RO' 
			and a."U_LLSX"='VCN' 
			and a."Status"='R' 
			and b."BaseQty" <0 
			and B."PlannedQty"<b."IssuedQty" 
			and b."ItemType"='4') a));
# "UV_OHEM"         
CREATE or replace VIEW "UV_OHEM" ( "USER_CODE",
	 "NAME" ) AS select
	 "empID" USER_CODE,
	 "lastName"||' '|| "firstName" "NAME" 
from OHEM T0 
INNER JOIN OBPL T1 ON T0."BPLId" = T1."BPLId";
# "UV_SOLUONGTON"
CREATE or replace VIEW "UV_SOLUONGTON" ( "U_GRID",
	 "U_To",
	 "U_Next",
	 "U_CDOAN",
	 "U_SPDICH",
	 "ItemCode",
	 "ItemName",
	 "SubItemCode",
	 "SubItemName",
	 "wareHouse",
	 "OnHand",
	 "OnHandTo",
	 "BaseQty" ) AS SELECT
	 T40."U_GRID",
	 T40."U_To",
	 T40."U_Next",
	 T40."U_CDOAN",
	 T40."U_SPDICH",
	 T40."ItemCode",
	 T40."ItemName",
	 T40."SubItemCode",
	 T40."SubItemName",
	 T40."wareHouse",
	 T40."OnHand",
	 IFNULL(T41."Quantiy",
	 0) AS "OnHandTo",
	 T40."BaseQty" 
FROM ( SELECT
	 T0."U_GRID",
	 T0."U_To",
	 T0."U_Next",
	 T0."U_CDOAN",
	 T0."U_SPDICH",
	 T0."ItemCode" AS "ItemCode",
	 T0."ProdName" AS "ItemName",
	 T1."ItemCode" AS "SubItemCode",
	 T3."ItemName" AS "SubItemName",
	 T1."wareHouse",
	 SUM(T2."OnHand") AS "OnHand",
	 T1."BaseQty" 
	FROM "OWOR" T0 
	INNER JOIN "WOR1" T1 ON T0."DocEntry" = T1."DocEntry" 
	INNER JOIN "OITW" T2 ON T1."ItemCode" = T2."ItemCode" 
	INNER JOIN "OITM" T3 ON T1."ItemCode" = T3."ItemCode" 
	AND T1."wareHouse" = T2."WhsCode" 
	WHERE T3."ItmsGrpCod" IN (101,
	 102) 
	AND T0."Status"='R' 
	AND T0."PlannedQty" > T0."CmpltQty" 
	GROUP BY T0."U_GRID",
	 T0."U_To",
	 T0."U_Next",
	 T0."U_CDOAN",
	 T0."U_SPDICH",
	 T0."ItemCode",
	 T0."ProdName",
	 T1."ItemCode",
	 T3."ItemName",
	 T1."wareHouse",
	 "BaseQty" ) T40 
LEFT JOIN ( SELECT
	 "U_To",
	 "SubItemCode",
	 "SubItemName",
	 SUM("Quantiy") AS "Quantiy" 
	FROM ( SELECT
	 T02."U_To",
	 T01."ItemCode" AS "SubItemCode",
	 T01."Dscription" AS "SubItemName",
	 T01."Quantity" AS "Quantiy" 
		FROM "OIGN" T00 
		INNER JOIN "IGN1" T01 ON T00."DocEntry" = T01."DocEntry" 
		INNER JOIN "OWOR" T02 ON T01."BaseEntry" = T02."DocEntry" 
		AND T01."BaseType" = 202 
		WHERE T02."Status"='R' 
		AND T02."PlannedQty" > T02."CmpltQty" 
		UNION ALL SELECT
	 T02."U_To",
	 T01."ItemCode" AS "SubItemCode",
	 T01."Dscription" AS "SubItemName",
	 T01."Quantity" * (-1) AS "Quantiy" 
		FROM "OIGE" T00 
		INNER JOIN "IGE1" T01 ON T00."DocEntry" = T01."DocEntry" 
		INNER JOIN "OWOR" T02 ON T01."BaseEntry" = T02."DocEntry" 
		WHERE T02."Status"='R' 
		AND T02."PlannedQty" > T02."CmpltQty" 
		AND T01."BaseType" = 202 ) T10 
	GROUP BY T10."U_To",
	 T10."SubItemCode",
	 T10."SubItemName" ) T41 ON T40."U_To" = T41."U_To" 
AND T40."SubItemCode" = T41."SubItemCode";
# "UV_SOLUONGTONVCN"
CREATE or replace VIEW "UV_SOLUONGTONVCN" ( "U_GRID",
	 "U_To",
	 "U_Next",
	 "U_CDOAN",
	 "U_SPDICH",
	 "ItemCode",
	 "ItemName",
	 "ProdType",
	 "SubItemCode",
	 "SubItemName",
	 "wareHouse",
	 "OnHand",
	 "OnHandTo",
	 "BaseQty" ) AS SELECT
	 T40."U_GRID",
	 T40."U_To",
	 T40."U_Next",
	 T40."U_CDOAN",
	 T40."U_SPDICH",
	 T40."ItemCode",
	 T40."ItemName",
	 T40."ProdType",
	 T40."SubItemCode",
	 T40."SubItemName",
	 T40."wareHouse",
	 T40."OnHand",
	 IFNULL(T41."Quantiy",
	 0) AS "OnHandTo",
	 T40."BaseQty" 
FROM ( SELECT
	 T0."U_GRID",
	 T0."U_To",
	 T0."U_Next",
	 T0."U_CDOAN",
	 T0."U_SPDICH",
	 T0."ItemCode" AS "ItemCode",
	 T0."ProdName" AS "ItemName",
	 (SELECT
	 T4."U_IType" 
		FROM "OITM" T4 
		WHERE T4."ItemCode" = T0."ItemCode") AS "ProdType",
	 T1."ItemCode" AS "SubItemCode",
	 T3."ItemName" AS "SubItemName",
	 T1."wareHouse",
	 T2."OnHand" AS "OnHand",
	 SUM(T1."BaseQty") AS "BaseQty" 
	FROM "OWOR" T0 
	INNER JOIN "WOR1" T1 ON T0."DocEntry" = T1."DocEntry" 
	AND T0."U_CDOAN"<>'RO' 
	and T0."U_LLSX"='VCN' 
	INNER JOIN "OITW" T2 ON T1."ItemCode" = T2."ItemCode" 
	INNER JOIN "OITM" T3 ON T1."ItemCode" = T3."ItemCode" 
	AND T1."wareHouse" = T2."WhsCode" 
	WHERE T3."ItmsGrpCod" IN (111,
	 113,
	 116,
	 117,
	 118,
	 119,
	 120) 
	AND T0."Status"='R' 
	AND T0."PlannedQty" > T0."CmpltQty" 
	GROUP BY T0."U_GRID",
	 T0."U_To",
	 T0."U_Next",
	 T0."U_CDOAN",
	 T0."U_SPDICH",
	 T0."ItemCode",
	 T0."ProdName",
	 T1."ItemCode",
	 T2."OnHand",
	 T3."U_IType",
	 T3."ItemName",
	 T1."wareHouse",
	 "BaseQty" ) T40 
LEFT JOIN ( SELECT
	 "U_To",
	 "SubItemCode",
	 "SubItemName",
	 SUM("Quantiy") AS "Quantiy" 
	FROM ( SELECT
	 T02."U_To",
	 T01."ItemCode" AS "SubItemCode",
	 T01."Dscription" AS "SubItemName",
	 T01."Quantity" AS "Quantiy" 
		FROM "OIGN" T00 
		INNER JOIN "IGN1" T01 ON T00."DocEntry" = T01."DocEntry" 
		INNER JOIN "OWOR" T02 ON T01."BaseEntry" = T02."DocEntry" 
		AND T01."BaseType" = 202 
		AND T02."Status"='R' 
		AND T02."PlannedQty" > T02."CmpltQty" 
		AND T02."U_CDOAN"<>'RO' 
		and T02."U_LLSX"='VCN' 
		UNION ALL SELECT
	 T02."U_To",
	 T01."ItemCode" AS "SubItemCode",
	 T01."Dscription" AS "SubItemName",
	 T01."Quantity" * (-1) AS "Quantiy" 
		FROM "OIGE" T00 
		INNER JOIN "IGE1" T01 ON T00."DocEntry" = T01."DocEntry" 
		INNER JOIN "OWOR" T02 ON T01."BaseEntry" = T02."DocEntry" 
		AND T01."BaseType" = 202 
		AND T02."Status"='R' 
		AND T02."PlannedQty" > T02."CmpltQty" 
		AND T02."U_CDOAN"<>'RO' 
		and T02."U_LLSX"='VCN') T10 
	GROUP BY T10."U_To",
	 T10."SubItemCode",
	 T10."SubItemName" ) T41 ON T40."U_To" = T41."U_To" 
AND T40."SubItemCode" = T41."SubItemCode" ;
# "UV_WEB_QUYCACHTPRONG"
CREATE VIEW "UV_WEB_QUYCACHTPRONG" ( "DocEntry",
	 "QuyCach" ) AS SELECT
	 a."DocEntry",
	 STRING_AGG(c."QuyCach",
	 ' - ') AS "QuyCach" 
FROM OWOR a JOIN WOR1 b on a."DocEntry" = b."DocEntry" 
and b."BaseQty"<0 join (select
	 "ItemCode",
	 ifnull("U_CDay",
	 '0')||'x'||ifnull("U_CRong",
	 '0')||'x'||ifnull("U_CDai",
	 '0') "QuyCach" 
	from OITM ) c ON b."ItemCode"=c."ItemCode" 
GROUP BY a."DocEntry";
# V_WEBONHANDBYTO
CREATE or replace VIEW "V_WEBONHANDBYTO" ( "U_GRID",
	 "U_To",
	 "U_Next",
	 "U_CDOAN",
	 "FatherCode",
	 "FatherName",
	 "U_SPDICH",
	 "ItemCode",
	 "wareHouse",
	 "OnHand",
	 "OnHandTo" ) AS Select
	 T40."U_GRID",
	 T40."U_To",
	 T40."U_Next",
	 T40."U_CDOAN",
	 T40."FatherCode",
	 T40."FatherName",
	 T40."U_SPDICH",
	 T40."ItemCode",
	 T40."wareHouse",
	 T40."OnHand",
	 ifnull(T41."Quantiy",
	 0) "OnHandTo" 
From ( Select
	 T0."U_GRID",
	 T0."U_To",
	 T0."U_Next",
	 T0."U_CDOAN",
	 T0."ItemCode" "FatherCode",
	 T0."ProdName" "FatherName",
	 T0."U_SPDICH",
	 T1."ItemCode",
	 T1."wareHouse",
	 sum(T2."OnHand") "OnHand" 
	From "OWOR" T0 
	Inner Join "WOR1" T1 on T0."DocEntry" = T1."DocEntry" 
	Inner Join OITW T2 on T1."ItemCode" = T2."ItemCode" 
	And T1."wareHouse" = T2."WhsCode" 
	Group By T0."U_GRID",
	 T0."U_To",
	 T0."U_Next",
	 T0."U_CDOAN",
	 T0."ItemCode" ,
	 T0."ProdName" ,
	 T0."U_SPDICH",
	 T1."ItemCode",
	 T1."wareHouse" )T40 
Left Join ( Select
	 "U_To",
	 "ItemCode",
	 sum ("Quantiy") "Quantiy" 
	From ( Select
	 T02."U_To",
	 T01."ItemCode",
	 T01."Quantity" "Quantiy" 
		From OIGN T00 
		Inner Join IGN1 T01 on T00."DocEntry" = T01."DocEntry" 
		Inner Join OWOR T02 on T01."BaseEntry" = T02."DocEntry" 
		And T01."BaseType" = 202 
		Union ALL Select
	 T02."U_To",
	 T01."ItemCode",
	 T01."Quantity" * (-1) "Quantiy" 
		From OIGE T00 
		Inner Join IGE1 T01 on T00."DocEntry" = T01."DocEntry" 
		Inner Join OWOR T02 on T01."BaseEntry" = T02."DocEntry" 
		And T01."BaseType" = 202 )T10 
	Group By T10."U_To",
	 T10."ItemCode" )T41 on T40."U_To" = T41."U_To" 
And T40."ItemCode" = T41."ItemCode" WITH READ ONLY