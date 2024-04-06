## 继承关系
![[Slicing Window Operator.png]]


## 核心逻辑
```Java
// 处理每一条数据
public void processElement(StreamRecord%3CRowData%3E element) throws Exception {  
    RowData inputRow = element.getValue();  
    RowData currentKey = (RowData) getCurrentKey();  
    boolean isElementDropped = windowProcessor.processElement(currentKey, inputRow);  
    if (isElementDropped) {  
        // markEvent will increase numLateRecordsDropped  
        lateRecordsDroppedRate.markEvent();  
    }}  

// 处理watermark  
public void processWatermark(Watermark mark) throws Exception {  
    if (mark.getTimestamp() > currentWatermark) {  
        windowProcessor.advanceProgress(mark.getTimestamp());  
        super.processWatermark(mark);  
    } else {  
        super.processWatermark(new Watermark(currentWatermark));  
    }}  

// event time trigger  
public void onEventTime(InternalTimer<K, W> timer) throws Exception {  
    onTimer(timer);  
}  

// processing time trigger  
public void onProcessingTime(InternalTimer<K, W> timer) throws Exception {  
    if (timer.getTimestamp() > lastTriggeredProcessingTime) {  
        // similar to the watermark advance,  
        // we need to notify WindowProcessor first to flush buffer into state        
        lastTriggeredProcessingTime = timer.getTimestamp();  
        windowProcessor.advanceProgress(timer.getTimestamp());  
        // timers registered in advanceProgress() should always be smaller than current timer  
        // so, it should be safe to trigger current timer straightforwards.    }  
    onTimer(timer);  
}  
  
private void onTimer(InternalTimer<K, W> timer) throws Exception {  
    setCurrentKey(timer.getKey());  
    W window = timer.getNamespace();  
    windowProcessor.fireWindow(window);  
    windowProcessor.clearWindow(window);  
    // we don't need to clear window timers,  
    // because there should only be one timer for each window now, which is current timer.}>)
```