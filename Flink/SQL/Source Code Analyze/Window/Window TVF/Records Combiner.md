	## 继承关系
![[Records Combiner.png]]

> [!NOTE]
> - AggCombiner - Default
> - LocalAggCombiner & GlobalAggCombiner - Two Stage Optimized
> - TopNRecordsCombiner - Window TopN
> - RowTimeDeduplicateRecordsCombiner - Window Deduplicate

## 核心逻辑
```Java  
public void combine(WindowKey windowKey, Iterator<RowData> records) throws Exception {  
    // step 0: set current key for states and timers  
    keyContext.setCurrentKey(windowKey.getKey());  
  
    // step 1: get the accumulator for the current key and window  
    Long window = windowKey.getWindow();  
    RowData acc = accState.value(window);  
    if (acc == null) {  
        acc = aggregator.createAccumulators();  
    }  
    // step 2: set accumulator to function  
    aggregator.setAccumulators(window, acc);  
  
    // step 3: do accumulate  
    while (records.hasNext()) {  
        RowData record = records.next();  
        if (isAccumulateMsg(record)) {  
            aggregator.accumulate(record);  
        } else {  
            aggregator.retract(record);  
        }  
    }  
  
    // step 4: update accumulator into state  
    acc = aggregator.getAccumulators();  
    accState.update(window, acc);  
  
    // step 5: register timer for current window  
    if (isEventTime) {  
        long currentWatermark = timerService.currentWatermark();  
        ZoneId shiftTimeZone = timerService.getShiftTimeZone();  
        // the registered window timer should hasn't been triggered  
        if (!isWindowFired(window, currentWatermark, shiftTimeZone)) {  
            timerService.registerEventTimeWindowTimer(window);  
        }  
    }  
    // we don't need register processing-time timer, because we already register them  
    // per-record in AbstractWindowAggProcessor.processElement()}
```