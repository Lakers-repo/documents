## SQL DDL
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
```
## Inner Join
### NoUniqueKey - 非笛卡尔积
- code
```SQL
insert into sink_table
select 
	l.order_id,
	l.user_id, 
	l.price  
from left_source_table l  
join right_source_table r 
on l.order_id = r.order_id;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalSink(table=[default_catalog.default_database.sink_table], fields=[order_id, user_id, price])
+- LogicalProject(order_id=[$0], user_id=[$1], price=[$2])
   +- LogicalJoin(condition=[=($0, $4)], joinType=[inner])
      :- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
      :  +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])
      +- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
         +- LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])

== Optimized Physical Plan ==
Sink(table=[default_catalog.default_database.sink_table], fields=[order_id, user_id, price], changelogMode=[NONE])
+- Calc(select=[order_id, user_id, price], changelogMode=[I])
   +- Join(joinType=[InnerJoin], where=[=(order_id, order_id0)], select=[order_id, user_id, price, order_id0], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey], changelogMode=[I])
      :- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      :  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
      +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
         +- Calc(select=[order_id], changelogMode=[I])
            +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
```
### NoUniqueKey - 笛卡尔积
- code
```SQL
insert into sink_table
select 
	l.order_id,
	l.user_id, 
	l.price  
from left_source_table l, right_source_table r;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2], order_id0=[$4])
+- LogicalJoin(condition=[true], joinType=[inner])
   :- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
   :  +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])
   +- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
      +- LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])

== Optimized Physical Plan ==
Join(joinType=[InnerJoin], where=[true], select=[order_id, user_id, price, order_id0], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey], changelogMode=[I])
:- Exchange(distribution=[single], changelogMode=[I])
:  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
+- Exchange(distribution=[single], changelogMode=[I])
   +- Calc(select=[order_id], changelogMode=[I])
      +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Join(joinType=[InnerJoin], where=[true], select=[order_id, user_id, price, order_id0], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey])
:- Exchange(distribution=[single])
:  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
+- Exchange(distribution=[single])
   +- Calc(select=[order_id])
      +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```
### JoinKeyContainsUniqueKey
- code
```SQL
-- view1
create view view1 as
select order_id, count(*) as cnt
from left_source_table 
group by order_id;

-- view2
create view view2 as 
select order_id, count(*) as cnt 
from right_source_table 
group by order_id;

-- query
select 
	l.order_id, 
	r.order_id, 
	l.cnt, 
	r.cnt 
from view1 l 
join view2 r 
on l.order_id = r.order_id;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], order_id0=[$2], cnt=[$1], cnt0=[$3])
+- LogicalJoin(condition=[=($0, $2)], joinType=[inner])
   :- LogicalAggregate(group=[{0}], cnt=[COUNT()])
   :  +- LogicalProject(order_id=[$0])
   :     +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])
   +- LogicalAggregate(group=[{0}], cnt=[COUNT()])
      +- LogicalProject(order_id=[$0])
         +- LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, order_id0, cnt, cnt0], changelogMode=[I,UA])
+- Join(joinType=[InnerJoin], where=[=(order_id, order_id0)], select=[order_id, cnt, order_id0, cnt0], leftInputSpec=[JoinKeyContainsUniqueKey], rightInputSpec=[JoinKeyContainsUniqueKey], changelogMode=[I,UA])
   :- Exchange(distribution=[hash[order_id]], changelogMode=[I,UA])
   :  +- GroupAggregate(groupBy=[order_id], select=[order_id, COUNT(*) AS cnt], changelogMode=[I,UA])
   :     +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
   :        +- Calc(select=[order_id], changelogMode=[I])
   :           +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I,UA])
      +- GroupAggregate(groupBy=[order_id], select=[order_id, COUNT(*) AS cnt], changelogMode=[I,UA])
         +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
            +- Calc(select=[order_id], changelogMode=[I])
               +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, order_id0, cnt, cnt0])
+- Join(joinType=[InnerJoin], where=[(order_id = order_id0)], select=[order_id, cnt, order_id0, cnt0], leftInputSpec=[JoinKeyContainsUniqueKey], rightInputSpec=[JoinKeyContainsUniqueKey])
   :- Exchange(distribution=[hash[order_id]])
   :  +- GroupAggregate(groupBy=[order_id], select=[order_id, COUNT(*) AS cnt])
   :     +- Exchange(distribution=[hash[order_id]])
   :        +- Calc(select=[order_id])
   :           +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
   +- Exchange(distribution=[hash[order_id]])
      +- GroupAggregate(groupBy=[order_id], select=[order_id, COUNT(*) AS cnt])
         +- Exchange(distribution=[hash[order_id]])
            +- Calc(select=[order_id])
               +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```
### HasUniqueKey
- code
```SQL
-- view1
create view view1 as
select order_id, user_id, count(*) as cnt
from left_source_table 
group by order_id, user_id;

-- view2
create view view2 as 
select order_id, count(*) as cnt 
from right_source_table 
group by order_id, user_id;

-- query
select 
	l.order_id, 
	r.order_id, 
	l.user_id,
	r.user_id
	l.cnt, 
	r.cnt 
from view1 l 
join view2 r 
on l.order_id = r.order_id;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], order_id0=[$3], user_id=[$1], user_id0=[$4], cnt=[$2], cnt0=[$5])
+- LogicalJoin(condition=[=($0, $3)], joinType=[inner])
   :- LogicalAggregate(group=[{0, 1}], cnt=[COUNT()])
   :  +- LogicalProject(order_id=[$0], user_id=[$1])
   :     +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])
   +- LogicalAggregate(group=[{0, 1}], cnt=[COUNT()])
      +- LogicalProject(order_id=[$0], user_id=[$1])
         +- LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, order_id0, user_id, user_id0, cnt, cnt0], changelogMode=[I,UA])
+- Join(joinType=[InnerJoin], where=[=(order_id, order_id0)], select=[order_id, user_id, cnt, order_id0, user_id0, cnt0], leftInputSpec=[HasUniqueKey], rightInputSpec=[HasUniqueKey], changelogMode=[I,UA])
   :- Exchange(distribution=[hash[order_id]], changelogMode=[I,UA])
   :  +- GroupAggregate(groupBy=[order_id, user_id], select=[order_id, user_id, COUNT(*) AS cnt], changelogMode=[I,UA])
   :     +- Exchange(distribution=[hash[order_id, user_id]], changelogMode=[I])
   :        +- Calc(select=[order_id, user_id], changelogMode=[I])
   :           +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I,UA])
      +- GroupAggregate(groupBy=[order_id, user_id], select=[order_id, user_id, COUNT(*) AS cnt], changelogMode=[I,UA])
         +- Exchange(distribution=[hash[order_id, user_id]], changelogMode=[I])
            +- Calc(select=[order_id, user_id], changelogMode=[I])
               +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, order_id0, user_id, user_id0, cnt, cnt0])
+- Join(joinType=[InnerJoin], where=[(order_id = order_id0)], select=[order_id, user_id, cnt, order_id0, user_id0, cnt0], leftInputSpec=[HasUniqueKey], rightInputSpec=[HasUniqueKey])
   :- Exchange(distribution=[hash[order_id]])
   :  +- GroupAggregate(groupBy=[order_id, user_id], select=[order_id, user_id, COUNT(*) AS cnt])
   :     +- Exchange(distribution=[hash[order_id, user_id]])
   :        +- Calc(select=[order_id, user_id])
   :           +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
   +- Exchange(distribution=[hash[order_id]])
      +- GroupAggregate(groupBy=[order_id, user_id], select=[order_id, user_id, COUNT(*) AS cnt])
         +- Exchange(distribution=[hash[order_id, user_id]])
            +- Calc(select=[order_id, user_id])
               +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```

## Left / Right Outer Join
- code
```SQL
select 
	l.order_id,
	l.user_id, 
	l.price  
from left_source_table l  
left join right_source_table r 
on l.order_id = r.order_id;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2])
+- LogicalJoin(condition=[=($0, $4)], joinType=[left])
   :- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
   :  +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])
   +- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
      +- LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, price], changelogMode=[I,UA,D])
+- Join(joinType=[LeftOuterJoin], where=[=(order_id, order_id0)], select=[order_id, user_id, price, order_id0], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey], changelogMode=[I,UA,D])
   :- Exchange(distribution=[hash[order_id]], changelogMode=[I])
   :  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, price])
+- Join(joinType=[LeftOuterJoin], where=[(order_id = order_id0)], select=[order_id, user_id, price, order_id0], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey])
   :- Exchange(distribution=[hash[order_id]])
   :  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id])
         +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```
## Full Outer Join
- code
```SQL
select 
	l.order_id,
	l.user_id, 
	l.price  
from left_source_table l  
full outer join right_source_table r 
on l.order_id = r.order_id;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2])
+- LogicalJoin(condition=[=($0, $4)], joinType=[full])
   :- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
   :  +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])
   +- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
      +- LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])

== Optimized Physical Plan ==
Calc(select=[order_id, user_id, price], changelogMode=[I,UA,D])
+- Join(joinType=[FullOuterJoin], where=[=(order_id, order_id0)], select=[order_id, user_id, price, order_id0], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey], changelogMode=[I,UA,D])
   :- Exchange(distribution=[hash[order_id]], changelogMode=[I])
   :  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
   +- Exchange(distribution=[hash[order_id]], changelogMode=[I])
      +- Calc(select=[order_id], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[order_id, user_id, price])
+- Join(joinType=[FullOuterJoin], where=[(order_id = order_id0)], select=[order_id, user_id, price, order_id0], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey])
   :- Exchange(distribution=[hash[order_id]])
   :  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
   +- Exchange(distribution=[hash[order_id]])
      +- Calc(select=[order_id])
         +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```
## Semi Join
- code
```SQL
select 
	order_id,
	user_id, 
	price  
from left_source_table  
where order_id in (
	select
		order_id 
	from right_source_table
)
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2])
+- LogicalFilter(condition=[IN($0, {
LogicalProject(order_id=[$0])
  LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])
})])
   +- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
      +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])

== Optimized Physical Plan ==
Join(joinType=[LeftSemiJoin], where=[=(order_id, order_id0)], select=[order_id, user_id, price], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey], changelogMode=[I])
:- Exchange(distribution=[hash[order_id]], changelogMode=[I])
:  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
+- Exchange(distribution=[hash[order_id]], changelogMode=[I])
   +- Calc(select=[order_id], changelogMode=[I])
      +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Join(joinType=[LeftSemiJoin], where=[(order_id = order_id0)], select=[order_id, user_id, price], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey])
:- Exchange(distribution=[hash[order_id]])
:  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
+- Exchange(distribution=[hash[order_id]])
   +- Calc(select=[order_id])
      +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```
## Anti Join
- code
```SQL
select 
	order_id,
	user_id, 
	price  
from left_source_table  
where order_id not in (
	select
		order_id 
	from right_source_table
)
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(order_id=[$0], user_id=[$1], price=[$2])
+- LogicalFilter(condition=[NOT(IN($0, {
LogicalProject(order_id=[$0])
  LogicalTableScan(table=[[default_catalog, default_database, right_source_table]])
}))])
   +- LogicalProject(order_id=[$0], user_id=[$1], price=[$2], proctime=[PROCTIME()])
      +- LogicalTableScan(table=[[default_catalog, default_database, left_source_table]])

== Optimized Physical Plan ==
Join(joinType=[LeftAntiJoin], where=[OR(IS NULL(order_id), IS NULL(order_id0), =(order_id, order_id0))], select=[order_id, user_id, price], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey], changelogMode=[I,UA,D])
:- Exchange(distribution=[single], changelogMode=[I])
:  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price], changelogMode=[I])
+- Exchange(distribution=[single], changelogMode=[I])
   +- Calc(select=[order_id], changelogMode=[I])
      +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price], changelogMode=[I])

== Optimized Execution Plan ==
Join(joinType=[LeftAntiJoin], where=[(order_id IS NULL OR order_id0 IS NULL OR (order_id = order_id0))], select=[order_id, user_id, price], leftInputSpec=[NoUniqueKey], rightInputSpec=[NoUniqueKey])
:- Exchange(distribution=[single])
:  +- TableSourceScan(table=[[default_catalog, default_database, left_source_table]], fields=[order_id, user_id, price])
+- Exchange(distribution=[single])
   +- Calc(select=[order_id])
      +- TableSourceScan(table=[[default_catalog, default_database, right_source_table]], fields=[order_id, user_id, price])
```