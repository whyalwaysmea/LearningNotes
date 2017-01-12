# Android的线程
在Java中默认情况下一个进程只有一个线程，也就是主线程，其他线程都是子线程，也叫工作线程。Android中的主线程主要处理和界面相关的事情，而子线程则往往用于执行耗时操作。线程的创建和销毁的开销较大，所以如果一个进程要频繁地创建和销毁线程的话，都会采用线程池的方式。

在Android中除了Thread，还有HandlerThread、AsyncTask以及IntentService等也都扮演着线程的角色，只是它们具有不同的特性和使用场景。
>AsyncTask封装了线程池和Handler，它主要是为了方便开发者在子线程中更新UI。
HandlerThread是一种具有消息循环的线程，在它的内部可以使用Handler。
IntentService是一个服务，它内部采用HandlerThread来执行任务，当任务执行完毕后就会自动退出。因为它是服务的缘故，所以和后台线程相比，它比较不容易被系统杀死。

# Android中的线程形态
## AsyncTask
AsyncTask是一种轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给主线程并在主线程中更新UI。
从实现上来说，AsyncTask封装了Thread和Handler，但是AsyncTask并不适合进行特别耗时的任务。
```Java
/**
* Params: 表示参数的类型
* Progress: 表示后台任务的执行进度的类型
* Result: 表示后台任务的返回结果的类型
*/
public abstract class AsyncTask<Params, Progress, Result> {

    // 在主线程中执行，在异步任务执行之前，此方法会被调用，一般可以用于做一些准备工作
    protected void onPreExecute();

    // 在线程池中执行。此方法用于执行异步任务，params参数表示异步任务的输入参数。
    // 在此方法中可以通过publicProgress方法来更新任务的进度，publicProgress方法会调用onProgressUpdate方法
    // 此方法需要返回计算结果给onPostExecute方法
    protected abstract Result doInBackground(Params... params);

    // 在主线程中执行，当后台任务的执行进度发生改变时此方法会被调用
    protected void onProgressUpdate(Progress.. values);

    // 在主线程中执行，在异步任务执行之后，此方法会被调用，其中result参数是后台任务的返回值，即doInBackground的返回值
    protected void onPostExecute(Result result);
}
```
AsyncTask在具体的使用过程中还有一些限制

1. AsyncTask的类必须在主线程中加载，这个过程在Android 4.1及以上版本中已经被系统自动完成。
2. AsyncTask对象必须在主线程中创建，
3. execute方法必须在UI线程中调用。
4. 不要在程序中直接调用onPreExecute()、onPostExecute()、doInBackground()和onProgressUpdate()方法
5. 一个AsyncTask对象只能执行一次，即只能调用一次execute方法，否则会报运行时异常。
6. 在Android 1.6之前，AsyncTask是串行执行任务的，Android 1.6的时候AsyncTask开始采用线程池并行处理任务，但是从Android 3.0开始，为了避免AsyncTask带来的并发错误，AsyncTask又采用一个线程来串行执行任务。尽管如此，在Android 3.0以及后续版本中，我们可以使用AsyncTask的executeOnExecutor方法来并行执行任务。但是这个方法是Android 3.0新添加的方法，并不能在低版本上使用。

### AsyncTask的工作原理
为了分析AsyncTask的工作原理，我们从它的execute方法开始分析，该方法又会调用executeOnExecutor
```Java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    // sDefaultExecutor实际上是一个串行的线程池
    // 一个进程中所有的AsyncTask全部在这个串行的线程池中排队执行
    return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
    // 状态检查，一个AsyncTask对象只能执行一次，即只能调用一次execute方法
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }
    // 状态标记
    mStatus = Status.RUNNING;
    // 可以看出，该方法最先执行
    onPreExecute();

    mWorker.mParams = params;
    // 线程池开始执行
    exec.execute(mFuture);

    return this;
}
```
```Java
/**
 * An {@link Executor} that executes tasks one at a time in serial
 * order.  This serialization is global to a particular process.
 */
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;
    // 这里的Runnable，就是exec.execute(mFuture);的FutureTask
    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        // 从mTasks队列中取出队首的一个Runnable任务叫做mActive
        if ((mActive = mTasks.poll()) != null) {
            // 这里就进入线程池中去完成任务了
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

1. 系统先把AsyncTask的Params的参数封装成为FutureTask对象，FutureTask是一个并发类，在这里它充当了Runnable的作用。
2. 接着这个FutureTask会交给SerialExecutor的execute方法去处理
3. execute方法首先会把FutureTask对象插入到任务队列mTasks中，如果这个时候没有正在活动的AsyncTask任务，那么就会调用SerialExecutor的scheduleNext方法来执行下一个AsyncTask任务。
4. 同时当一个AsyncTask任务执行完后，AsyncTask会继续执行其他任务直到所有的任务都被执行为止。

在默认情况下，AsyncTask是串行执行的。

AsyncTask中有两个线程池：SerialExecutor和THREAD_POOL_EXECUTOR。前者是用于任务的排队，默认是串行的线程池；后者用于真正执行任务。
AsyncTask中还有一个Handler，即InternalHandler，用于将执行环境从线程池切换到主线程。
AsyncTask内部就是通过InternalHandler来发送任务执行的进度以及执行结束等消息。
在AsyncTask的构造方法中有如下一段代码，由于FutureTask的run方法会调用mWorker的call方法，因此mWorker的call方法最终会在线程池中执行
```Java
mWorker = new WorkerRunnable<Params, Result>() {
    public Result call() throws Exception {
        // 表明当前任务已经被调用过了
        mTaskInvoked.set(true);
        // 将这个线程的优先级设为后台线程优先级
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        //noinspection unchecked
        Result result = doInBackground(mParams);
        Binder.flushPendingCommands();
        return postResult(result);
    }
};

private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}

// 为了能够执行线程切换，所以AsyncTask就必须在主线程中加载
private static InternalHandler sHandler;

private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}

// 最后任务执行完了会调用该方法
private void finish(Result result) {
    // 如果任务被取消
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
### 为什么AsyncTask不适合进行特别耗时的后台任务
首先，默认的AsyncTask的线程池中的核心线程数是有限的，不管是和CPU数目有关还是以前的固定的5， 还是有一个数量的限制，因此不适合大量的后台任务处理，例如瀑布流图片的加载等。

其次，AsyncTask类必须在主线程初始化，必须在主线程创建，因为return executeOnExecutor(sDefaultExecutor, params)这里也只能在UI线程走。

最后，我们也讲到了，AsyncTask在3.0后改成了串行的，因此想真正做一些并行的后台任务，就不太适合了。

## HandlerThread
HandlerThread就是一种可以使用Handler的Thread，它的实现就是在run方法中通过Looper.prepare()来创建消息队列，并通过Looper.loop()来开启消息循环，这样在实际的使用中就允许在HandlerThread中创建Handler了，外界可以通过Handler的消息方式通知HandlerThread执行一个具体的任务。

HandlerThread的run方法是一个无限循环，因此当明确不需要再使用HandlerThread的时候，可以通过它的quit或者quitSafely方法来终止线程的执行。

HandlerThread的最主要的应用场景就是用在IntentService中。

## IntentService
IntentService是一个继承自Service的抽象类，要使用它就要创建它的子类。因为IntentService是服务的原因，适合执行一些高优先级的后台任务，这样不容易被系统杀死。

在实现上，IntentService封装了HandlerThread和Handler。可以在它的onCreate方法中看出来
```Java
@Override
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.

    super.onCreate();
    // 这个mName就是构造方法中传进来的
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    mServiceLooper = thread.getLooper();
    // mServiceHandler发送的消息最终都会在HandlerThread中执行
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```
每次启动IntentService，它的onStartCommand方法就会调用一次，IntentService在onStartCommand中处理每个后台任务的IntentService
```Java
@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```
可以看出来，仅仅是同归mServiceHandler发送了一个消息，这个消息会在HandlerThread中被处理。
mServiceHandler收到消息后，会将Intent对象传递给onHandleIntent方法去处理。所以IntentService的子类都要是实现这个方法。
```Java
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        // 可以看出，当消息处理完后，调用了stopSelf(int startId)来停止服务
        stopSelf(msg.arg1);
    }
}
```
这里之所以采用stopSelf(int startId)而不是stopSelf()来停止服务，是因为stopSelf()会立刻停止服务，而这个时候可能还有其他消息未处理，stopSelf(int startId)则会等待所有的消息都处理完毕后才终止服务。
IntentService是顺序执行后台任务。


## Android中的线程池
使用线程池的好处：

1. 重用线程，避免线程的创建和销毁带来的性能开销；
2. 能有效控制线程池的最大并发数，避免大量的线程之间因互相抢占系统资源而导致的阻塞现象；
3. 能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能。

Android中的线程池来源于Java中的Executor,Executor是一个接口，真正的线程池是ThreadPoolExecutor。ThreadPoolExecutor提供了一系列参数来配置线程池，通过不同的参数可以创建不同的线程池，Android的线程池都是通过Executors提供的工厂方法得到的。

### ThreadPoolExecutor
ThreadPoolExecutor是线程池的真正实现，它的构造函数提供了一系列参数来配置线程池。
```Java
/**
* corePoolSize: corePoolSize 核心线程池大小，也就是线程池中可以维持的线程数目，即使这些线城是空闲的，也不会终止（除非设置了allowCoreThreadTimeOut），因此可以理解为常驻线程池的线程数目。
* maximumPoolSize: 最大线程数，当活动线程数达到这个数值后，后续的任务将会被阻塞；
* keepAliveTime: 非核心线程闲置时的超时时长，超过这个时长，闲置的非核心线程就会被回收；当设置了allowCoreThreadTimeOut后，keepAliveTime同样会作用于核心线程
* unit：用于指定keepAliveTime参数的时间单位，有TimeUnit.MILLISECONDS、TimeUnit.SECONDS、TimeUnit.MINUTES等；
* workQueue： 任务队列，通过线程池的execute方法提交的Runnable对象会存储在这个参数中；
* threadFactory： 线程工厂，为线程池提供创建新线程的功能。它是一个接口，它只有一个方法Thread newThread(Runnable r)；
* RejectedExecutionHandler：当线程池无法执行新任务时，可能是由于任务队列已满或者是无法成功执行任务，这个时候就会调用这个Handler的rejectedExecution方法来通知调用者，默认情况下，rejectedExecution会直接抛出一个rejectedExecutionException。
*/
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {

    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}
```
ThreadPoolExecutor执行任务的规则：

1. 如果线程池中的线程数未达到核心线程的数量，那么会直接启动一个核心线程来执行任务；
2. 如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行；
3. 如果在步骤2中无法将任务插入到的任务队列中，可能是任务队列已满，这个时候如果线程数量没有达到规定的最大值，那么会立刻启动非核心线程来执行这个任务；
4. 如果步骤3中线程数量已经达到线程池规定的最大值，那么就拒绝执行此任务，ThreadPoolExecutor会调用RejectedExecutionHandler的rejectedExecution方法来通知调用者。

我们现在来看看ThreadPoolExecutor的execute方法
```Java
public void execute(Runnable command) {
    // 判断提交的任务command是否为null，若是null，则抛出空指针异常；
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    // 这个int值保存了线程池中任务数目和线程池的状态等信息
    int c = ctl.get();
    // 当线程池中的任务数量 < 核心线程数目时，
    if (workerCountOf(c) < corePoolSize) {
        // 通过addWorker(command, true)新建一个线程，并将任务(command)添加到该线程中，然后启动该线程来执行任务。
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 二次检查
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

#### AsyncTask中的ThreadPoolExecutor
AsyncTask中的ThreadPoolExecutor的配置
```Java
public static final Executor THREAD_POOL_EXECUTOR
           = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                   TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
// 核心线程数等于CPU核心数+1
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
// 线程池的最大线程数为CPU核心数*2 + 1
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
// 核心线程无超时机制，非核心线程闲置超时时间为1秒
private static final int KEEP_ALIVE = 1;
// 任务队列的容量为128
private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);                
```


### 线程池的分类
Android中常见的4类具有不同功能特性的线程池，它们都是直接或间接地通过配置ThreadPoolExecutor来实现自己的功能特性。

1. FixedThreadPool：线程数量固定的线程池，它只有核心线程，且核心线程不会被回收；意味着它能够更加 快速地响应外界的请求
2. CachedThreadPool：线程数量不固定的线程池，它只有非核心线程；比较适合执行大量的耗时较少的任务
3. ScheduledThreadPool：核心线程数量固定，非核心线程数量没有限制的线程池，并且非核心线程闲置时会被立即回收。主要用于执行定时任务和具有固定周期的任务；
4. SingleThreadPool：只有一个核心线程的线程池，确保了所有的任务都在同一个线程中按顺序执行。
