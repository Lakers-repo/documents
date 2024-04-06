## Required &  Provided Change Log Mode

`FlinkChangelogModeInferenceProgram -> SatisfyModifyKindSetTraitVisitor.visit()`
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

`FlinkChangelogModeInferenceProgram -> SatisfyUpdateKindTraitVisitor.visit()`
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

`StreamPhysicalJoin.inputUniqueKeyContainsJoinKey()`
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
##  Logic & Physical Plan


## 核心逻辑


## Resources
1. [[Join Related Core Class]]
2. [[Change Log 原理与实现]]