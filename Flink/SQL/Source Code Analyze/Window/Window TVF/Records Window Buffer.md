## 继承关系
![[Records Window Buffer.png]]

## 核心逻辑
- addElement
```Java  
public void addElement(RowData key, long sliceEnd, RowData element) throws Exception {  
    // track the lowest trigger time, if watermark exceeds the trigger time,  
    // it means there are some elements in the buffer belong to a window going to be fired,    
    // and we need to flush the buffer into state for firing.    
    minSliceEnd = Math.min(sliceEnd, minSliceEnd);  
  
    reuseWindowKey.replace(sliceEnd, key);  
    LookupInfo<WindowKey, Iterator<RowData>> lookup = recordsBuffer.lookup(reuseWindowKey);  
    try {  
        recordsBuffer.append(lookup, recordSerializer.toBinaryRow(element));  
    } catch (EOFException e) {  
        // buffer is full, flush it to state  
        flush();  
        // remember to add the input element again  
        addElement(key, sliceEnd, element);  
    }}
```

- flush
```Java  
public void flush() throws Exception {  
    if (recordsBuffer.getNumKeys() > 0) {  
        KeyValueIterator<WindowKey, Iterator<RowData>> entryIterator =  
                recordsBuffer.getEntryIterator(requiresCopy);  
        while (entryIterator.advanceNext()) {  
            combineFunction.combine(entryIterator.getKey(), entryIterator.getValue());  
        }  
        recordsBuffer.reset();  
        // reset trigger time  
        minSliceEnd = Long.MAX_VALUE;  
    }}
```

- advanceProgress
```Java  
public void advanceProgress(long progress) throws Exception {  
    if (isWindowFired(minSliceEnd, progress, shiftTimeZone)) {  
        // there should be some window to be fired, flush buffer to state first  
        flush();  
    }}
```