## Inner Join
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