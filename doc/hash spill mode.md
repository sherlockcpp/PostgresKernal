# Disk-based Hash Aggregation
## EXAMPLE:
```sql
create table agg_data_20k as
select g from generate_series(0, 19999) g;
analyze agg_data_20k;
set work_mem='64kB';
set enable_hashagg = true;
set enable_sort = false;
set jit_above_cost = 0;
explain (costs off)
select g%10000 as c1, sum(g::numeric) as c2, count(*) as c3
  from agg_data_20k group by g%10000;
```
## QUERY PLAN
```
           QUERY PLAN           
--------------------------------
 HashAggregate
   Group Key: (g % 10000)
   ->  Seq Scan on agg_data_20k
```

## INNER
```C
									
	1.agg_fill_hash_table()								
	遍历所有tuple							
									
	 ①fetch_input_tuple()								
	  ExecProcNode()								
	 执行agg的子查询计划获取tuple									
									
	 ②lookup_hash_entries()								
	 搜索①中取得tuple	对应的hashtable entries								
	 不存在对应entries的场合,同时hash_spill_mode生效的场合								
	  hashagg_spill_tuple()								
	  将数据分裂后暂时存储在磁盘								
	 不存在对应entries的场合,同时hash_spill_mode不生效的场合								
	  initialize_hash_entry()								
	   hash_agg_check_limits()								
	    hash_agg_enter_spill_mode()								
	  创建新的entries,同时估算内存使用量是否超出限制，
	  如果超出限制，设置hash_spill_mode生效								
									
	 ③hashagg_finish_initial_spills()								
	 如果磁盘中已经有分裂后的数据，将其放入batch中								
									
	2.agg_retrieve_hash_table								
	 ①agg_retrieve_hash_table_in_memory()								
	 循环取出内存中hashtable保存的所有数据								
	  finalize_aggregates()								
	  利用hashtable中数据开始聚集计算，并返回结果								
									
									
	 ②agg_refill_hash_table()								
	 当内存中hashtable已经没有数据的场合								
	 以batch为单位读取在步骤1.②中分裂后保存在磁盘中的数据,								
	 将磁盘中数据存入hashtable,进入步骤2.①循环读取数据
	 当磁盘中也没有需要的数据时结束聚合流程agg_done = true;	
```
## RELATED COMMITS
- https://github.com/postgres/postgres/commit/1f39bce021540fde00990af55b4432c55ef4b3c7
									
