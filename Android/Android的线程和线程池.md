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
