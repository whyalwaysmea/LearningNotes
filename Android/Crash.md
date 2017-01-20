## 简介
### 什么是Crash
Crash就是由于代码异常而导致App非正常退出现象，也就是我们常说的『崩溃』
Android 应用不可避免得会发生 Crash，也叫做做崩溃。收集这些异常信息并把它们发送都服务器是尤为重要的。  

### Android中有哪些类型Crash
通常情况下会有以下两种类型Crash：  
* Java Crash
* Native Crash

## Java Crash
Java Crash在Android上的特点：
* 这类错误一般是由Java层代码触发的
* 一般情况下程序出错时会弹出提示框，JVM虚拟机退出
* 一般的Crash工具都能够捕获，系统也提供了API

### 解决办法
Android提供了解决此类问题的方法，Thread 类中的一个方法是 setDefaultExceptionHandler，作用就是设置默认异常处理器。  

当 crash 发生的时候，系统会回调UncaughtExceptionHandler 的 uncaughtException 方法，在这个方法中就可以选择把异常信息储存在 SD 卡中，然后在合适的时机通过网络将 crash 上传到服务器。  

所以，只需要实现一个 UncaughtExceptionHandler 对象，在它的 uncaughtException 方法中获取异常信息并储存在SD卡上以及上传到服务器供开发人员分析，然后调用 Thread 的 setDefaultExceptionHandler 方法把它设置为默认的异常处理器就可以了。

### 异常处理器的实现
```java
public class CrashHandler implements Thread.UncaughtExceptionHandler {

    private static final String TAG = "CrashHandler";
    private static final boolean DEBUG = true;

    private static final String PATH = Environment.getExternalStorageDirectory().
            getPath() + "/CrashTest/log/";
    private static final String FILE_NAME = "crash";
    private static final String FILE_NAME_SUFFIX = "trace";

    private static CrashHandler sInstance = new CrashHandler();
    private Thread.UncaughtExceptionHandler mDefaultCrashHandler;
    private Context mContext;

    private CrashHandler() {
    }

    public static CrashHandler getInstance() {
        return sInstance;
    }

    public void init(Context context) {
        mDefaultCrashHandler =  Thread.getDefaultUncaughtExceptionHandler();
        Thread.setDefaultUncaughtExceptionHandler(this);
        mContext = context.getApplicationContext();
    }

    /**
     * 当程序中有未被捕获的异常，系统会自动调用 uncaughtException 方法
     *
     * @param thread    出现未捕获异常的线程
     * @param throwable 未捕获的异常
     */
    @Override
    public void uncaughtException(Thread thread, Throwable throwable) {
        try {
            // 导出异常信息到SD卡
            dumpExceptionToSDCard(throwable);
            // 上传异常信息到服务器
            uploadExceptionToServer();
        } catch (IOException e) {
            e.printStackTrace();
        }

        throwable.printStackTrace();
        //如果系统提供了默认异常处理器，则交给系统去结束程序，否则就自己结束程序
        if (mDefaultCrashHandler != null) {
            mDefaultCrashHandler.uncaughtException(thread, throwable);
        } else {
            Process.killProcess(Process.myPid());
        }
    }

    /**
     * 将异常信息写入SD卡中
     *
     * @param throwable
     * @throws IOException
     */
    private void dumpExceptionToSDCard(Throwable throwable) throws IOException {
        //如果 SD 卡不存在或无法使用，则无法把异常信息写入SD卡中
        if (!Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)) {
            if (DEBUG) {
                Log.w(TAG, "sdcard unmounted,skip dump exception");
                return;
            }
        }

        File dir = new File(PATH);
        if (!dir.exists()) {
            dir.mkdirs();
        }
        long current = System.currentTimeMillis();
        String time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date(current));
        File file = new File(PATH + FILE_NAME + time + FILE_NAME_SUFFIX);
        try {
            PrintWriter pw = new PrintWriter(new BufferedWriter(new FileWriter(file)));
            pw.println(time);
            dumpPhoneInfo(pw);
            pw.println();
            throwable.printStackTrace(pw);
            pw.close();
        } catch (IOException e) {
            Log.e(TAG, "dump crash info failed");
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }


    }

    /**
     * 将手机信息写入文件中
     *
     * @param pw
     * @throws PackageManager.NameNotFoundException
     */
    private void dumpPhoneInfo(PrintWriter pw) throws PackageManager.NameNotFoundException {

        PackageManager pm = mContext.getPackageManager();
        PackageInfo pi = pm.getPackageInfo(mContext.getPackageName(), PackageManager.GET_ACTIVITIES);
        pw.write("App Version：");
        pw.print(pi.versionName);
        pw.print('_');
        pw.println(pi.versionCode);

        //Android 版本号
        pw.print("OS Version： ");
        pw.print(Build.VERSION.RELEASE);
        pw.print('_');
        pw.println(Build.VERSION.SDK_INT);

        //手机制造商
        pw.print("Model： ");
        pw.println(Build.MANUFACTURER);

        //手机型号
        pw.print("Model： ");
        pw.println(Build.MODEL);

        //CPU 架构
        pw.print("CPU ABI：");
        pw.println(Build.CPU_ABI);

    }

    private void uploadExceptionToServer() {
        //把文件上传到服务器的操作
    }
}
```
因为这里涉及到了往SD卡写数据的操作，所以需要相应的权限
```xml
<!--往sdcard中写入数据的权限 -->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"></uses-permission>
<!--在sdcard中创建/删除文件的权限 -->
<uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"></uses-permission>
```

### 使用
可以在 Application 初始化的时候为线程设置 CrashHandler，如下所示
```java
public class App extends Application {

    private static App sInstance;

    @Override
    public void onCreate() {
        super.onCreate();
        sInstance = this;
        CrashHandler crashHandler = CrashHandler.getInstance();
        crashHandler.init(this);
    }

    public static App getInstance() {
        return sInstance;
    }
}
```

### 小结
关于Java Crash的分析已经介绍完了，相对还是比较简单，通过简单的方式就能够捕获到异常，但别忘了，Android最头痛的不是这种异常，而是Native层的异常，有时候就算能让你拿到堆栈信息你也不一定会解决问题，比如你使用了第三方的so库，如果发生崩溃了，你也会崩溃的。

## Native Crash
[Android Crash之Native Crash分析](http://www.jianshu.com/p/1d3e8f251c9c)

## 崩溃重启

```java
public class App extends Application  {

    protected static App instance;
    public void onCreate() {
        super.onCreate();
        instance = this;
        Thread.setDefaultUncaughtExceptionHandler(restartHandler); // 程序崩溃时触发线程  以下用来捕获程序崩溃异常
    }
    // 创建服务用于捕获崩溃异常
    private Thread.UncaughtExceptionHandler restartHandler = new Thread.UncaughtExceptionHandler() {
        public void uncaughtException(Thread thread, Throwable ex) {
            restartApp();//发生崩溃异常时,重启应用
        }
    };
    public void restartApp(){
        Intent intent = new Intent(instance,MainActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        instance.startActivity(intent);
        android.os.Process.killProcess(android.os.Process.myPid());  //结束进程之前可以把你程序的注销或者退出代码放在这段代码之前
    }
}
```
