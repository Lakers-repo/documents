## Window TVF Aggregate - Non-Optimized
- code
```SQL
SELECT 
	window_start, 
	window_end, 
	COUNT(*) AS order_num 
FROM TABLE(TUMBLE(TABLE orders, DESCRIPTOR(ts), INTERVAL '5' SECONDS)) GROUP BY window_start, window_end;
```

- plan
`WindowAggregate`
```Text
== Abstract Syntax Tree ==
LogicalAggregate(group=[{0, 1}], order_num=[COUNT()])
+- LogicalProject(window_start=[$4], window_end=[$5])
   +- LogicalTableFunctionScan(invocation=[TUMBLE($3, DESCRIPTOR($3), 5000:INTERVAL SECOND)], rowType=[RecordType(INTEGER user_id, VARCHAR(2147483647) product, INTEGER amount, TIMESTAMP_LTZ(3) *PROCTIME* ts, TIMESTAMP(3) window_start, TIMESTAMP(3) window_end, TIMESTAMP_LTZ(3) *PROCTIME* window_time)])
      +- LogicalProject(user_id=[$0], product=[$1], amount=[$2], ts=[PROCTIME()])
         +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount)]]])

== Optimized Physical Plan ==
Calc(select=[window_start, window_end, order_num], changelogMode=[I])
+- WindowAggregate(window=[TUMBLE(time_col=[ts], size=[5 s])], select=[COUNT(*) AS order_num, start('w$) AS window_start, end('w$) AS window_end], changelogMode=[I])
   +- Exchange(distribution=[single], changelogMode=[I])
      +- Calc(select=[PROCTIME() AS ts], changelogMode=[I])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: )]]], fields=[], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[window_start, window_end, order_num])
+- WindowAggregate(window=[TUMBLE(time_col=[ts], size=[5 s])], select=[COUNT(*) AS order_num, start('w$) AS window_start, end('w$) AS window_end])
   +- Exchange(distribution=[single])
      +- Calc(select=[PROCTIME() AS ts])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: )]]], fields=[])

```