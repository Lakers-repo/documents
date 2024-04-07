## JoinRecordStateView

### Overview
![[Join Record State View.png]]
### JoinRecordStateViews
#### 类关系图
![[Non Outer Join Record State View.png]]
#### 详细说明
> [!NOTE]
> 1. Join Key很好理解，就是join的关联条件;
> Unique Key有点疑惑，其实就是唯一键，例如`group by key1, key2`，那key1, key2对于下游算子来说unique key.
> 2. 如果一个上游表unique key是key1, key2,但是下游查询的时候，一个没查，那么state结构会变成no unique key. 具体逻辑有待debug.

- 对于上面第二点的补充
```SQL
-- view1 
-- unique key: order_id & user_id
create view view1 as select order_id, user_id, count(*) as cnt from left_source_table group by order_id, user_id
-- view2
-- unique key: order_id
create view view2 as select order_id from right_source_table group by order_id
--query1 
-- left table state: InputSideHasUniqueKey 
-- right table state : JoinKeyContainsUniqueKey
select order_id, user_id, cnt from view1 where order_id in (select order_id from view2)

-- query2
-- left table state: InputSideHasNoUniqueKey 
-- right table state : JoinKeyContainsUniqueKey
select cnt from view1 where order_id in (select order_id from view2)
```


| Category                 | State Structure                       | Update Row时间复杂度 | Query By JK时间复杂度 | Comment              |
| ------------------------ | ------------------------------------- | --------------- | ---------------- | -------------------- |
| JoinKeyContainsUniqueKey | `<JK,ValueState<Record>>`             | O(1)            | O(1)             |                      |
| InputSideHasUniqueKey    | `<JK,MapState<UK,Record>>`            | O(2)            | O(N)             | N = size of MapState |
| InputSideHasNoUniqueKey  | `<JK,MapState<Record, appear-times>>` | O(2)            | O(N)             | N = size of MapState |

上述表格中的内容其实不难理解，根据 Join Key 和 Unique Key 的特性，状态的结构分为三种情况：
- 如果 Join Key 包含了 Unique Key，那么一个 Join Key 只会对应一条记录，因此状态存储选择的是 `ValueState`;
- 如果输入存在 Unique Key，但 Join Key 不包含 Unique Key，一个 Join Key 可能会对应多条记录，但这些记录的 Unique Key 一定不同，因此选择使用 `MapState`，key 为 Unique Key， value 为对应的记录;
- 如果输入不存在 Unique Key，那么状态状态只能是 `ListState` 或者 `MapState`，从 update 和 retract 的效率考虑，选择使用 `MapState`，直接使用记录本身作为 Key，value 为记录出现的次数.

还有一种特殊的情况，即不存在 Join Key（笛卡尔积），这种情况其实是 `InputSideHasNoUniqueKey` 的一种特例，所有记录的 Join Key 都是 `BinaryRowUtil.EMPTY_ROW`.

从最终的性能上来看，`JoinkKeyContainsUniqueKey` > `InputSideHasUniqueKey` > `InputSideHasNoUniqueKey`.

#### InputSideHasNoUniqueKey
```Java
// key为数据本身，value是出现的次数
private final MapState<RowData, Integer> recordState;

public void addRecord(RowData record) throws Exception {  
    Integer cnt = recordState.get(record);
    // 如果这个数据曾经出现过  
    if (cnt != null) {  
        cnt += 1;  
    } else {  
        cnt = 1;  
    }  
    recordState.put(record, cnt);  
}

public void retractRecord(RowData record) throws Exception {  
    Integer cnt = recordState.get(record);
    // 如果这个数据曾经出现过超过2次  
    if (cnt != null) {  
        if (cnt > 1) {  
            recordState.put(record, cnt - 1);  
        } else { 
	        // 曾经只出现过最多一次，直接删除 
            recordState.remove(record);  
        }  
    }  
    // ignore cnt == null, which means state may be expired  
}
```

### OuterJoinRecordStateViews
#### 类关系图
![[Outer Join Record State View.png]]

#### 详细说明
`OuterJoinRecordStateView` 是对 `JoinRecordStateView` 的扩展，除了会将记录本身存储在状态里，还会将该条记录在另一侧关联到的记录数存储下来。之所以要将关联记录数存储在状态中，主要是为了方便 Outer Join 中处理和撤回用 NULL 值填充的结果。在下文介绍关联的具体逻辑时会进一步介绍。除此以外，`OuterJoinRecordStateView` 和 `JoinRecordStateView` 的存储结构是一致的。
#### InputSideHasNoUniqueKey
```Java
// stores record in the mapping <Record, <appear-times, associated-num>> 
// key为数据本身，value第一个元素是出现的次数，第二个是关联上另一侧数据的个数
private final MapState<RowData, Tuple2<Integer, Integer>> recordState;

public void addRecord(RowData record, int numOfAssociations) throws Exception {  
    Tuple2<Integer, Integer> tuple = recordState.get(record);  
    if (tuple != null) {
	    // 出现次数加一，关联个数加上本次关联上的数据个数  
        tuple.f0 = tuple.f0 + 1;  
        tuple.f1 = numOfAssociations;  
    } else {  
        tuple = Tuple2.of(1, numOfAssociations);  
    }  
    recordState.put(record, tuple);  
}

// 更新另一侧的state里的associate num
public void updateNumOfAssociations(RowData record, int numOfAssociations)  
        throws Exception {  
    Tuple2<Integer, Integer> tuple = recordState.get(record);  
    if (tuple != null) {  
        tuple.f1 = numOfAssociations;  
    } else {  
        // compatible for state ttl
        // 如果state被清理了，那么appear times重置一，associate num重置最新关联上的数据个数  
        tuple = Tuple2.of(1, numOfAssociations);  
    }  
    recordState.put(record, tuple);  
}

public void retractRecord(RowData record) throws Exception {  
    Tuple2<Integer, Integer> tuple = recordState.get(record);  
    if (tuple != null) {  
        if (tuple.f0 > 1) {
	        // 出现次数减一，关联上的数据个数不变  
            tuple.f0 = tuple.f0 - 1;  
            recordState.put(record, tuple);  
        } else {
	        // 如果以前最多出现过一次，删除state  
            recordState.remove(record);  
        }  
    }  
}
```
## AssociatedRecords
```Java
protected static final class AssociatedRecords {  
    private final List<OuterRecord> records;  
  
    private AssociatedRecords(List<OuterRecord> records) {  
        checkNotNull(records);  
        this.records = records;  
    }  
    public boolean isEmpty() {  
        return records.isEmpty();  
    }  
    public int size() {  
        return records.size();  
    }  
    /**  
     * Gets the iterable of records. This is usually be called when the {@link  
     * AssociatedRecords} is from inner side.  
     */    
     public Iterable<RowData> getRecords() {  
        return new RecordsIterable(records);  
    }  
    /**  
     * Gets the iterable of {@link OuterRecord} which composites 
     * record and numOfAssociations.  
     * This is usually be called when the {@link AssociatedRecords} is from outer side.  
     */    
     public Iterable<OuterRecord> getOuterRecords() {  
        return records;  
    }  
    /**  
     * Creates an {@link AssociatedRecords} which represents the records 
     * associated to the input row.     
     */    
    public static AssociatedRecords of(  
            RowData input,  
            boolean inputIsLeft,  
            JoinRecordStateView otherSideStateView,  
            JoinCondition condition)  
            throws Exception {  
        List<OuterRecord> associations = new ArrayList<>();  
        if (otherSideStateView instanceof OuterJoinRecordStateView) {
        // 处理outer join  
            OuterJoinRecordStateView outerStateView =  
                    (OuterJoinRecordStateView) otherSideStateView;  
            Iterable<Tuple2<RowData, Integer>> records =  
                    outerStateView.getRecordsAndNumOfAssociations();  
            for (Tuple2<RowData, Integer> record : records) {  
                boolean matched =  
                        inputIsLeft  
                                ? condition.apply(input, record.f0)  
                                : condition.apply(record.f0, input);  
                if (matched) {  
                    associations.add(new OuterRecord(record.f0, record.f1));  
                }  
            }  
        } else {
        // 处理inner join  
            Iterable<RowData> records = otherSideStateView.getRecords();  
            for (RowData record : records) {  
                boolean matched =  
                        inputIsLeft  
                                ? condition.apply(input, record)  
                                : condition.apply(record, input);  
                if (matched) {  
					// use -1 as the default number of associations  
                    associations.add(new OuterRecord(record, -1));  
                }  
            }  
        }  
        return new AssociatedRecords(associations);  
    }}
```
## OuterRecord
```Java
protected static final class OuterRecord {  
    public final RowData record;
    // 关联上的数据个数  
    public final int numOfAssociations;  
  
    private OuterRecord(RowData record, int numOfAssociations) {  
        this.record = record;  
        this.numOfAssociations = numOfAssociations;  
    }}
```
## JoinSpec
## JoinCondition
