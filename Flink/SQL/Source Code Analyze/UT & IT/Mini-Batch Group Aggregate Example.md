## Proctime
- code
```SQL
-- mini batch config
SET 'table.exec.mini-batch.enabled' = 'true';
SET 'table.exec.mini-batch.allow-latency' = '10s';
SET 'table.exec.mini-batch.size' = '5';

-- source ddl
CREATE TABLE source_table  
    order_id STRING  
    user_id STRING, 
	price BIGINT  
) WITH ( 
    'connector' = 'datagen',  
    'rows-per-second' = '10',
    'fields.order_id.length' = '1',
    'fields.user_id.length' = '10',
    'fields.price.min' = '1',
    'fields.price.max' = '1000000'
);

-- query
select
	user_id,
	count(order_id) as cnt
from source_table
group by user_id;
```
- plan
```Text
== Abstract Syntax Tree ==
LogicalAggregate(group=[{0}], cnt=[COUNT($1)])
+- LogicalProject(user_id=[$1], order_id=[$0])
   +- LogicalTableScan(table=[[default_catalog, default_database, source_table]])

== Optimized Physical Plan ==
GroupAggregate(groupBy=[user_id], select=[user_id, COUNT(order_id) AS cnt], changelogMode=[I,UA])
+- Exchange(distribution=[hash[user_id]], changelogMode=[I])
   +- Calc(select=[user_id, order_id], changelogMode=[I])
      +- MiniBatchAssigner(interval=[10000ms], mode=[ProcTime], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
GroupAggregate(groupBy=[user_id], select=[user_id, COUNT(order_id) AS cnt])
+- Exchange(distribution=[hash[user_id]])
   +- Calc(select=[user_id, order_id])
      +- MiniBatchAssigner(interval=[10000ms], mode=[ProcTime])
         +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price])
```

## Rowtime
- code
```SQL
-- mini batch config
SET 'table.exec.mini-batch.enabled' = 'true';
SET 'table.exec.mini-batch.allow-latency' = '10s';
SET 'table.exec.mini-batch.size' = '5';

-- source ddl
CREATE TABLE source_table  
    order_id STRING  
    user_id STRING, 
	price BIGINT,
	rowtime TIMESTAMP(3),
	WATERMARK FOR rowtime as rowtime  
) WITH ( 
    'connector' = 'datagen',  
    'rows-per-second' = '10',
    'fields.order_id.length' = '1',
    'fields.user_id.length' = '10',
    'fields.price.min' = '1',
    'fields.price.max' = '1000000'
);

-- query
select
	user_id,
	count(order_id) as cnt
from source_table
group by user_id;
```
- plan
```Text
== Abstract Syntax Tree ==
LogicalAggregate(group=[{0}], cnt=[COUNT($1)])
+- LogicalProject(user_id=[$1], order_id=[$0])
   +- LogicalWatermarkAssigner(rowtime=[rowtime], watermark=[$3])
      +- LogicalTableScan(table=[[default_catalog, default_database, source_table]])

== Optimized Physical Plan ==
GroupAggregate(groupBy=[user_id], select=[user_id, COUNT(order_id) AS cnt], changelogMode=[I,UA])
+- Exchange(distribution=[hash[user_id]], changelogMode=[I])
   +- Calc(select=[user_id, order_id], changelogMode=[I])
      +- MiniBatchAssigner(interval=[10000ms], mode=[ProcTime], changelogMode=[I])
         +- WatermarkAssigner(rowtime=[rowtime], watermark=[rowtime], changelogMode=[I])
            +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price, rowtime], changelogMode=[I])

== Optimized Execution Plan ==
GroupAggregate(groupBy=[user_id], select=[user_id, COUNT(order_id) AS cnt])
+- Exchange(distribution=[hash[user_id]])
   +- Calc(select=[user_id, order_id])
      +- MiniBatchAssigner(interval=[10000ms], mode=[ProcTime])
         +- WatermarkAssigner(rowtime=[rowtime], watermark=[rowtime])
            +- TableSourceScan(table=[[default_catalog, default_database, source_table]], fields=[order_id, user_id, price, rowtime])
```