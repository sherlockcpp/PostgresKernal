RelOptInfo::baserestrictinfo
代表以下SQL文的WHERE条件部分
```sql
select * from table1 where a = 2 and b = 1;

内部储存的时[ RestrictInfo*... ] 数组
```
```c
remove_useless_joins
	join_is_removable
		查询优化阶段，检查是否可以不生成多表连接类型的查询计划，
		比如某张表的某一列是unique类型，同时查询过滤条件中由对unique列类型的等值条件，
		则对该表的查询只可能返回一种值，可以不进行连接
```
```c
reduce_unique_semijoins
	rel_supports_distinctness
	优化条件和remove_useless_joins类似，某一张表只可能返回唯一值时优化删除掉连接结点
```

```c
create_distinct_paths
	创建使用distinct查询计划的结点
```

```c
create_xxx_paths
	创建xxx类型的查询计划结点
```

```c
set_cheapest
	利用之前一系列的create_xxx_path构建的树型结构，进行cost计算，查询优化
```