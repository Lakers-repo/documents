## 继承关系
![[Processing Timer Service.png]]

## 核心逻辑

### SystemProcessingTimeService
- registerTimer
```Java
public ScheduledFuture<?> registerTimer(long timestamp, ProcessingTimeCallback callback) {  
  
    long delay =  
            ProcessingTimeServiceUtil.getProcessingTimeDelay(  
                    timestamp, getCurrentProcessingTime());  
  
    // we directly try to register the timer and only react to the status on exception  
    // that way we save unnecessary volatile accesses for each timer    
    try {  
        return timerService.schedule(  
                wrapOnTimerCallback(callback, timestamp), delay, TimeUnit.MILLISECONDS);  
    } catch (RejectedExecutionException e) {  
        final int status = this.status.get();  
        if (status == STATUS_QUIESCED) {  
            return new NeverCompleteFuture(delay);  
        } else if (status == STATUS_SHUTDOWN) {  
            throw new IllegalStateException("Timer service is shut down");  
        } else {  
            // something else happened, so propagate the exception  
            throw e;  
        }  
    }  
}
```

- 通过wrapOnTimerCallback方法将ProcessingTimeCallback与Runnable绑定起来
```Java
private Runnable wrapOnTimerCallback(ProcessingTimeCallback callback, long timestamp) {  
    return new ScheduledTask(status, exceptionHandler, callback, timestamp, 0);  
}

private static final class ScheduledTask implements Runnable {  
    private final AtomicInteger serviceStatus;  
    private final ExceptionHandler exceptionHandler;  
    private final ProcessingTimeCallback callback;  
  
    private long nextTimestamp;  
    private final long period;  
  
    ScheduledTask(  
            AtomicInteger serviceStatus,  
            ExceptionHandler exceptionHandler,  
            ProcessingTimeCallback callback,  
            long timestamp,  
            long period) {  
        this.serviceStatus = serviceStatus;  
        this.exceptionHandler = exceptionHandler;  
        this.callback = callback;  
        this.nextTimestamp = timestamp;  
        this.period = period;  
    }  
    @Override  
    public void run() {  
        if (serviceStatus.get() != STATUS_ALIVE) {  
            return;  
        }  
        try {  
            callback.onProcessingTime(nextTimestamp);  
        } catch (Exception ex) {  
            exceptionHandler.handleException(ex);  
        }  
        nextTimestamp += period;  
    }}
```

### ProcessingTimeCallback
```Java
/**  
 * A service that allows to get the current processing time and register timers that will execute * the given {@link ProcessingTimeCallback} when firing.  
 */
public interface ProcessingTimeService {  
    /** 
     * Returns the current processing time. 
     */  
    long getCurrentProcessingTime();  
  
    /**  
     * Registers a task to be executed when (processing) time is {@code timestamp}.  
     * @param timestamp Time when the task is to be executed (in processing time)  
     * @param target The task to be executed  
     * @return The future that represents the scheduled task. This always returns some future, even  
     * if the timer was shut down     
     * */    
    ScheduledFuture<?> registerTimer(long timestamp, ProcessingTimeCallback target);  
  
    /**  
     * A callback that can be registered via {@link #registerTimer(long, ProcessingTimeCallback)}.  
     */    
    interface ProcessingTimeCallback {  
        /**  
         * This method is invoked with the time which the callback register for.         
         * @param time The time this callback was registered for.  
         */        
         void onProcessingTime(long time) throws IOException, InterruptedException, Exception;  
    }  
}
```

### TimestampsAndWatermarksOperator - 周期性发送watermark
```Java
public void onProcessingTime(long timestamp) throws Exception {  
    watermarkGenerator.onPeriodicEmit(wmOutput);  
  
    final long now = getProcessingTimeService().getCurrentProcessingTime();  
    getProcessingTimeService().registerTimer(now + watermarkInterval, this);  
}
```

## Resources
1. https://blog.csdn.net/WenWu_Both/article/details/123723037?spm=1001.2014.3001.5502
2. https://www.cnblogs.com/gentlescholar/p/15061000.html