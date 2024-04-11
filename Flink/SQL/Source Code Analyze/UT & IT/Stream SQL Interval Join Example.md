## ProcTime
- code
```SQL
-- left table ddl
CREATE TABLE left_source_table  
    order_id STRING  
    user_id STRING, 
	price BIGINT, 
    proctime AS PROCTIME()  
) WITH ( 
    'connector' = 'datagen',  
    'rows-per-second' = '10',
    'fields.order_id.length' = '1',
    'fields.user_id.length' = '10',
    'fields.price.min' = '1',
    'fields.price.max' = '1000000'
);

-- right table ddl
CREATE TABLE right_source_table  
    order_id STRING  
    user_id STRING, 
	price BIGINT, 
    proctime AS PROCTIME()  
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
	l.order_id, 
	l.user_id, 
	r.order_id, 
	r.user_id, 
	l.price, 
	r.price  
from left_source_table l, right_source_table r  
where l.order_id = r.order_id  
and l.proctime BETWEEN r.proctime - INTERVAL '10' SECOND AND r.proctime + INTERVAL '5' SECOND;
```

- plan
```Text
Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], order_id0=[$4], user_id0=[$5], price=[$2], price0=[$6])
+- LogicalFilter(condition=[AND(=($0, $4), >=($3, -($7, 10000:INTERVAL SECOND)), <=($3, +($7, 5000:INTERVAL SECOND)))])
   +- LogicalJoin(condition=[true], joinType=[inner])
      :- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
      :  +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])
      +- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
         +- LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, order_id0, user_id0, price, price0], changelogMode=[I])
+- IntervalJoin(joinType=[InnerJoin], windowBounds=[isRowTime=false, leftLowerBound=-10000, leftUpperBound=5000, leftTimeIndex=3, rightTimeIndex=3], where=[AND(=(order_id, order_id0), >=(proctime, -(proctime0, 10000:INTERVAL SECOND)), <=(proctime, +(proctime0, 5000:INTERVAL SECOND)))], select=[order_id, user_id, price, proctime, order_id0, user_id0, price0, proctime0], changelogMode=[I])
   :- Exchange(distribution=[hash[order_id]], changelogMode=[I])
   :  +- Calc(select=[order_id, user_id, price, PROCTIME() AS proctime], changelogMode=[I])
   :     +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS proctime], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, order_id0, user_id0, price, price0])
+- IntervalJoin(joinType=[InnerJoin], windowBounds=[isRowTime=false, leftLowerBound=-10000, leftUpperBound=5000, leftTimeIndex=3, rightTimeIndex=3], where=[((order_id = order_id0) AND (proctime >= (proctime0 - 10000:INTERVAL SECOND)) AND (proctime <= (proctime0 + 5000:INTERVAL SECOND)))], select=[order_id, user_id, price, proctime, order_id0, user_id0, price0, proctime0])
   :- Exchange(distribution=[hash[order_id]])
   :  +- Calc(select=[order_id, user_id, price, PROCTIME() AS proctime])
   :     +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id, user_id, price, PROCTIME() AS proctime])
         +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```