## JoinRecordStateView

### Overview
![[Join Record State View.png]]
### JoinRecordStateViews
#### 类关系图
![[Non Outer Join Record State View.png]]
#### 详细说明
> [!NOTE]
> Join Key很好理解，就是join的关联条件;
> Unique Key有点疑惑，其实就是唯一键，例如`group by key1, key2`，那key1, key2对于下游算子来说unique key.

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

### OuterJoinRecordStateViews
#### 类关系图
![[Outer Join Record State View.png]]

#### 详细说明
`OuterJoinRecordStateView` 是对 `JoinRecordStateView` 的扩展，除了会将记录本身存储在状态里，还会将该条记录在另一侧关联到的记录数存储下来。之所以要将关联记录数存储在状态中，主要是为了方便 Outer Join 中处理和撤回用 NULL 值填充的结果。在下文介绍关联的具体逻辑时会进一步介绍。除此以外，`OuterJoinRecordStateView` 和 `JoinRecordStateView` 的存储结构是一致的。