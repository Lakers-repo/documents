## Over Aggregate
- code
```SQL
select 
	order_id, 
	user_id, 
	ROW_NUMBER() OVER (PARTITION BY order_id,user_id ORDER BY proctime) AS row_num 
FROM source_table
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], row_num=[ROW_NUMBER() OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST)])
+- LogicalTableScan(table=[[default_catalog, default_database, source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, w0$o0 AS $2], changelogMode=[I])
+- OverAggregate(partitionBy=[order_id], orderBy=[$2 ASC], window=[ ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[order_id, user_id, $2, ROW_NUMBER(*) AS w0$o0], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id, user_id, PROCTIME() AS $2], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, w0$o0 AS $2])
+- OverAggregate(partitionBy=[order_id], orderBy=[$2 ASC], window=[ ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[order_id, user_id, $2, ROW_NUMBER(*) AS w0$o0])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id, user_id, PROCTIME() AS $2])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price])
```