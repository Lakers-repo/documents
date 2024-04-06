## Window Group Aggregate(will Deprecated)

* code
```SQL
SELECT 
	CAST(TUMBLE_START(ts, INTERVAL '5' SECOND) AS STRING),
	window_start,
	COUNT(*) AS order_num 
FROM orders 
GROUP BY TUMBLE(ts, INTERVAL '5' SECOND);
```

- plan 
`GroupWindowAggregate`
```Text
== Abstract Syntax Tree ==
LogicalProject(window_start=[CAST(TUMBLE_START($0)):VARCHAR(2147483647) CHARACTER SET "UTF-16LE" NOT NULL], order_num=[$1])
+- LogicalAggregate(group=[{0}], order_num=[COUNT()])
   +- LogicalProject($f0=[$TUMBLE($3, 5000:INTERVAL SECOND)])
      +- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
         +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
Calc(select=[CAST(w$start AS VARCHAR(2147483647) CHARACTER SET "UTF-16LE" NOT NULL) AS window_start, order_num], changelogMode=[I])
+- GroupWindowAggregate(window=[TumblingGroupWindow('w$, ts, 5000)], properties=[w$start, w$end, w$rowtime, w$proctime], select=[COUNT(*) AS order_num, start('w$) AS w$start, end('w$) AS w$end, rowtime('w$) AS w$rowtime, proctime('w$) AS w$proctime], changelogMode=[I])
   +- Exchange(distribution=[single], changelogMode=[I])
      +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: ts)]]], fields=[ts], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[CAST(w$start AS VARCHAR(2147483647)) AS window_start, order_num])
+- GroupWindowAggregate(window=[TumblingGroupWindow('w$, ts, 5000)], properties=[w$start, w$end, w$rowtime, w$proctime], select=[COUNT(*) AS order_num, start('w$) AS w$start, end('w$) AS w$end, rowtime('w$) AS w$rowtime, proctime('w$) AS w$proctime])
   +- Exchange(distribution=[single])
      +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: ts)]]], fields=[ts])
```

## Window TopN
- code
```SQL
select 
	user_id,
	product,amount, 
	window_start,
	window_end 
FROM ( 
		SELECT 
			user_id, 
			product, 
			amount, 
			window_start, 
			window_end, 
			ROW_NUMBER() OVER (PARTITION BY window_start, window_end ORDER BY amount DESC) AS row_num  
        FROM TABLE(TUMBLE(TABLE orders, DESCRIPTOR(ts), INTERVAL '5' SECONDS))
    ) where row_num <= 3;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(user_id=[$0], product=[$1], amount=[$2], window_start=[$3], window_end=[$4])
+- LogicalFilter(condition=[<=($5, 3)])
   +- LogicalProject(user_id=[$0], product=[$1], amount=[$2], window_start=[$4], window_end=[$5], row_num=[ROW_NUMBER() OVER (PARTITION BY $4, $5 ORDER BY $2 DESC NULLS LAST)])
      +- LogicalTableFunctionScan(invocation=[TUMBLE($3, DESCRIPTOR($3), 5000:INTERVAL SECOND)], rowType=[RecordType(INTEGER user_id, VARCHAR(2147483647) product, INTEGER amount, TIMESTAMP(3) *ROWTIME* ts, TIMESTAMP(3) window_start, TIMESTAMP(3) window_end, TIMESTAMP(3) *ROWTIME* window_time)])
         +- LogicalProject(user_id=[$0], product=[$1], amount=[$2], ts=[$3])
            +- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
               +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
WindowRank(window=[TUMBLE(win_start=[window_start], win_end=[window_end], size=[5 s])], rankType=[ROW_NUMBER], rankRange=[rankStart=1, rankEnd=3], partitionBy=[], orderBy=[amount DESC], select=[user_id, product, amount, window_start, window_end], changelogMode=[I])
+- Exchange(distribution=[single], changelogMode=[I])
   +- Calc(select=[user_id, product, amount, window_start, window_end], changelogMode=[I])
      +- WindowTableFunction(window=[TUMBLE(time_col=[ts], size=[5 s])], changelogMode=[I])
         +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
            +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts], changelogMode=[I])

== Optimized Execution Plan ==
WindowRank(window=[TUMBLE(win_start=[window_start], win_end=[window_end], size=[5 s])], rankType=[ROW_NUMBER], rankRange=[rankStart=1, rankEnd=3], partitionBy=[], orderBy=[amount DESC], select=[user_id, product, amount, window_start, window_end])
+- Exchange(distribution=[single])
   +- Calc(select=[user_id, product, amount, window_start, window_end])
      +- WindowTableFunction(window=[TUMBLE(time_col=[ts], size=[5 s])])
         +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
            +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts])
```
## Window Deduplicate
- code
```SQL
select 
	user_id,
	product,amount, 
	window_start,
	window_end 
FROM ( 
		SELECT 
			user_id, 
			product, 
			amount, 
			window_start, 
			window_end, 
			ROW_NUMBER() OVER (PARTITION BY window_start, window_end ORDER BY ts DESC) AS row_num  
        FROM TABLE(TUMBLE(TABLE orders, DESCRIPTOR(ts), INTERVAL '5' SECONDS))
    ) where row_num <= 1;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(user_id=[$0], product=[$1], amount=[$2], window_start=[$3], window_end=[$4])
+- LogicalFilter(condition=[<=($5, 1)])
   +- LogicalProject(user_id=[$0], product=[$1], amount=[$2], window_start=[$4], window_end=[$5], row_num=[ROW_NUMBER() OVER (PARTITION BY $0, $4, $5 ORDER BY $3 DESC NULLS LAST)])
      +- LogicalTableFunctionScan(invocation=[TUMBLE($3, DESCRIPTOR($3), 5000:INTERVAL SECOND)], rowType=[RecordType(INTEGER user_id, VARCHAR(2147483647) product, INTEGER amount, TIMESTAMP(3) *ROWTIME* ts, TIMESTAMP(3) window_start, TIMESTAMP(3) window_end, TIMESTAMP(3) *ROWTIME* window_time)])
         +- LogicalProject(user_id=[$0], product=[$1], amount=[$2], ts=[$3])
            +- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
               +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
Calc(select=[user_id, product, amount, window_start, window_end], changelogMode=[I])
+- WindowDeduplicate(window=[TUMBLE(win_start=[window_start], win_end=[window_end], size=[5 s])], keep=[LastRow], partitionKeys=[user_id], orderKey=[ts], order=[ROWTIME], changelogMode=[I])
   +- Exchange(distribution=[hash[user_id]], changelogMode=[I])
      +- Calc(select=[user_id, product, amount, ts, window_start, window_end], changelogMode=[I])
         +- WindowTableFunction(window=[TUMBLE(time_col=[ts], size=[5 s])], changelogMode=[I])
            +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
               +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[user_id, product, amount, window_start, window_end])
+- WindowDeduplicate(window=[TUMBLE(win_start=[window_start], win_end=[window_end], size=[5 s])], keep=[LastRow], partitionKeys=[user_id], orderKey=[ts], order=[ROWTIME])
   +- Exchange(distribution=[hash[user_id]])
      +- Calc(select=[user_id, product, amount, ts, window_start, window_end])
         +- WindowTableFunction(window=[TUMBLE(time_col=[ts], size=[5 s])])
            +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
               +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts])
```
## Window TVF Aggregate - Optimized
- code
```SQL
SELECT 
	window_start, 
	window_end, 
	COUNT(*) AS order_num 
FROM TABLE(TUMBLE(TABLE orders, DESCRIPTOR(ts), INTERVAL '5' SECONDS)) GROUP BY window_start, window_end;
```

- plan
`LocalWindowAggregate`  `GlobalWindowAggregate`
```Text
== Abstract Syntax Tree ==
LogicalAggregate(group=[{0, 1}], order_num=[COUNT()])
+- LogicalProject(window_start=[$4], window_end=[$5])
   +- LogicalTableFunctionScan(invocation=[TUMBLE($3, DESCRIPTOR($3), 5000:INTERVAL SECOND)], rowType=[RecordType(INTEGER user_id, VARCHAR(2147483647) product, INTEGER amount, TIMESTAMP(3) *ROWTIME* ts, TIMESTAMP(3) window_start, TIMESTAMP(3) window_end, TIMESTAMP(3) *ROWTIME* window_time)])
      +- LogicalProject(user_id=[$0], product=[$1], amount=[$2], ts=[$3])
         +- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
            +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
Calc(select=[window_start, window_end, order_num], changelogMode=[I])
+- GlobalWindowAggregate(window=[TUMBLE(slice_end=[$slice_end], size=[5 s])], select=[COUNT(count1$0) AS order_num, start('w$) AS window_start, end('w$) AS window_end], changelogMode=[I])
   +- Exchange(distribution=[single], changelogMode=[I])
      +- LocalWindowAggregate(window=[TUMBLE(time_col=[ts], size=[5 s])], select=[COUNT(*) AS count1$0, slice_end('w$) AS $slice_end], changelogMode=[I])
         +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
            +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: ts)]]], fields=[ts], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[window_start, window_end, order_num])
+- GlobalWindowAggregate(window=[TUMBLE(slice_end=[$slice_end], size=[5 s])], select=[COUNT(count1$0) AS order_num, start('w$) AS window_start, end('w$) AS window_end])
   +- Exchange(distribution=[single])
      +- LocalWindowAggregate(window=[TUMBLE(time_col=[ts], size=[5 s])], select=[COUNT(*) AS count1$0, slice_end('w$) AS $slice_end])
         +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
            +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: ts)]]], fields=[ts])
```
---