## 继承关系
![[Slice Assigners.png]]

## 重点Class
![[Keyed Slice Assigners.png]]

## 核心逻辑
`HoppingSliceAssigner`

> [!assignSliceEnd]
> 计算数据所属的slice end, Flink SQL Window TVF的最小划分单元是slice,而之前老的版称为pane. 每个元素只属于某一个slice或者pane，但是可能属于多个window.

```Java  
public long assignSliceEnd(long timestamp) {  
    long start = TimeWindow.getWindowStartWithOffset(timestamp, offset, sliceSize);  
    return start + sliceSize;  
}
```

- getWindowStartWithOffset
```Java
/**  
 * Method to get the window start for a timestamp.
 * @param timestamp epoch millisecond to get the window start.  
 * @param offset The offset which window start would be shifted by.  
 * @param windowSize The size of the generated windows.  
 * @return window start  
 */
 public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {  
    final long remainder = (timestamp - offset) % windowSize;  
    // handle both positive and negative cases  
    if (remainder < 0) {  
        return timestamp - (remainder + windowSize);  
    } else {  
        return timestamp - remainder;  
    }}
```

> [!getWindowStart & getLastWindowEnd]
> 或者整个窗口的start time和获取整个窗口的end time(下一次要触发的窗口),并不是slice end.

```Java  
public long getLastWindowEnd(long sliceEnd) {  
    return sliceEnd - sliceSize + size;  
}  
    
public long getWindowStart(long windowEnd) {  
    return windowEnd - size;  
}
```

> [!expiredSlices]
> 在给定了一个window结束时间之后，expiredSlices会返回对于所有应该expired slice的iterator. 并且把窗口的第一个slice状态清理掉.

```Java  
public Iterable<Long> expiredSlices(long windowEnd) {  
    // we need to cleanup the first slice of the window  
    long windowStart = getWindowStart(windowEnd);  
    long firstSliceEnd = windowStart + sliceSize;  
    reuseExpiredList.reset(firstSliceEnd);  
    return reuseExpiredList;  
}
```

> [!mergeSlices]
> 将不同的slice状态合并

```Java  
public void mergeSlices(long sliceEnd, MergeCallback callback) throws Exception {  
    // the iterable to list all the slices of the triggered window  
    Iterable<Long> toBeMerged =  
            new HoppingSlicesIterable(sliceEnd, sliceSize, numSlicesPerWindow);  
    // null namespace means use heap data views, instead of state data views  
    callback.merge(null, toBeMerged);  
}
```