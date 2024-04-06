## 继承关系
![[Slice Processor.png]]

## 重要字段
```Java
// 缓存
protected transient WindowBuffer windowBuffer;  
  
/** state schema: [key, window_end, accumulator]. */  
protected transient WindowValueState<Long> windowState;

```

## state初始化
```Java
this.windowState =  
        new WindowValueState<>((InternalValueState<RowData, Long, RowData>) state);
```

## 核心逻辑 
> [!用window aggregate举例, Window TopN以及Deduplicate在单独文档]
> - 处理每一条数据
> `AbstractWindowAggProcessor`

```Java
public boolean processElement(RowData key, RowData element) throws Exception {  
    long sliceEnd = sliceAssigner.assignSliceEnd(element, clockService);  
    if (!isEventTime) {  
        // always register processing time for every element when processing time mode  
        windowTimerService.registerProcessingTimeWindowTimer(sliceEnd);  
    }  
    if (isEventTime && isWindowFired(sliceEnd, currentProgress, shiftTimeZone)) {  
        // the assigned slice has been triggered, which means current element is late,  
        // but maybe not need to drop        long lastWindowEnd = sliceAssigner.getLastWindowEnd(sliceEnd);  
        if (isWindowFired(lastWindowEnd, currentProgress, shiftTimeZone)) {  
            // the last window has been triggered, so the element can be dropped now  
            return true;  
        } else {  
            windowBuffer.addElement(key, sliceStateMergeTarget(sliceEnd), element);  
            // we need to register a timer for the next unfired window,  
            // because this may the first time we see elements under the key            long unfiredFirstWindow = sliceEnd;  
            while (isWindowFired(unfiredFirstWindow, currentProgress, shiftTimeZone)) {  
                unfiredFirstWindow += windowInterval;  
            }  
            windowTimerService.registerEventTimeWindowTimer(unfiredFirstWindow);  
            return false;  
        }  
    } else {  
        // the assigned slice hasn't been triggered, accumulate into the assigned slice  
        windowBuffer.addElement(key, sliceEnd, element);  
        return false;  
    }}
```

- 处理watermark
```Java
public void advanceProgress(long progress) throws Exception {  
    if (progress > currentProgress) {  
        currentProgress = progress;  
        if (currentProgress >= nextTriggerProgress) {  
            // in order to buffer as much as possible data, we only need to call  
            // advanceProgress() when currentWatermark may trigger window.            // this is a good optimization when receiving late but un-dropped events, because            // they will register small timers and normal watermark will flush the buffer            windowBuffer.advanceProgress(currentProgress);  
            nextTriggerProgress =  
                    getNextTriggerWatermark(  
                            currentProgress, windowInterval, shiftTimeZone, useDayLightSaving);  
        }  
    }  
}
```