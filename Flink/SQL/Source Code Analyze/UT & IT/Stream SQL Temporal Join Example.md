## Temporal Table Join
### RowTime
- code
```SQL
-- fact table ddl
drop table if exists transactions;
create table if not exists transactions (
    id string,
    currency_code string,
    total decimal(10,2),
    transaction_time timestamp(3),
    -- 定义事件时间属性 
    watermark for `transaction_time` as transaction_time - interval '15' second 
) with (
    'connector' = 'datagen',
    'rows-per-second' = '100'
);

-- dim table ddl
drop table if exists currency_rates;
-- 时态表（版本表）条件1：定义主键
-- 时态表（版本表）条件2：定义事件时间属性
create table if not exists currency_rates (
    currency_code string,
    eur_rate decimal(6,4),
    rate_time timestamp(3),
    primary key (currency_code) not enforced, 
    watermark for `rate_time` as rate_time - interval '15' second 
)
with (
    'connector' = 'datagen',
    'rows-per-second' = '100'
);

-- query
-- 使用 Temporal Table DDL + FOR SYSTEM_TIME AS OF 关键字实现 Temporal Join
-- 在时态维度上关联时指定的是事实表的事件时间（transaction_time）
select
    t.id,
    t.total * c.eur_rate as total_eur,
    t.total,
    c.currency_code,
    t.transaction_time
from 
	transactions t
left join
    currency_rates for system_time as of t.transaction_time as c
on
    t.currency_code = c.currency_code;
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(id=[$0], total_eur=[*($2, $5)], total=[$2], currency_code=[$4], transaction_time=[$3])
+- LogicalCorrelate(correlation=[$cor0], joinType=[left], requiredColumns=[{1, 3}])
   :- LogicalWatermarkAssigner(rowtime=[transaction_time], watermark=[-($3, 15000:INTERVAL SECOND)])
   :  +- LogicalTableScan(table=[[default_catalog, default_database, transactions]])
   +- LogicalFilter(condition=[=($cor0.currency_code, $0)])
      +- LogicalSnapshot(period=[$cor0.transaction_time])
         +- LogicalWatermarkAssigner(rowtime=[rate_time], watermark=[-($2, 15000:INTERVAL SECOND)])
            +- LogicalTableScan(table=[[default_catalog, default_database, currency_rates]])

== Optimized Physical Plan ==
Calc(select=[id, *(total, eur_rate) AS total_eur, total, currency_code0 AS currency_code, transaction_time], changelogMode=[I])
+- TemporalJoin(joinType=[LeftOuterJoin], where=[AND(=(currency_code, currency_code0), __TEMPORAL_JOIN_CONDITION(transaction_time, rate_time, __TEMPORAL_JOIN_CONDITION_PRIMARY_KEY(currency_code0), __TEMPORAL_JOIN_LEFT_KEY(currency_code), __TEMPORAL_JOIN_RIGHT_KEY(currency_code0)))], select=[id, currency_code, total, transaction_time, currency_code0, eur_rate, rate_time], changelogMode=[I])
   :- Exchange(distribution=[hash[currency_code]], changelogMode=[I])
   :  +- WatermarkAssigner(rowtime=[transaction_time], watermark=[-(transaction_time, 15000:INTERVAL SECOND)], changelogMode=[I])
   :     +- TableSourceScan(table=[[default_catalog, default_database, transactions]], fields=[id, currency_code, total, transaction_time], changelogMode=[I])
   +- Exchange(distribution=[hash[currency_code]], changelogMode=[I])
      +- WatermarkAssigner(rowtime=[rate_time], watermark=[-(rate_time, 15000:INTERVAL SECOND)], changelogMode=[I])
         +- TableSourceScan(table=[[default_catalog, default_database, currency_rates]], fields=[currency_code, eur_rate, rate_time], changelogMode=[I])

== Optimized Execution Plan ==
Calc(select=[id, (total * eur_rate) AS total_eur, total, currency_code0 AS currency_code, transaction_time])
+- TemporalJoin(joinType=[LeftOuterJoin], where=[((currency_code = currency_code0) AND __TEMPORAL_JOIN_CONDITION(transaction_time, rate_time, __TEMPORAL_JOIN_CONDITION_PRIMARY_KEY(currency_code0), __TEMPORAL_JOIN_LEFT_KEY(currency_code), __TEMPORAL_JOIN_RIGHT_KEY(currency_code0)))], select=[id, currency_code, total, transaction_time, currency_code0, eur_rate, rate_time])
   :- Exchange(distribution=[hash[currency_code]])
   :  +- WatermarkAssigner(rowtime=[transaction_time], watermark=[(transaction_time - 15000:INTERVAL SECOND)])
   :     +- TableSourceScan(table=[[default_catalog, default_database, transactions]], fields=[id, currency_code, total, transaction_time])
   +- Exchange(distribution=[hash[currency_code]])
      +- WatermarkAssigner(rowtime=[rate_time], watermark=[(rate_time - 15000:INTERVAL SECOND)])
         +- TableSourceScan(table=[[default_catalog, default_database, currency_rates]], fields=[currency_code, eur_rate, rate_time])
```
### ProcTime
- code
```SQL
-- fact table ddl
drop table if exists transactions;
create table if not exists transactions (
    id string,
    currency_code string,
    total decimal(10,2),
    proctime as proctime()
) with (
    'connector' = 'datagen',
    'rows-per-second' = '100'
);

-- dim table ddl
drop table if exists currency_rates;
-- 时态表（版本表 - latest version）条件1：定义主键
-- 时态表（版本表 - latest version）条件2：定义处理时间属性,必须有
create table if not exists currency_rates (
    currency_code string,
    eur_rate decimal(6,4),
    proc_time as proctime(),
    primary key (currency_code) not enforced
    -- watermark for `rate_time` as rate_time - interval '15' second 
)
with (
    'connector' = 'datagen',
    'rows-per-second' = '100'
);

-- query
select
    t.id,
    t.total * c.eur_rate as total_eur,
    t.total,
    c.currency_code,
    t.proc_time
from 
	transactions t
left join
    currency_rates for system_time as of t.proc_time as c
on
    t.currency_code = c.currency_code;
```

#### Error Cases
##### 右表没有定义主键
```Java
Exception in thread "main" org.apache.flink.table.api.ValidationException: Temporal Table Join requires primary key in versioned table, but no primary key can be found.
```

##### Temporal Join DDL 不支持processing time
```Java
Exception in thread "main" org.apache.flink.table.api.TableException: Processing-time temporal join is not supported yet.
```
## Temporal Table Function
### RowTime
- code
```Java
package org.apache.flink.table.examples.java.basics;  
  
import org.apache.flink.table.api.EnvironmentSettings;  
import org.apache.flink.table.api.TableEnvironment;  
import org.apache.flink.table.functions.TemporalTableFunction;  
  
import static org.apache.flink.table.api.Expressions.$;  
  
/**  
 * @author haxi  
 * @description  
 * @date 2024/4/11 15:03  
 */public class TemporalTableFunctionRowTimeExample {  
    public static void main(String[] args) {  
  
        EnvironmentSettings settings = EnvironmentSettings  
                .newInstance()  
                .inStreamingMode()  
                .build();  
        TableEnvironment tableEnv = TableEnvironment.create(settings);  
  
        // 事实表: transactions  
        tableEnv  
                .executeSql(  
                        "create temporary table transactions (\n" +  
                                "    id string,\n" +  
                                "    currency_code string,\n" +  
                                "    total decimal(10,2),\n" +  
                                "    transaction_time timestamp(3), \n" +  
                                "    watermark for `transaction_time` as transaction_time - interval '15' second \n"  
                                +  
                                ") with (\n" +  
                                "    'connector' = 'datagen',\n" +  
                                "    'rows-per-second' = '100'\n" +  
                                ")")  
                .print();  
  
        // 维表: currency_rates 
        // 可以没有主键 
        tableEnv  
                .executeSql(  
                        "create temporary table currency_rates (\n" +  
                                "    currency_code string,\n" +  
                                "    eur_rate decimal(6,4),\n" +  
                                "    rate_time timestamp(3),\n" +  
                                "    primary key (currency_code) not enforced,\n"  
                                +  
                                "    watermark for `rate_time` as rate_time - interval '15' second \n"  
                                +  
                                ")\n" +  
                                "with (\n" +  
                                "    'connector' = 'datagen',\n" +  
                                "    'rows-per-second' = '100'\n" +  
                                ");")  
                .print();  
  
        // 注册 Temporal Table Function        
        TemporalTableFunction rates = tableEnv  
                .from("currency_rates")  
                .createTemporalTableFunction($("rate_time"), $("currency_code"));  
  
        tableEnv.createTemporarySystemFunction("rates", rates);  
  
        String query = "select\n" +  
                "    t.id,\n" +  
                "    t.total * c.eur_rate as total_eur,\n" +  
                "    t.total,\n" +  
                "    c.currency_code,\n" +  
                "    t.transaction_time\n" +  
                "from\n" +  
                "    transactions t,\n" +  
                "    lateral table (rates(transaction_time)) as c\n" +  
                "where\n" +  
                "    t.currency_code = c.currency_code";  
        System.out.println(tableEnv.explainSql(query));  
  
        // 使用 Temporal Table Function + LATERAL TABLE 关键字实现 Temporal Join        
        // 在时态维度上关联时指定的是事实表的事件时间（transaction_time）  
		tableEnv.executeSql(query).print(); 
    } 
}
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(id=[$0], total_eur=[*($2, $5)], total=[$2], currency_code=[$4], transaction_time=[$3])
+- LogicalFilter(condition=[=($1, $4)])
   +- LogicalCorrelate(correlation=[$cor0], joinType=[inner], requiredColumns=[{3}])
      :- LogicalWatermarkAssigner(rowtime=[transaction_time], watermark=[-($3, 15000:INTERVAL SECOND)])
      :  +- LogicalTableScan(table=[[default_catalog, default_database, transactions]])
      +- LogicalTableFunctionScan(invocation=[rates($cor0.transaction_time)], rowType=[RecordType:peek_no_expand(VARCHAR(2147483647) currency_code, DECIMAL(6, 4) eur_rate, TIMESTAMP(3) *ROWTIME* rate_time)])

== Optimized Physical Plan ==
Calc(select=[id, *(total, eur_rate) AS total_eur, total, currency_code0 AS currency_code, transaction_time])
+- TemporalJoin(joinType=[InnerJoin], where=[AND(__TEMPORAL_JOIN_CONDITION(transaction_time, rate_time, __TEMPORAL_JOIN_CONDITION_PRIMARY_KEY(currency_code0)), =(currency_code, currency_code0))], select=[id, currency_code, total, transaction_time, currency_code0, eur_rate, rate_time])
   :- Exchange(distribution=[hash[currency_code]])
   :  +- WatermarkAssigner(rowtime=[transaction_time], watermark=[-(transaction_time, 15000:INTERVAL SECOND)])
   :     +- TableSourceScan(table=[[default_catalog, default_database, transactions]], fields=[id, currency_code, total, transaction_time])
   +- Exchange(distribution=[hash[currency_code]])
      +- WatermarkAssigner(rowtime=[rate_time], watermark=[-(rate_time, 15000:INTERVAL SECOND)])
         +- TableSourceScan(table=[[default_catalog, default_database, currency_rates]], fields=[currency_code, eur_rate, rate_time])

== Optimized Execution Plan ==
Calc(select=[id, (total * eur_rate) AS total_eur, total, currency_code0 AS currency_code, transaction_time])
+- TemporalJoin(joinType=[InnerJoin], where=[(__TEMPORAL_JOIN_CONDITION(transaction_time, rate_time, __TEMPORAL_JOIN_CONDITION_PRIMARY_KEY(currency_code0)) AND (currency_code = currency_code0))], select=[id, currency_code, total, transaction_time, currency_code0, eur_rate, rate_time])
   :- Exchange(distribution=[hash[currency_code]])
   :  +- WatermarkAssigner(rowtime=[transaction_time], watermark=[(transaction_time - 15000:INTERVAL SECOND)])
   :     +- TableSourceScan(table=[[default_catalog, default_database, transactions]], fields=[id, currency_code, total, transaction_time])
   +- Exchange(distribution=[hash[currency_code]])
      +- WatermarkAssigner(rowtime=[rate_time], watermark=[(rate_time - 15000:INTERVAL SECOND)])
         +- TableSourceScan(table=[[default_catalog, default_database, currency_rates]], fields=[currency_code, eur_rate, rate_time])
```
### ProcTime
- code
```Java
package org.apache.flink.table.examples.java.basics;  
  
import org.apache.flink.table.api.EnvironmentSettings;  
import org.apache.flink.table.api.TableEnvironment;  
import org.apache.flink.table.functions.TemporalTableFunction;  
  
import static org.apache.flink.table.api.Expressions.$;  
  
/**  
 * @author haxi  
 * @description  
 * @date 2024/4/11 15:11  
 */public class TemporalTableFunctionProcTimeExample {  
    public static void main(String[] args) {  
  
        EnvironmentSettings settings = EnvironmentSettings  
                .newInstance()  
                .inStreamingMode()  
                .build();  
        TableEnvironment tableEnv = TableEnvironment.create(settings);  
  
        // 事实表: transactions  
        tableEnv  
                .executeSql(  
                        "create temporary table transactions (\n" +  
                                "    id string,\n" +  
                                "    currency_code string,\n" +  
                                "    total decimal(10,2),\n" +  
                                "    proc_time as proctime() \n" +  
                                ") with (\n" +  
                                "    'connector' = 'datagen',\n" +  
                                "    'rows-per-second' = '100'\n" +  
                                ")")  
                .print();  
  
        // 维表: currency_rates 
        // 可以没有主键 
        tableEnv  
                .executeSql(  
                        "create temporary table currency_rates (\n" +  
                                "    currency_code string,\n" +  
                                "    eur_rate decimal(6,4),\n" +  
                                "    proc_time as proctime(),\n" +  
                                "    primary key (currency_code) not enforced \n"  
                                +  
                                ")\n" +  
                                "with (\n" +  
                                "    'connector' = 'datagen',\n" +  
                                "    'rows-per-second' = '100'\n" +  
                                ");")  
                .print();  
  
        // 注册 Temporal Table Function        
        TemporalTableFunction rates = tableEnv  
                .from("currency_rates")  
                .createTemporalTableFunction($("proc_time"), $("currency_code"));  
  
        tableEnv.createTemporarySystemFunction("rates", rates);  
  
        String query = "select\n" +  
                "    t.id,\n" +  
                "    t.total * c.eur_rate as total_eur,\n" +  
                "    t.total,\n" +  
                "    c.currency_code,\n" +  
                "    t.proc_time\n" +  
                "from\n" +  
                "    transactions t,\n" +  
                "    lateral table (rates(t.proc_time)) as c\n" +  
                "where\n" +  
                "    t.currency_code = c.currency_code";  
        System.out.println(tableEnv.explainSql(query));  
  
        // 使用 Temporal Table Function + LATERAL TABLE 关键字实现 Temporal Join        
        // 在时态维度上关联时指定的是事实表的处理时间（proc_time）  
		tableEnv.executeSql(query).print();
    }  
}
```

- plan
```Text
== Abstract Syntax Tree ==
LogicalProject(id=[$0], total_eur=[*($2, $5)], total=[$2], currency_code=[$4], proc_time=[$3])
+- LogicalFilter(condition=[=($1, $4)])
   +- LogicalCorrelate(correlation=[$cor0], joinType=[inner], requiredColumns=[{3}])
      :- LogicalProject(id=[$0], currency_code=[$1], total=[$2], proc_time=[PROCTIME()])
      :  +- LogicalTableScan(table=[[default_catalog, default_database, transactions]])
      +- LogicalTableFunctionScan(invocation=[rates($cor0.proc_time)], rowType=[RecordType:peek_no_expand(VARCHAR(2147483647) currency_code, DECIMAL(6, 4) eur_rate, TIMESTAMP_LTZ(3) *PROCTIME* proc_time)])

== Optimized Physical Plan ==
Calc(select=[id, *(total, eur_rate) AS total_eur, total, currency_code0 AS currency_code, PROCTIME_MATERIALIZE(proc_time) AS proc_time])
+- TemporalJoin(joinType=[InnerJoin], where=[AND(__TEMPORAL_JOIN_CONDITION(proc_time, __TEMPORAL_JOIN_CONDITION_PRIMARY_KEY(currency_code0)), =(currency_code, currency_code0))], select=[id, currency_code, total, proc_time, currency_code0, eur_rate])
   :- Exchange(distribution=[hash[currency_code]])
   :  +- Calc(select=[id, currency_code, total, PROCTIME() AS proc_time])
   :     +- TableSourceScan(table=[[default_catalog, default_database, transactions]], fields=[id, currency_code, total])
   +- Exchange(distribution=[hash[currency_code]])
      +- TableSourceScan(table=[[default_catalog, default_database, currency_rates]], fields=[currency_code, eur_rate])

== Optimized Execution Plan ==
Calc(select=[id, (total * eur_rate) AS total_eur, total, currency_code0 AS currency_code, PROCTIME_MATERIALIZE(proc_time) AS proc_time])
+- TemporalJoin(joinType=[InnerJoin], where=[(__TEMPORAL_JOIN_CONDITION(proc_time, __TEMPORAL_JOIN_CONDITION_PRIMARY_KEY(currency_code0)) AND (currency_code = currency_code0))], select=[id, currency_code, total, proc_time, currency_code0, eur_rate])
   :- Exchange(distribution=[hash[currency_code]])
   :  +- Calc(select=[id, currency_code, total, PROCTIME() AS proc_time])
   :     +- TableSourceScan(table=[[default_catalog, default_database, transactions]], fields=[id, currency_code, total])
   +- Exchange(distribution=[hash[currency_code]])
      +- TableSourceScan(table=[[default_catalog, default_database, currency_rates]], fields=[currency_code, eur_rate])
```