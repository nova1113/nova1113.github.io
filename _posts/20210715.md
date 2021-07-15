최신 가격 정보를 가져오는 쿼리문을 개선해 봤다.

기존의 쿼리

```
with BASE as (select a.fac_cd, a.fr_dt, a.chg_dt, a.cust_cd, b.cust_nm, 
						a.itm_id, c.itm_bc, c.itm_cd, c.itm_nm, c.spec, a.um_bc, 
						c.grp1_cd, c.grp2_cd, c.grp3_cd, c.grp4_cd, c.src_bc,
						a.up, a.cury_bc, a.up_bc, 
						a.adm_yn, a.adm_uid, v.nm as adm_nm, a.adm_dt,
						a.rmks, a.mid, d.nm, a.mdt, a.cid, a.cdt
				from A a 
						left outer join B b on a.cust_cd = b.cust_cd
						left outer join C c on a.itm_id = c.itm_id
						left outer join D d on a.mid = d.reg_id
						left outer join V v on a.adm_uid = v.reg_id
						left outer join E e on e.fac_cd = a.fac_cd
				)
        
select co_cd = @co_cd, a.*
from 	BASE a
where '0' = @chk1

union all

select co_cd = @co_cd, a.*
from 	BASE a
where '1' = @chk1
and 	a.chg_dt =	(select top 1 a3.chg_dt from BASE a3
					where	a.itm_id = a3.itm_id  
                 and 	a.fac_cd = a3.fac_cd 
				 	and 	a.cust_cd = a3.cust_cd
					and 	a.um_bc = a3.um_bc
					and 	a.cury_bc = a3.cury_bc
				 	order by a3.chg_dt desc)
and 	a.fr_dt =	(select top 1 a2.fr_dt from BASE a2
                  where	a.itm_id = a2.itm_id  
                  and 	a.fac_cd = a2.fac_cd
					 and 	a.cust_cd = a2.cust_cd
					 and 	a.um_bc = a2.um_bc
					 and 	a.cury_bc = a2.cury_bc
				 	 and 	a.chg_dt = a2.chg_dt
					 order by a2.fr_dt desc)
order by fac_cd, cust_nm, fr_dt desc
```
일단 컨셉을 이해 하자면 @chk1 값이 0일땐 제품, 거래처별 단가를 전부 다 가져오는 것이고, 
@chk1 값이 1일 땐 제품, 거래처, 공장별 최신의 단가를 가져오는 쿼리이다.

위 쿼리를 보고선 하단의 
```where '1' = @chk1
and 	a.chg_dt =	(select top 1 a3.chg_dt from BASE a3
					where	a.itm_id = a3.itm_id  
                 and 	a.fac_cd = a3.fac_cd 
				 	and 	a.cust_cd = a3.cust_cd
					and 	a.um_bc = a3.um_bc
					and 	a.cury_bc = a3.cury_bc
				 	order by a3.chg_dt desc)
and 	a.fr_dt =	(select top 1 a2.fr_dt from BASE a2
                  where	a.itm_id = a2.itm_id   
                  and 	a.fac_cd = a2.fac_cd
					 and 	a.cust_cd = a2.cust_cd
					 and 	a.um_bc = a2.um_bc
					 and 	a.cury_bc = a2.cury_bc
				 	 and 	a.chg_dt = a2.chg_dt
					 order by a2.fr_dt desc)
order by fac_cd, cust_nm, fr_dt desc
```
이 부분을 이해하기가 너무 어려웠고, 솔직히 지금도 잘 모르겠다.
대충 생각하면 협의 일자와, 시작일자가 최신인 데이터와 조인해서 가져오는 쿼리로 보인다.

다음으로 @chk1은 0 혹은 1의 이진 값이 들어온다.

즉, 0이면서 1인 경우는 없다는 말인데, Union All로 합집합을 쿼리 할 이유가 없어 보인다.
```
select co_cd = @co_cd, a.*
from 	BASE a
where '0' = @chk1

union all

select co_cd = @co_cd, a.*
from 	BASE a
where '1' = @chk1
```

비효율 적이라 생각 된 부분을 수정 해 봤다.

```
with BASE as (select a.fac_cd, a.fr_dt, a.chg_dt, a.cust_cd, b.cust_nm, 
						a.itm_id, c.itm_bc, c.itm_cd, c.itm_nm, c.spec, a.um_bc, 
						c.grp1_cd, c.grp2_cd, c.grp3_cd, c.grp4_cd, c.src_bc,
						a.up, a.cury_bc, a.up_bc, 
						a.adm_yn, a.adm_uid, v.nm as adm_nm, a.adm_dt,
						a.rmks, a.mid, d.nm, a.mdt, a.cid, a.cdt
				from A a 
						left outer join B b on a.cust_cd = b.cust_cd
						left outer join C c on a.itm_id = c.itm_id
						left outer join D d on a.mid = d.reg_id
						left outer join V v on a.adm_uid = v.reg_id
						left outer join E e on e.fac_cd = a.fac_cd
)


SELECT *
FROM(
	SELECT ROW_NUMBER() OVER (PARTITION BY A.fac_cd, A.cust_cd, A.itm_id, A.um_bc, A.cury_bc ORDER BY fr_dt DESC, chg_dt DESC) as rowNum,
		A.*
	FROM BASE A
) T
WHERE (@chk1 = 1 AND T.rowNum = 1) OR
	(@chk1 = 0)
```
창고, 거래처, 품목, 단위, 통화별로 협의일자와 시작일자를 내림차순으로 정렬하면 가장 상단으로 정렬된 데이터가 최신의 가격 정보로 이해 할 수 있다.
따라서, RowNumber를 붙이고, RowNumber가 1인 행을 불러 오는 쿼리로 변경해 본 것이다.

이 쿼리의 문제는 @chk1가 0인 경우 즉, 최신단가만이 아니라 모든 단가 정보를 다 가져올 때에도 필요 없는 RowNumber를 붙인다는 것이다.

그래서 쿼리를 수정해 봤다.
```
with BASE as (select a.fac_cd, a.fr_dt, a.chg_dt, a.cust_cd, b.cust_nm, 
						a.itm_id, c.itm_bc, c.itm_cd, c.itm_nm, c.spec, a.um_bc, 
						c.grp1_cd, c.grp2_cd, c.grp3_cd, c.grp4_cd, c.src_bc,
						a.up, a.cury_bc, a.up_bc, 
						a.adm_yn, a.adm_uid, v.nm as adm_nm, a.adm_dt,
						a.rmks, a.mid, d.nm, a.mdt, a.cid, a.cdt
				from A a 
						left outer join B b on a.cust_cd = b.cust_cd
						left outer join C c on a.itm_id = c.itm_id
						left outer join D d on a.mid = d.reg_id
						left outer join V v on a.adm_uid = v.reg_id
						left outer join E e on e.fac_cd = a.fac_cd
)

IF(@chk1 = '0')
	SELECT *
	FROM BASE A

ELSE IF(@chk1 = '1')
	SELECT *
	FROM(
		SELECT ROW_NUMBER() OVER (PARTITION BY A.fac_cd, A.cust_cd, A.itm_id, A.um_bc, A.cury_bc ORDER BY fr_dt DESC, chg_dt DESC) as rowNum,
		A.*
		FROM BASE A
	) T
	WHERE T.rowNum = 1
  ```
그냥 단순히 IF문을 추가한 것인데 문제가 생겼다.
IF근처의 구문오류가 발생한 것

아직까지 이유는 잘 모르겠고, 새로운 과제가 하나 생겨 버렸다.
