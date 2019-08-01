

ScheduleThreadPoolExecutor本质上还是线程池，只是使用了DelayedWorkQueue作为任务队列。对应周期性任务而已，只是每次执行完成后，重新计算下一次执行时间，然后再重新放入队列而已。如果出现异常，没有捕获的话，就无法再次放入队列，周期性任务也就无法继续执行了。

## ScheduledThreadPoolExecutor类

```java

/*
自定义核心线程数   
最大线程数为Integer.MAX_VALUE
线程空闲时间为0
使用延迟工作队列

这个线程池是一个固定容量的线程池，尽管有最大线程数，但是队列是一个无界队列。因此不能设置corePoolSize为0

*/

//构造函数
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue(), threadFactory, handler);
}



/*
这4个方法都是提交任务方法，相当于ThreadPoolExecutor的execute、submit方法；并且都有返回值，类型为ScheduledFuture，相当于普通线程池的Future，可以用于控制任务生命周期。


第1、2个是延迟任务，即延迟固定period时间后，执行任务。区别是参数不同；

第3、4个是周期性任务，
scheduleAtFixedRate是固定频率，任务时间到后就立即执行，等待周期分两种情况：如果程序的执行时间大于间隔时间，等待周期为执行时间，如果程序的执行时间小于间隔时间等待周期为间隔时间；s

cheduleWithFixedDelay是固定周期，不管执行任务需要花多长时间，下次执行必须等待固定delay时间。
*/



/*
ScheduledFutureTask从根本上实现了三个接口：Future、Runnable、Delayed。另外还持有一个Callable的成员变量对象，即上面说的把Callable对象包装成ScheduledFutureTask对象

ScheduledFutureTask相对于FutureTask而言，多实现了一个Delayed接口，这个是关键。因为ScheduledThreadPoolExecutor 的任务队列是DelayedWorkQueue，该队列要求放入的对象必须是实现了Delayed接口的RunnableScheduledFuture类型，而ScheduledFutureTask刚好实现了RunnableScheduledFuture。放入队列的入口就是delayedExecute方法。
*/


//延迟任务 执行方法，参数为Runnable类型对象  
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay,
                                   TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> t = decorateTask(command,
        new ScheduledFutureTask<Void>(command, null,
                                      triggerTime(delay, unit)));
    delayedExecute(t);
    return t;
}


//延迟任务 执行方法，参数为Callable类型对象  
 public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
     
        //把Callable对象包装成ScheduledFutureTask对象  
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
     //提交任务     
     delayedExecute(t);
        return t;
    }

//固定频率 执行方法，时间到了以后就执行  
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));  //FixedRate用正数存
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

//固定延迟周期 执行方法，上次执行完成后固定等待延迟时间后 再执行  
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit), //下次要执行的时间
                                          unit.toNanos(-delay)); //Delay的数目用负数存
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

/**
如果是延迟执行的话，就是每次执行完之后50s在执行一次，下次的执行时间就是当前时间加上delay的时间
因为delay的时间是用负数存的，所以上面的triggerTime(-p)是-p.
* Returns the trigger time of a delayed action.
*/
long triggerTime(long delay) {
return now() +
    ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}


/*

执行方法的主要方法，上面四个方法都有

这里主要的操作就是super.getQueue().add(task)，调用“任务队列”的add方法把任务放入队列。这个队列在构造方法中，已经可以看出来了，是DelayedWorkQueue。如何实现延迟任务，核心操作都在DelayedWorkQueue中实现。
*/
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())//线程池已关闭，执行拒绝策略  
        reject(task);
    else {
        super.getQueue().add(task);//调用任务队列的add方法加入队列  
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false); //取消任务  
        else
            ensurePrestart();//是否需要创建新线程  

    }
}


/**
 * Same as prestartCoreThread except arranges that at least one
 * thread is started even if corePoolSize is 0.
 至少保持有一个线程执行，从上面的 delayedExecute()可以看出，将任务之间添加到队列中去了，然后需要判断一下，是否有线程在运行，可以执行任务。
 
 */
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    if (wc < corePoolSize)
        addWorker(null, true);
    else if (wc == 0)
        addWorker(null, false);
}


/**
 * Requeues a periodic task unless current run state precludes it.
 * Same idea as delayedExecute except drops task rather than rejecting.
 *
 * @param task the task
 */
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
    if (canRunInCurrentRunState(true)) {
        super.getQueue().add(task);	//将该任务重新添加到队列中
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}

```



## DelayedWorkQueue类

```java

public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;   
    lock.lockInterruptibly();   //加上锁，一次只能一个线程取
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];                    //取出头结点  
            if (first == null)
                available.await();//如果队列为空，就阻塞  
            else {
                long delay = first.getDelay(NANOSECONDS);     //获取头结点的，需要等待的时间  

                if (delay <= 0)  //早就该取出来的，现在已经迟到了，马上取走，否则需要等待
                    return finishPoll(first);//取出任务，并重写计算二叉堆
                first = null; // don't retain ref while waiting  在等待的时候不保持引用
                if (leader != null)
                    available.await();//阻塞等待其他线程take完成
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;//当前线程作为leader
                    try {
                        available.awaitNanos(delay);//阻塞等待delay时间  
                    } finally {
                        if (leader == thisThread)
                            leader = null;//取消当前线程作为leader
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();   //继续唤醒其他线程take  
        lock.unlock();
    }
}
```





## ScheduledFutureTask类

```java


//任务下一次执行间隔时间  
//初始延迟时间 和 period计算得到  
  private long time;

        /**
         * Period in nanoseconds for repeating tasks.  A positive
         * value indicates fixed-rate execution.  A negative value
         * indicates fixed-delay execution.  A value of 0 indicates a
         * non-repeating task.
         */
//重复任务的周期    正数代表固定频率的执行   负数代表固定的延迟执行   0代表不重复
  private final long period;


//计算这次要是执行该任务，还需要等多长时间。   用应该执行的时间减去当前时间，就是要等待的时间
public long getDelay(TimeUnit unit) {
            return unit.convert(time - now(), NANOSECONDS);
}

/*
'compareTo方法 
用于二叉堆算法的排序
就是比较time的大小

'*/
public int compareTo(Delayed other) {
    if (other == this) // compare zero if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
    return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
}


/**
   判断是否是周期任务
 * Returns {@code true} if this is a periodic (not a one-shot) action.
 *
 * @return {@code true} if periodic
 */

public boolean isPeriodic() {
    return period != 0;
}

/**

设置下次的运行时间

 * Sets the next time to run for a periodic task.
 */
private void setNextRunTime() {
    long p = period;
    
    //如果是正数代表固定频率的执行 例如每一分钟执行一次  这次执行完之后,下次的执行时间就是上次执行的时间加上频率执行的时间，不管这次执行的时间多长。如果这次执行的时间过长，超过了预期的时间也没事。
    
    if (p > 0)
        time += p;
    else
        time = triggerTime(-p);
}




/**
 * Overrides FutureTask version so as to reset/requeue if periodic.
 */
public void run() {
    boolean periodic = isPeriodic();   //判断是不是 周期性任务 
    if (!canRunInCurrentRunState(periodic))  
        cancel(false);
    else if (!periodic)  //如果只是延时任务，只需执行一次
        ScheduledFutureTask.super.run();     //调用FutureTask的run()方法，执行callbe任务
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();   //周期性任务，需要计算下个周期执行时间  
        reExecutePeriodic(outerTask);  //把任务重新放入队列  
    }
}

```



