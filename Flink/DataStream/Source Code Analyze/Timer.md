## 应用场景

> [!Keyed Process Function]
> https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/datastream/operators/process_function/

## 继承关系
`SimpleTimerService`
![[Simple Timer Service.png]]

`InternalTimerServiceImpl`
![[Internal Timer Service.png]]

## 核心逻辑
### Event Timer
- registerEventTimeTimer
```Java  
public void registerEventTimeTimer(N namespace, long time) {  
    eventTimeTimersQueue.add(  
            new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace));  
}
```

- advanceWatermark
```Java
public void advanceWatermark(long time) throws Exception {  
    currentWatermark = time;  
  
    InternalTimer<K, N> timer;  
    // 将所有需要触发的定时器取出，并调用对应operator的onEventTime方法
    while ((timer = eventTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {  
        keyContext.setCurrentKey(timer.getKey());  
        eventTimeTimersQueue.poll();  
        triggerTarget.onEventTime(timer);  
    }}
```
### Processing Timer
- registerProcessingTimeTimer
```Java  
public void registerProcessingTimeTimer(N namespace, long time) {  
	// 获取下一个即将触发的timer
    InternalTimer<K, N> oldHead = processingTimeTimersQueue.peek(); 
    // 相同时间戳的timer不会重复创建 
    if (processingTimeTimersQueue.add(  
            new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace))) {  
        long nextTriggerTime = oldHead != null ? oldHead.getTimestamp() : Long.MAX_VALUE;  
        // check if we need to re-schedule our timer to earlier
        // 如果新创建的timer时间戳小于下一个即将触发的timer时间戳，则保留当前新创建的timer  
        if (time < nextTriggerTime) {  
            if (nextTimer != null) {  
                nextTimer.cancel(false);  
            }  
            nextTimer = processingTimeService.registerTimer(time, this::onProcessingTime);  
        }  
    }  
}
```

- onProcessingTime
```Java
private void onProcessingTime(long time) throws Exception {  
    // null out the timer in case the Triggerable calls registerProcessingTimeTimer()  
    // inside the callback.    
    nextTimer = null;  
  
    InternalTimer<K, N> timer;  
	// 将所有需要触发的定时器取出，并调用对应operator的onProcessingTime方法
    while ((timer = processingTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {  
        keyContext.setCurrentKey(timer.getKey());  
        processingTimeTimersQueue.poll();  
        triggerTarget.onProcessingTime(timer);  
    }  
    // 如果timer不为空，说明当前时间下有的timer还没有到触发时间，那就循环下一轮
    if (timer != null && nextTimer == null) {  
        nextTimer =  
                processingTimeService.registerTimer(  
                        timer.getTimestamp(), this::onProcessingTime);  
    }}
```

## Watermark生成流程
### Watermark Propagation Overview
![[Watermark Propagation.png]]

### Timer Service Overview
![[Timer Service.png]]

### Watermark Processing
![[Watermark Processing.png]]
## Resources
1. https://blog.csdn.net/nazeniwaresakini/article/details/104220113
2. https://blog.csdn.net/WenWu_Both/article/details/123758708
3. [[Flink DataStream Window Sharing]]