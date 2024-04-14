## Logic & Physical Plan & Execution
### Physical Logical Optimized
- Proctime
```mermaid
flowchart LR
StreamPhysicalXXXScan-->|MiniBatchIntervalInferRule|StreamPhysicalMiniBatchAssigner
```
- Rowtime
```mermaid
flowchart LR
StreamPhysicalWatermarkAssigner-->|MiniBatchIntervalInferRule|StreamPhysicalMiniBatchAssigner
```
### Transform to Exec Plan
```mermaid
flowchart LR
StreamPhysicalMiniBatchAssigner-->StreamExecMiniBatchAssigner
```
### Stream Operator
- Proctime
```mermaid
flowchart LR
StreamExecMiniBatchAssigner-->|MiniBatchMode.ProcTime|ProcTimeMiniBatchAssignerOperator
```
- Rowtime
```mermaid
flowchart LR
StreamExecMiniBatchAssigner-->|MiniBatchMode.RowTime|RowTimeMiniBatchAssginerOperator
```
## Limitation
1. [[Group Aggregate]]
2. [[Regular Deduplicate]]
3. [[Window TVF Aggregate]]
## MiniBatchInterval
```Java
/** interval of mini-batch. */  
// batch时间大小
@JsonProperty(FIELD_NAME_INTERVAL)  
private final long interval;  
/** The type of mini-batch: rowtime/proctime. */  
@JsonProperty(FIELD_NAME_MODE)  
private final MiniBatchMode mode;
```
## MiniBatchIntervalTrait
```Text
`MiniBatchIntervalTrait`是Calcite中`RelTrait`的一种实现, 用于标识`StreamPhysicalRel`节点的Mini-Batch属性. 其内部持有一个`MiniBatchInterval`实例, 可通过`getMiniBatchInterval()`方法获取.
```
## Config
```Java
// enable mini-batch optimization
configuration.set("table.exec.mini-batch.enabled", "true"); 

// use 5 seconds to buffer input records
configuration.set("table.exec.mini-batch.allow-latency", "5s"); 

// the maximum number of records can be buffered by each aggregate operator task
configuration.set("table.exec.mini-batch.size", "5000"); 
```
## MiniBatchIntervalInferRule
### onMatch()
```Scala
override def onMatch(call: RelOptRuleCall): Unit = {  
  val rel: StreamPhysicalRel = call.rel(0)  
  val miniBatchIntervalTrait = rel.getTraitSet.getTrait(MiniBatchIntervalTraitDef.INSTANCE)  
  val inputs = getInputs(rel)  
  val tableConfig = unwrapTableConfig(rel)  
  val miniBatchEnabled = tableConfig.get(ExecutionConfigOptions.TABLE_EXEC_MINIBATCH_ENABLED)  
  
  val updatedTrait = rel match {  
    case _: StreamPhysicalGroupWindowAggregate =>  
      // TODO introduce mini-batch window aggregate later  
      MiniBatchIntervalTrait.NO_MINIBATCH  
  
    case _: StreamPhysicalWatermarkAssigner => MiniBatchIntervalTrait.NONE  
  
    case _: StreamPhysicalMiniBatchAssigner => MiniBatchIntervalTrait.NONE  
  
    case _ =>  
    // 合并当前节点的mini-batch interval
      if (rel.requireWatermark && miniBatchEnabled) {  
        val mergedInterval = FlinkRelOptUtil.mergeMiniBatchInterval(  
          miniBatchIntervalTrait.getMiniBatchInterval,  
          new MiniBatchInterval(0, MiniBatchMode.RowTime))  
        new MiniBatchIntervalTrait(mergedInterval)  
      } else {  
        miniBatchIntervalTrait  
      }  
  }  
  
  // propagate parent's MiniBatchInterval to children.
  // 传递mini-batch trait到子节点  
  val updatedInputs = inputs.map {  
    input =>  
      // add mini-batch watermark assigner node.  
      if (shouldAppendMiniBatchAssignerNode(input)) {  
        new StreamPhysicalMiniBatchAssigner(  
          input.getCluster,  
          input.getTraitSet,  
          // attach NONE trait for all of the inputs of MiniBatchAssigner,  
          // as they are leaf nodes and don't need to do propagate
          // 只有匹配到source scan和watermark assigner节点才会走到这个方法，
          // 因为是叶子节点，所以不需要添加mini batch trait物理属性          
          input.copy(input.getTraitSet.plus(MiniBatchIntervalTrait.NONE), input.getInputs)  
        )  
      } else {  
      // 否则合并parent和child的mini batch属性
        val originTrait = input.getTraitSet.getTrait(MiniBatchIntervalTraitDef.INSTANCE)  
        if (originTrait != updatedTrait) {  
          // calculate new MiniBatchIntervalTrait according parent's miniBatchInterval  
          // and the child's original miniBatchInterval.          
          val mergedMiniBatchInterval = FlinkRelOptUtil.mergeMiniBatchInterval(  
            originTrait.getMiniBatchInterval,  
            updatedTrait.getMiniBatchInterval)  
          val inferredTrait = new MiniBatchIntervalTrait(mergedMiniBatchInterval)  
          input.copy(input.getTraitSet.plus(inferredTrait), input.getInputs)  
        } else {  
          input        }      }  
  }  
  // update parent if a child was updated  
  if (inputs != updatedInputs) {  
    val newRel = rel.copy(rel.getTraitSet, updatedInputs)  
    call.transformTo(newRel)  
  }  
}
```
### shouldAppendMiniBatchAssignerNode()
```Scala
private def shouldAppendMiniBatchAssignerNode(node: RelNode): Boolean = {  
  val mode = node.getTraitSet  
    .getTrait(MiniBatchIntervalTraitDef.INSTANCE)  
    .getMiniBatchInterval  
    .getMode  
  node match {  
    case _: StreamPhysicalDataStreamScan | _: StreamPhysicalLegacyTableSourceScan |  
        _: StreamPhysicalTableSourceScan =>  
      // append minibatch node if the mode is not NONE and reach a source leaf node  
      mode == MiniBatchMode.RowTime || mode == MiniBatchMode.ProcTime  
    case _: StreamPhysicalWatermarkAssigner =>  
      // append minibatch node if it is rowtime mode and the child is watermark assigner  
      // TODO: if it is ProcTime mode, we also append a minibatch node for now.  
      //  Because the downstream can be a regular aggregate and the watermark assigner  
      //  might be redundant. In FLINK-14621, we will remove redundant watermark assigner,  
      //  then we can remove the ProcTime condition.  
      mode == MiniBatchMode.RowTime || mode == MiniBatchMode.ProcTime  
    case _ =>  
      // others do not append minibatch node  
      false  
  }  
}
```
## FlinkRelOptUtil
### mergeMiniBatchInterval()
```Scala
/**  
 * Merge two MiniBatchInterval as a new one. 
 * The Merge Logic: MiniBatchMode: (R: rowtime, P: proctime, N: None), I: Interval 
 * Possible values:  
 *   - (R, I = 0): operators that require watermark (window excluded).  
 *   - (R, I > 0): window / operators that require watermark with minibatch enabled.  
 *   - (R, I = -1): existing window aggregate  
 *   - (P, I > 0): unbounded agg with minibatch enabled.  
 *   - (N, I = 0): no operator requires watermark, minibatch disabled
 * 
 * 主要逻辑：
 * | A | B | merged result
 * | R, I_a == 0 | R, I_b | R, gcd(I_a, I_b) 
 * | R, I_a == 0 | P, I_b | R, I_b 
 * | R, I_a > 0 | R, I_b | R, gcd(I_a, I_b) 
 * | R, I_a > 0 | P, I_b | R, I_a 
 * | R, I_a = -1 | R, I_b | R, I_a 
 * | R, I_a = -1 | P, I_b | R, I_a 
 * | P, I_a | R, I_b == 0 | R, I_a 
 * | P, I_a | R, I_b > 0 | R, I_b 
 * | P, I_a | P, I_b > 0 | P, I_a 
 */
 def mergeMiniBatchInterval(  
    interval1: MiniBatchInterval,  
    interval2: MiniBatchInterval): MiniBatchInterval = {  
  if (  
    interval1 == MiniBatchInterval.NO_MINIBATCH ||  
    interval2 == MiniBatchInterval.NO_MINIBATCH  
  ) {  
    return MiniBatchInterval.NO_MINIBATCH  
  }  
  interval1.getMode match {  
    case MiniBatchMode.None => interval2  
    case MiniBatchMode.RowTime =>  
      interval2.getMode match {  
        case MiniBatchMode.None => interval1  
        case MiniBatchMode.RowTime =>  
          val gcd = ArithmeticUtils.gcd(interval1.getInterval, interval2.getInterval)  
          new MiniBatchInterval(gcd, MiniBatchMode.RowTime)  
        case MiniBatchMode.ProcTime =>  
          if (interval1.getInterval == 0) {  
            new MiniBatchInterval(interval2.getInterval, MiniBatchMode.RowTime)  
          } else {  
            interval1          }      }  
    case MiniBatchMode.ProcTime =>  
      interval2.getMode match {  
        case MiniBatchMode.None | MiniBatchMode.ProcTime => interval1  
        case MiniBatchMode.RowTime =>  
          if (interval2.getInterval > 0) {  
            interval2          } else {  
            new MiniBatchInterval(interval1.getInterval, MiniBatchMode.RowTime)  
          }  
      }  
  }  
}
```
## ProcTimeMiniBatchAssignerOperator
```Java
public void open() throws Exception {  
    super.open();  
  
    currentWatermark = 0;  
  
    long now = getProcessingTimeService().getCurrentProcessingTime();  
    getProcessingTimeService().registerTimer(now + intervalMs, this);  
  
    // report marker metric  
    getRuntimeContext()  
            .getMetricGroup()  
            .gauge("currentBatch", (Gauge<Long>) () -> currentWatermark);  
}  

/**
 * Processing Time场景下的实现在`ProcTimeMiniBatchAssignerOperator`中, 主要代码如下. 
 * 在`processElement()`会判断当前时间是否可以出发一个Batch, 如果可以则会下发一个Watermark. 
 * 如果没有持续的数据输入, `onProcessingTime()`会保证每隔一个Mini-Batch窗口下发一个Watermark.
 */
public void processElement(StreamRecord<RowData> element) throws Exception {  
    long now = getProcessingTimeService().getCurrentProcessingTime();  
    long currentBatch = now - now % intervalMs;  
    if (currentBatch > currentWatermark) {  
        currentWatermark = currentBatch;  
        // emit  
        output.emitWatermark(new Watermark(currentBatch));  
    }    
    output.collect(element);  
}  

public void onProcessingTime(long timestamp) throws Exception {  
    long now = getProcessingTimeService().getCurrentProcessingTime();  
    long currentBatch = now - now % intervalMs;  
    if (currentBatch > currentWatermark) {  
        currentWatermark = currentBatch;  
        // emit  
        output.emitWatermark(new Watermark(currentBatch));  
    }    
    getProcessingTimeService().registerTimer(currentBatch + intervalMs, this);  
}  
  
public void processWatermark(Watermark mark) throws Exception {  
    // if we receive a Long.MAX_VALUE watermark we forward it since it is used  
    // to signal the end of input and to not block watermark progress downstream    
    if (mark.getTimestamp() == Long.MAX_VALUE && currentWatermark != Long.MAX_VALUE) {  
        currentWatermark = Long.MAX_VALUE;  
        output.emitWatermark(mark);  
    }}
```
## RowTimeMiniBatchAssginerOperator
```Java
public void open() throws Exception {  
    super.open();  
  
    currentWatermark = 0;  
    nextWatermark =  
	    getMiniBatchStart(currentWatermark, minibatchInterval) + minibatchInterval - 1;  
}  
  
public void processElement(StreamRecord<RowData> element) throws Exception {  
    // forward records  
    output.collect(element);  
}  
  
public void processWatermark(Watermark mark) throws Exception {  
    // if we receive a Long.MAX_VALUE watermark we forward it since it is used  
    // to signal the end of input and to not block watermark progress downstream    
    if (mark.getTimestamp() == Long.MAX_VALUE && currentWatermark != Long.MAX_VALUE) {  
        currentWatermark = Long.MAX_VALUE;  
        output.emitWatermark(mark);  
        return;  
    }  
    currentWatermark = Math.max(currentWatermark, mark.getTimestamp());  
    if (currentWatermark >= nextWatermark) {  
        advanceWatermark();  
    }}  
  
private void advanceWatermark() {  
    output.emitWatermark(new Watermark(currentWatermark));  
    long start = getMiniBatchStart(currentWatermark, minibatchInterval);  
    long end = start + minibatchInterval - 1;  
    nextWatermark = end > currentWatermark ? end : end + minibatchInterval;  
}  

// 获取当前watermark所属batch的start timestamp
/** Method to get the mini-batch start for a watermark. */  
private static long getMiniBatchStart(long watermark, long interval) {  
    return watermark - (watermark + interval) % interval;  
}
```
## Resources
1. [Performance Tuning | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/table/tuning/#minibatch-aggregation)
2. [Configuration | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/table/config/#execution-options)
3. [Flink SQL源码 - Mini-Batch原理与实现 - Liebing's Blog](https://liebing.org.cn/flink-sql-minibatch.html)
4. [Flink源码分析 - Flink SQL中的MiniBatch Aggregation优化 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/463194259)
5. [FLink聚合性能优化--MiniBatch分析_flink mini-batch-CSDN博客](https://blog.csdn.net/LS_ice/article/details/100132560)