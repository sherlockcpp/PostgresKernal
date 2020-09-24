# HashSubPlan: store subselect result in an in-memory hash table


## EXAMPLE:
```sql
CREATE TABLE INT8_TBL(q1 int8, q2 int8);

INSERT INTO INT8_TBL VALUES('  123   ','  456');
INSERT INTO INT8_TBL VALUES('123   ','4567890123456789');
INSERT INTO INT8_TBL VALUES('4567890123456789','123');
INSERT INTO INT8_TBL VALUES(+4567890123456789,'4567890123456789');
INSERT INTO INT8_TBL VALUES('+4567890123456789','-4567890123456789');

create temp table inner_text (c1 text, c2 text);
insert into inner_text values ('a', null);
insert into inner_text values ('123', '456');

create function bogus_int8_text_eq(int8, text) returns boolean
language sql as 'select $1::text = $2';

select * from int8_tbl where q1 in (select c1 from inner_text);
```
## QUERY PLAN
```
           QUERY PLAN           
--------------------------------
 Seq Scan on int8_tbl
   Filter: (hashed SubPlan 1)
   SubPlan 1
     ->  Seq Scan on inner_text
```
## INNER

```C
ExecEvalSubPlan()
ExecAlternativeSubPlan()
	ExecSubPlan()
		ExecHashSubPlan()
		将SubPlan中的数据储存在hashtable中

		buildSubPlanHash()
		使用SubPlan父结点中源数据搜索hashtable
		如果源数据中不存在NULL数据，匹配hashtable
			在非空hashtable中匹配到数据时返回true
			其他情况返回false

		如果源数据中存在NULL数据只能返回FALSE
```

## RELATED COMMITS
- https://github.com/postgres/postgres/commit/5190707d7436089a33dd0e83482a314333ab6e59?branch=5190707d7436089a33dd0e83482a314333ab6e59&diff=split