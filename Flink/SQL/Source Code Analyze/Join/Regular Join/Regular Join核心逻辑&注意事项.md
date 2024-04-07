## StreamPhysicalJoinRule
- 匹配条件和限制
```Java
override def matches(call: RelOptRuleCall): Boolean = {  
  val join: FlinkLogicalJoin = call.rel(0)  
  val left: FlinkLogicalRel = call.rel(1).asInstanceOf[FlinkLogicalRel]  
  val right: FlinkLogicalRel = call.rel(2).asInstanceOf[FlinkLogicalRel]  
  // 是否满足regular join
  if (!satisfyRegularJoin(join, left, right)) {  
    return false  
  }  
  
  // validate the join
  // Temporal table join只能是右边 
  if (left.isInstanceOf[FlinkLogicalSnapshot]) {  
    throw new TableException(  
      "Temporal table join only support apply FOR SYSTEM_TIME AS OF on the right table.")  }  
  
  // INITIAL_TEMPORAL_JOIN_CONDITION should not appear in physical phase in case which fallback  
  // to regular join  
  checkState(!containsInitialTemporalJoinCondition(join.getCondition))  
  
  // Time attributes must not be in the output type of a regular join
  // regular join输出不能包含`事件时间`属性,处理时间可以 
  val timeAttrInOutput = join.getRowType.getFieldList  
    .exists(f => FlinkTypeFactory.isTimeIndicatorType(f.getType))  
  checkState(!timeAttrInOutput)  
  
  // Join condition must not access time attributes
  // join 条件中不能包含`事件时间`属性，处理时间可以  
  val remainingPredsAccessTime =  
    accessesTimeAttribute(join.getCondition, combineJoinInputsRowType(join))  
  checkState(!remainingPredsAccessTime)  
  true  
}
```
## Required &  Provided Change Log Mode
### FlinkChangelogModeInferenceProgram 
`SatisfyModifyKindSetTraitVisitor.visit()`
```Java
case join: StreamPhysicalJoin =>  
  // join support all changes in input 
  // required change log为all,也就是没要求 
  val children = visitChildren(rel, ModifyKindSetTrait.ALL_CHANGES)  
  val leftKindSet = getModifyKindSet(children.head)  
  val rightKindSet = getModifyKindSet(children.last)  
  val innerOrSemi = join.joinSpec.getJoinType == FlinkJoinType.INNER ||  
    join.joinSpec.getJoinType == FlinkJoinType.SEMI
  // 产生的change log类型取决于:
  // 1. join的类型，如果是inner join或者semi join，就是两个子节点输出的change log类型之和；如果是其他类型的join，则是all
  // 2. join两边输出的change log类型，也就是两个子节点
  val providedTrait = if (innerOrSemi) {  
    // forward left and right modify operations  
    new ModifyKindSetTrait(leftKindSet.union(rightKindSet))  
  } else {  
    // otherwise, it may produce any kinds of changes  
    ModifyKindSetTrait.ALL_CHANGES  
  }  
  createNewNode(join, children, providedTrait, requiredTrait, requester)
```

`SatisfyUpdateKindTraitVisitor.visit()`
```Java
case join: StreamPhysicalJoin =>
  // 如果父节点只要求update_before
  val onlyAfterByParent = requiredTrait.updateKind == UpdateKind.ONLY_UPDATE_AFTER  
  val children = join.getInputs.zipWithIndex.map {
	// 遍历两个子节点  
    case (child, childOrdinal) =>  
      val physicalChild = child.asInstanceOf[StreamPhysicalRel]
      // 子节点输入数据的unique key是否包含join key，如果包含，那么子节点可以不输出update_bofore  
      val supportOnlyAfter = join.inputUniqueKeyContainsJoinKey(childOrdinal)  
      val inputModifyKindSet = getModifyKindSet(physicalChild)  
      if (onlyAfterByParent) {  
        if (inputModifyKindSet.contains(ModifyKind.UPDATE) && !supportOnlyAfter) {  
          // the parent requires only-after, however, the join doesn't support this  
          None  
        } else {  
          this.visit(physicalChild, onlyAfterOrNone(inputModifyKindSet))  
        }  
      } else {  
        this.visit(physicalChild, beforeAfterOrNone(inputModifyKindSet))  
      }  
  }  
  if (children.exists(_.isEmpty)) {  
    None  
  } else {  
    createNewNode(join, Some(children.flatten.toList), requiredTrait)  
  }
```

### StreamPhysicalJoin
`inputUniqueKeyContainsJoinKey()`
```Java
/**  
 * This is mainly used in `FlinkChangelogModeInferenceProgram.SatisfyUpdateKindTraitVisitor`. If  
 * the unique key of input contains join key, then it can support ignoring UPDATE_BEFORE. 
 * Otherwise, it can't ignore UPDATE_BEFORE. For example, if the input schema is [id, name, cnt] 
 * with the unique key (id). The join key is (id, name), then an insert and update on the id: 
 *
 * +I(1001, Tim, 10) * -U(1001, Tim, 10) +U(1001, Timo, 11) 
 * 
 * If the UPDATE_BEFORE is ignored, the `+I(1001, Tim, 10)` record in join will never be 
 * retracted. Therefore, if we want to ignore UPDATE_BEFORE, the unique key must contain join key. 
 * 
 * @see FlinkChangelogModeInferenceProgram  
 */
 // 结合join state存储结构，分析一下为啥忽略update_bofore,结果也是正确的(针对父节点只要求update_after)
 /*
  * case1: 如果input unique key -> id, join key -> id + name, 那么join state会使用JoinKeyContainsUniqueKey，state结构是<JK, Value<Record>>, 也就是<id_name, record>，那么此时如果不发送回撤消息，那么上游update_after消息就会发到join的另一个subtask，所以老数据在join端永远不会删除
  * case2: 如果input unique key -> id + name, join key -> id, 那么join state会使用InputSideHasUniqueKey，state结构是<JK, Map<UK, record>>, 也就是<id, Map<id+name, record>>，那么此时发不发送回撤消息，都不影响下游最终结果,反而减少了下游数据量
  */
 def inputUniqueKeyContainsJoinKey(inputOrdinal: Int): Boolean = {  
  val input = getInput(inputOrdinal)  
  val joinKeys = if (inputOrdinal == 0) joinSpec.getLeftKeys else joinSpec.getRightKeys  
  val inputUniqueKeys = getUpsertKeys(input, joinKeys)  
  if (inputUniqueKeys != null) {  
    inputUniqueKeys.exists(uniqueKey => joinKeys.forall(uniqueKey.contains(_)))  
  } else {  
    false  
  }  
}
```
## 核心逻辑
### StreamExecJoin
`translateToPlanInternal() - 获取左边和右边join key的特征`
```Java
final InternalTypeInfo<RowData> rightTypeInfo = InternalTypeInfo.of(rightType);
final JoinInputSideSpec leftInputSpec =  
        JoinUtil.analyzeJoinInput(  
                planner.getFlinkContext().getClassLoader(),  
                leftTypeInfo,  
                leftJoinKey,  
                leftUpsertKeys);  
  
final InternalTypeInfo<RowData> rightTypeInfo = InternalTypeInfo.of(rightType);  
final JoinInputSideSpec rightInputSpec =  
        JoinUtil.analyzeJoinInput(  
                planner.getFlinkContext().getClassLoader(),  
                rightTypeInfo,  
                rightJoinKey,  
                rightUpsertKeys);

```

### StreamingJoinOperator - Inner & Outer Join
`open() - 根据join key的特征，决定左右表state的结构` - [[Join Related Core Basic Class]]
```Java
// initialize states  
if (leftIsOuter) {  
    this.leftRecordStateView =  
            OuterJoinRecordStateViews.create(  
                    getRuntimeContext(),  
                    "left-records",  
                    leftInputSideSpec,  
                    leftType,  
                    stateRetentionTime);  
} else {  
    this.leftRecordStateView =  
            JoinRecordStateViews.create(  
                    getRuntimeContext(),  
                    "left-records",  
                    leftInputSideSpec,  
                    leftType,  
                    stateRetentionTime);  
}  
  
if (rightIsOuter) {  
    this.rightRecordStateView =  
            OuterJoinRecordStateViews.create(  
                    getRuntimeContext(),  
                    "right-records",  
                    rightInputSideSpec,  
                    rightType,  
                    stateRetentionTime);  
} else {  
    this.rightRecordStateView =  
            JoinRecordStateViews.create(  
                    getRuntimeContext(),  
                    "right-records",  
                    rightInputSideSpec,  
                    rightType,  
                    stateRetentionTime);  
}
```

`processElement()`
```Java
// 处理左表数据
@Override  
public void processElement1(StreamRecord<RowData> element) throws Exception {  
    processElement(element.getValue(), leftRecordStateView, rightRecordStateView, true);  
}  

// 处理右表数据
@Override  
public void processElement2(StreamRecord<RowData> element) throws Exception {  
    processElement(element.getValue(), rightRecordStateView, leftRecordStateView, false);  
}

// 公共逻辑
private void processElement(  
        RowData input,  
        JoinRecordStateView inputSideStateView,  
        JoinRecordStateView otherSideStateView,  
        boolean inputIsLeft)  
        throws Exception {
    // 判断是不是outer join  
    boolean inputIsOuter = inputIsLeft ? leftIsOuter : rightIsOuter;
    // 处理左表的时候这里就是右边，处理右表的时候这里就是左表  
    boolean otherIsOuter = inputIsLeft ? rightIsOuter : leftIsOuter;  
    boolean isAccumulateMsg = RowDataUtil.isAccumulateMsg(input);  
    RowKind inputRowKind = input.getRowKind();  
    input.setRowKind(RowKind.INSERT); // erase RowKind for later state updating  

	// look up 另一侧表的数据，看有没有match上的数据
    AssociatedRecords associatedRecords =  
            AssociatedRecords.of(input, inputIsLeft, otherSideStateView, joinCondition);  
    // insert & update_after数据流
    if (isAccumulateMsg) { 
	    // record is accumulate
        // 处理outer join  
        if (inputIsOuter) { 
	        // input side is outer  
            OuterJoinRecordStateView inputSideOuterStateView =  
                    (OuterJoinRecordStateView) inputSideStateView;  
            if (associatedRecords.isEmpty()) { 
            // there is no matched rows on the other side  
	        // send +I[record+null]                
		        outRow.setRowKind(RowKind.INSERT);
		        // 没有join上的话，另一侧所有字段为null  
                outputNullPadding(input, inputIsLeft);  
                // state.add(record, 0)  
                inputSideOuterStateView.addRecord(input, 0);  
            } else { 
	            // there are matched rows on the other side  
                if (otherIsOuter) { 
	                // other side is outer
	                // 如果另一侧也是outer join，意味着这是full outer join  
                    OuterJoinRecordStateView otherSideOuterStateView =  
                            (OuterJoinRecordStateView) otherSideStateView;  
                    for (OuterRecord outerRecord : associatedRecords.getOuterRecords()) {  
                        RowData other = outerRecord.record;  
                        // if the matched num in the matched rows == 0
                        // 如果之前另一侧表没有join上数据，那么需要将之前的null数据撤回  
                        if (outerRecord.numOfAssociations == 0) {  
                            // send -D[null+other]  
                            outRow.setRowKind(RowKind.DELETE);  
                            outputNullPadding(other, !inputIsLeft);  
                        } 
                        // ignore matched number > 0  
                        // otherState.update(other, old + 1)
                        // 同时需要更新另一侧state，将associate num加上join上的数据个数        
                        otherSideOuterStateView.updateNumOfAssociations(  
                                other, outerRecord.numOfAssociations + 1);  
                    }  
                }  
                // send +I[record+other]s  
                outRow.setRowKind(RowKind.INSERT); 
                // 将所有关联的数据发出去 
                for (RowData other : associatedRecords.getRecords()) {  
                    output(input, other, inputIsLeft);  
                }  
                // state.add(record, other.size)
                // 更新state,需要记录关联数据的个数，方便后面retract,如果关联个数为0，说明上一次
                // 没有关联上，下次关联上的时候，需要发出撤回数据  
                inputSideOuterStateView.addRecord(input, associatedRecords.size());  
            }  
        } else { 
	        // input side not outer  
            // state.add(record)
            // inner join            
            inputSideStateView.addRecord(input);  
            if (!associatedRecords.isEmpty()) { 
            // if there are matched rows on the other side  
                if (otherIsOuter) { 
	                // if other side is outer  
                    OuterJoinRecordStateView otherSideOuterStateView =  
                            (OuterJoinRecordStateView) otherSideStateView;  
                    for (OuterRecord outerRecord : associatedRecords.getOuterRecords()) {  
                        if (outerRecord.numOfAssociations  
                                == 0) { 
            // if the matched num in the matched rows == 0, send -D[null+other]          
                            outRow.setRowKind(RowKind.DELETE);  
                            outputNullPadding(outerRecord.record, !inputIsLeft);  
                        }                        
                        // otherState.update(other, old + 1)  
                        otherSideOuterStateView.updateNumOfAssociations(  
                                outerRecord.record, outerRecord.numOfAssociations + 1);  
                    }  
                    // send +I[record+other]s  
                    outRow.setRowKind(RowKind.INSERT);  
                } else {  
                    // send +I/+U[record+other]s (using input RowKind)  
                    outRow.setRowKind(inputRowKind);  
                }  
                for (RowData other : associatedRecords.getRecords()) {
			        // 发送数据  
                    output(input, other, inputIsLeft);  
                }  
            }  
            // 没有任何match上的数据，inner join什么都不做
            // skip when there is no matched rows on the other side  
        }  
    } else {
	    // update_bofore & delete数据 
	    // input record is retract  
        // state.retract(record)        
        inputSideStateView.retractRecord(input);  
        if (associatedRecords.isEmpty()) { 
        // there is no matched rows on the other side  
            if (inputIsOuter) { 
            // input side is outer, send -D[record+null]
            outRow.setRowKind(RowKind.DELETE);  
                outputNullPadding(input, inputIsLeft);  
            }  
            // nothing to do when input side is not outer  
        } else { 
	        // there are matched rows on the other side  
            if (inputIsOuter) {  
                // send -D[record+other]s  
                outRow.setRowKind(RowKind.DELETE);  
            } else {  
                // send -D/-U[record+other]s (using input RowKind)  
                outRow.setRowKind(inputRowKind);  
            }  
            for (RowData other : associatedRecords.getRecords()) {  
                output(input, other, inputIsLeft);  
            }  
            // if other side is outer  
            if (otherIsOuter) {
	            // 如果另一侧也是outer join，说明是full outer join  
                OuterJoinRecordStateView otherSideOuterStateView =  
                        (OuterJoinRecordStateView) otherSideStateView;  
                for (OuterRecord outerRecord : associatedRecords.getOuterRecords()) {  
                    if (outerRecord.numOfAssociations == 1) {  
                        // send +I[null+other]
                        // 如果曾经只关联上一次，那么发送一条null数据  
                        outRow.setRowKind(RowKind.INSERT);  
                        outputNullPadding(outerRecord.record, !inputIsLeft);  
                    } 
                    // nothing else to do when number of associations > 1  
                    // otherState.update(other, old - 1)
                    // 更新一下另一侧state，将associate num减一
                    otherSideOuterStateView.updateNumOfAssociations(  
                            outerRecord.record, outerRecord.numOfAssociations - 1);  
                }  
            }  
        }  
    }  
}
```

### StreamingSemiAntiJoinOperator - Semi & Anti Join
`open() - 根据join key的特征，决定左右表state的结构` - [[Join Related Core Basic Class]]
```Java
// initialize states  
this.leftRecordStateView =  
        OuterJoinRecordStateViews.create(  
                getRuntimeContext(),  
                LEFT_RECORDS_STATE_NAME,  
                leftInputSideSpec,  
                leftType,  
                stateRetentionTime);  
  
this.rightRecordStateView =  
        JoinRecordStateViews.create(  
                getRuntimeContext(),  
                RIGHT_RECORDS_STATE_NAME,  
                rightInputSideSpec,  
                rightType,  
                stateRetentionTime);
```

`processElement()`
```Java
// 处理左表数据
public void processElement1(StreamRecord<RowData> element) throws Exception {  
    RowData input = element.getValue();  
    // 这里会返回所有records,即使record一样，也要返回 --- 重点 
    // 这里没有返回associate num,因为右边state里面没有记录 --- 重点
    AssociatedRecords associatedRecords =  
            AssociatedRecords.of(input, true, rightRecordStateView, joinCondition);  
    if (associatedRecords.isEmpty()) {  
        if (isAntiJoin) {  
            // 没有join上数据，发出去(anti join)  
            collector.collect(input);  
        }  
        // 如果没关联上，所以不需要发送数据(semi join) --- 重点 
    } else { // there are matched rows on the other side  
        if (!isAntiJoin) {  
            // semi join  
            // 这里是不是可以优化一下，判断一下associatedRecords的大小，
            // 如果大于1，说明曾经join到过，不需要再发出去了  
            // 但是要做这个优化，必须要改一下right state结构，需要增加一下associated record nums  
            collector.collect(input);  
        }  
        // 不是空，那么什么都不做(anti join)  --- 重点
    }  
    // 如果关联上了，不需要发送数据(anti join)  
    if (RowDataUtil.isAccumulateMsg(input)) {  
        // erase RowKind for state updating  
        input.setRowKind(RowKind.INSERT);  
        leftRecordStateView.addRecord(input, associatedRecords.size());  
    } else { // input is retract  
        // erase RowKind for state updating        
        input.setRowKind(RowKind.INSERT);  
        // 将record出现的次数减一，如果出现的次数小于等于1了，那么就从state中remove，
        // 注意右表state不变化(semi join & anti join)  
        leftRecordStateView.retractRecord(input);  
    }}

// 处理右表数据
public void processElement2(StreamRecord<RowData> element) throws Exception {  
    RowData input = element.getValue();  
    boolean isAccumulateMsg = RowDataUtil.isAccumulateMsg(input);  
    RowKind inputRowKind = input.getRowKind();  
    input.setRowKind(RowKind.INSERT); // erase RowKind for later state updating  
  
    // 这里会返回所有records,即使record一样，也要返回(semi & anti)  --- 重点
    AssociatedRecords associatedRecords =  
            AssociatedRecords.of(input, false, leftRecordStateView, joinCondition);  
    if (isAccumulateMsg) { // record is accumulate  
        rightRecordStateView.addRecord(input);  
        if (!associatedRecords.isEmpty()) {  
            // there are matched rows on the other side  
            for (OuterRecord outerRecord : associatedRecords.getOuterRecords()) {  
                RowData other = outerRecord.record;  
                if (outerRecord.numOfAssociations == 0) {  
                    if (isAntiJoin) {  
                        // send -D[other]  
                        // 如果之前associate是0，说明之前没有join上数据，所以发出去了，
                        // 但是现在join上了，所以要撤回数据(anti join)  --- 重点
                        other.setRowKind(RowKind.DELETE);  
                    } else {  
                        // send +I/+U[other] (using input RowKind)  
                        // 如果以前没join上，现在join上了，需要发送数据(semi join) --- 重点 
                        other.setRowKind(inputRowKind);  
                    }  
                    collector.collect(other);  
                    // set header back to INSERT, 
                    // because we will update the other row to state  
                    other.setRowKind(RowKind.INSERT);  
                } // ignore when number > 0  
                // 如果以前已经join上了，现在join上了也不需要发送数据了，
                // 但是associate num要加一 (semi join & anti join)                
                // 这种针对右表连续来了相同的记录,左表就不需要发送数据了，
                // 减少发送的数据量(semi join)  
                // 如果以前join上了，现在也join上了，说明以前已经撤回过数据了，
                // 现在只需要将associate num要加一(anti join)  --- 重点
                leftRecordStateView.updateNumOfAssociations(  
                        other, outerRecord.numOfAssociations + 1);  
            }  
        }  
        // ignore when associated number == 0  --- 重点，有点绕
        // 如果右表没有数据，对于semi join或者anti join都不需要处理  
		// 如果有数据只是没有关联上，那么对于semi join不需要做什么  
		// 对于anti join,在处理左表的时候，已经将数据发出了，这里也不需要做处理 
    } else { // retract input  
        // 如果只有一条数据了，从state中remove  
        rightRecordStateView.retractRecord(input);  
        if (!associatedRecords.isEmpty()) {  
            // there are matched rows on the other side  
            for (OuterRecord outerRecord : associatedRecords.getOuterRecords()) {  
                RowData other = outerRecord.record;  
                // 如果每条数据之前只关联上了一条数据(semi join & anti join)  
                if (outerRecord.numOfAssociations == 1) {  
                    if (!isAntiJoin) {  
                        // 发送回撤数据 (semi join)                        
                        // send -D/-U[other] (using input RowKind)                       
                        other.setRowKind(inputRowKind);  
                    } else {  
                        // 发送数据，因为之前每条数据只关联上一条(anti join) 
                        // send +I[other]  
                        other.setRowKind(RowKind.INSERT);  
                    }  
                    collector.collect(other);  
                    // set RowKind back, because we will update the other row to state  
                    other.setRowKind(RowKind.INSERT);  
                } // ignore when number > 0  
                // 如果每条数据关联上了不只一条数据，
                // 那么更新左表的associate num(semi join & anti join)  
                leftRecordStateView.updateNumOfAssociations(  
                        other, outerRecord.numOfAssociations - 1);  
            }  
        } // ignore when associated number == 0
        // 如果左边没有数据或者没有关联上，那么对于semi join不需要做任何处理
        // 如果左边没有数据，对于anti join也不需要做任何处理
        // 如果左边有数据但没有关联上，那么这条数据在处理左表的时候，已经处理了，现在不需要做任何处理
        // 只需要将右表state更新就好
    }  
}
```

## Limitation
1. [时间属性问题](https://www.zybuluo.com/22221cjp/note/1675448)
2. https://blog.jrwang.me/2020/2020-01-05-flink-sourcecode-sql-stream-join/