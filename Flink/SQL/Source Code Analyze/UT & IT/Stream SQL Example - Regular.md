## Over Aggregate

> [!NOTE]
> ROW_NUMBER, RANK, DENSE_RANK不支持Bounded语法
> `Caused by: org.apache.calcite.sql.validate.SqlValidatorException: ROW/RANGE not allowed with RANK, DENSE_RANK or ROW_NUMBER functions`

> [!NOTE] 
> ROW_NUMBER, RANK, DENSE_RANK支持Unbounded语法,默认是基于ROWS的Interval,而SUM默认是基于RANK的Interval.
> 与时间属性没有关系.

### Bounded
#### Processing Time 
##### RANGE
- code
```SQL
select 
	order_id, 
	user_id, 
	price,
	SUM(price) OVER (PARTITION BY order_id ORDER BY proctime RANGE BETWEEN INTERVAL '30' SECONDS PRECEDING AND CURRENT ROW) AS row_num 
FROM source_table
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST RANGE 30000:INTERVAL SECOND PRECEDING), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST RANGE 30000:INTERVAL SECOND PRECEDING), null:BIGINT)])
+- LogicalTableScan(table=[[default_catalog, default_database, source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, price, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:BIGINT) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ RANG BETWEEN 30000 PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, price, CASE((w0$o0 > 0), w0$o1, null:BIGINT) AS row_num])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ RANG BETWEEN 30000 PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price])
```

##### ROWS
- code
```SQL
select 
	order_id, 
	user_id, 
	price,
	SUM(price) OVER (PARTITION BY order_id ORDER BY proctime ROWS BETWEEN 10 PRECEDING AND CURRENT ROW) AS row_num 
FROM source_table
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST ROWS 10 PRECEDING), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST ROWS 10 PRECEDING), null:BIGINT)])
+- LogicalTableScan(table=[[default_catalog, default_database, source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, price, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:BIGINT) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ ROWS BETWEEN 10 PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, price, CASE((w0$o0 > 0), w0$o1, null:BIGINT) AS row_num])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ ROWS BETWEEN 10 PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price])
```

#### Event Time
##### ##### RANGE
- code
```SQL
select 
	user_id, 
	product,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY ts RANGE BETWEEN INTERVAL '30' SECONDS PRECEDING AND CURRENT ROW) AS row_num 
FROM orders
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(user_id=[$0], product=[$1], amount=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST RANGE 30000:INTERVAL SECOND PRECEDING), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST RANGE 30000:INTERVAL SECOND PRECEDING), null:INTEGER)])
+- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
   +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
Calc(select=[user_id, product, amount, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:INTEGER) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ RANG BETWEEN 30000 PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[user_id]], changelogMode=[I])
      +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[user_id, product, amount, CASE((w0$o0 > 0), w0$o1, null:INTEGER) AS row_num])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ RANG BETWEEN 30000 PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1])
   +- Exchange(distribution=[hash[user_id]])
      +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts])
```

##### ROWS
- code
```SQL
select 
	user_id, 
	product,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY ts ROWS BETWEEN 10 PRECEDING AND CURRENT ROW) AS row_num 
FROM orders
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(user_id=[$0], product=[$1], amount=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST ROWS 10 PRECEDING), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST ROWS 10 PRECEDING), null:INTEGER)])
+- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
   +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
Calc(select=[user_id, product, amount, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:INTEGER) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ ROWS BETWEEN 10 PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[user_id]], changelogMode=[I])
      +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[user_id, product, amount, CASE((w0$o0 > 0), w0$o1, null:INTEGER) AS row_num])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ ROWS BETWEEN 10 PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1])
   +- Exchange(distribution=[hash[user_id]])
      +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts])
```



### Unbounded
#### Processing Time 
##### RANGE
- code
```SQL
select 
	order_id, 
	user_id, 
	price,
	SUM(price) OVER (PARTITION BY order_id ORDER BY proctime RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS row_num 
FROM source_table
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST), null:BIGINT)])
+- LogicalTableScan(table=[[default_catalog, default_database, source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, price, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:BIGINT) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ RANG BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, price, CASE((w0$o0 > 0), w0$o1, null:BIGINT) AS row_num])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ RANG BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price])
```

##### ROWS
- code
```SQL
select 
	order_id, 
	user_id, 
	price,
	SUM(price) OVER (PARTITION BY order_id ORDER BY proctime ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS row_num 
FROM source_table
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST ROWS UNBOUNDED PRECEDING), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY PROCTIME() NULLS FIRST ROWS UNBOUNDED PRECEDING), null:BIGINT)])
+- LogicalTableScan(table=[[default_catalog, default_database, source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, price, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:BIGINT) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, price, CASE((w0$o0 > 0), w0$o1, null:BIGINT) AS row_num])
+- OverAggregate(partitionBy=[order_id], orderBy=[$3 ASC], window=[ ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[order_id, user_id, price, $3, COUNT(price) AS w0$o0, $SUM0(price) AS w0$o1])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS $3])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price])
```

#### Event Time
##### ##### RANGE
- code
```SQL
select 
	user_id, 
	product,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY ts RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS row_num 
FROM orders
```

- plan
```Text
== Abstract Syntax Tree ==
== Abstract Syntax Tree ==
LogicalProject(user_id=[$0], product=[$1], amount=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST), null:INTEGER)])
+- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
   +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
Calc(select=[user_id, product, amount, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:INTEGER) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ RANG BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[user_id]], changelogMode=[I])
      +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[user_id, product, amount, CASE((w0$o0 > 0), w0$o1, null:INTEGER) AS row_num])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ RANG BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1])
   +- Exchange(distribution=[hash[user_id]])
      +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts])
```

##### ROWS
- code
```SQL
select 
	user_id, 
	product,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY ts ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS row_num 
FROM orders
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(user_id=[$0], product=[$1], amount=[$2], row_num=[CASE(>(COUNT($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST ROWS UNBOUNDED PRECEDING), 0), $SUM0($2) OVER (PARTITION BY $0 ORDER BY $3 NULLS FIRST ROWS UNBOUNDED PRECEDING), null:INTEGER)])
+- LogicalWatermarkAssigner(rowtime=[ts], watermark=[-($3, 3000:INTERVAL SECOND)])
   +- LogicalTableScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]])

== Optimized Physical Plan ==
Calc(select=[user_id, product, amount, CASE(>(w0$o0, 0:BIGINT), w0$o1, null:INTEGER) AS row_num], changelogMode=[I])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1], changelogMode=[I])
   +- Exchange(distribution=[hash[user_id]], changelogMode=[I])
      +- WatermarkAssigner(rowtime=[ts], watermark=[-(ts, 3000:INTERVAL SECOND)], changelogMode=[I])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[user_id, product, amount, CASE((w0$o0 > 0), w0$o1, null:INTEGER) AS row_num])
+- OverAggregate(partitionBy=[user_id], orderBy=[ts ASC], window=[ ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW], select=[user_id, product, amount, ts, COUNT(amount) AS w0$o0, $SUM0(amount) AS w0$o1])
   +- Exchange(distribution=[hash[user_id]])
      +- WatermarkAssigner(rowtime=[ts], watermark=[(ts - 3000:INTERVAL SECOND)])
         +- LegacyTableSourceScan(table=[[default_catalog, default_database, orders, source: [CsvTableSource(read fields: user_id, product, amount, ts)]]], fields=[user_id, product, amount, ts])
```

