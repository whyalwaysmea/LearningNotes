# subscribeOn()和observeOn()的区别
* subscribeOn()改变调用它之前代码的线程
* observeOn()改变调用它之后代码的线程

# RxJava中的scheduler
* Schedulers.immediate(): 直接在当前线程运行，相当于不指定线程。这是默认的 Scheduler。
* Schedulers.newThread(): 总是启用新线程，并在新线程执行操作。
* Schedulers.io(): I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。
* Schedulers.computation(): 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
* Scheduler.trampoline 把任务放到当前线程的队列中，等当前任务执行完了，再继续执行队列中的任务
* Scheduler.test 用于测试和调试
* 另外， Android 还有一个专用的 AndroidSchedulers.mainThread()，它指定的操作将在 Android 主线程运行。

# subscribeOn()
我们先看一个示例：
```java
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        subscriber.onNext(1);
        subscriber.onNext(2);
        subscriber.onCompleted();
    }
}).map(new Func1<Integer, String>() {
    @Override
    public String call(Integer integer) {
        System.out.println("map在线程" + Thread.currentThread().getName() + "中");
        return integer + "";
    }
}).subscribeOn(Schedulers.newThread())
    .subscribe(new Subscriber<String>() {
        @Override
        public void onCompleted() {

        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onNext(String s) {
            System.out.println("onNext在线程" + Thread.currentThread().getName() + "中");

        }
    });

//打印结果如下
12-28 20:39:16.361 5015-5031/? I/System.out: map在线程RxNewThreadScheduler-1中
12-28 20:39:16.361 5015-5031/? I/System.out: onNext在线程RxNewThreadScheduler-1中
12-28 20:39:16.361 5015-5031/? I/System.out: map在线程RxNewThreadScheduler-1中
12-28 20:39:16.361 5015-5031/? I/System.out: onNext在线程RxNewThreadScheduler-1中
```
所以我们可以先得出一个结论：*subscribeOn操作符会影响在subscribeOn上面的代码的执行线程*
现在我们需求变了，发射的时候我们要运行在IO线程，但是map操作符的变换我们想在安卓的主线程执行！
想想似乎比较简单，就改成以下代码：
```java
Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                subscriber.onNext(1);
                subscriber.onNext(2);
                subscriber.onCompleted();
            }
        }).subscribeOn(Schedulers.io())
                .map(new Func1<Integer, String>() {
                    @Override
                    public String call(Integer integer) {
                        System.out.println("map在线程" + Thread.currentThread().getName() + "中");
                        return integer + "";
                    }
                }).subscribeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(String s) {
                        System.out.println("onNext在线程" + Thread.currentThread().getName() + "中");

                    }
                });
// 打印结果
12-28 20:44:57.133 10076-10092/? I/System.out: map在线程RxIoScheduler-2中
12-28 20:44:57.133 10076-10092/? I/System.out: onNext在线程RxIoScheduler-2中
12-28 20:44:57.133 10076-10092/? I/System.out: map在线程RxIoScheduler-2中
12-28 20:44:57.133 10076-10092/? I/System.out: onNext在线程RxIoScheduler-2中
```
说好的subscribeOn会改变上面代码的执行线程嘛？？？怎么不对了
>同一个Observable，无论你call多少次subscribeOn，只有第一个subscribeOn会起作用。


# observeOn()
```java
Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                subscriber.onNext(1);
                subscriber.onNext(2);
                subscriber.onCompleted();
            }
        })
        .subscribeOn(Schedulers.newThread())
        .observeOn(Schedulers.io())
        .map(new Func1<Integer, String>() {
            @Override
            public String call(Integer integer) {
                System.out.println("map1在线程" + Thread.currentThread().getName() + "中");
                return integer + "";
            }
        })
        .observeOn(Schedulers.computation())
        .map(new Func1<String, String>() {
            @Override
            public String call(String s) {
                System.out.println("map2在线程" + Thread.currentThread().getName() + "中");

                return s + "1";
            }
        })
        .observeOn(Schedulers.newThread())
        .subscribe(new Subscriber<String>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext在线程" + Thread.currentThread().getName() + "中");

            }
        });

// 打印结果
12-28 21:00:32.433 23980-24011/com.xlzg.yishuxiyi I/System.out: map1在线程RxIoScheduler-2中
12-28 21:00:32.437 23980-24008/com.xlzg.yishuxiyi I/System.out: map2在线程RxComputationScheduler-1中
12-28 21:00:32.437 23980-24010/com.xlzg.yishuxiyi I/System.out: onNext在线程RxNewThreadScheduler-1中
12-28 21:00:32.437 23980-24011/com.xlzg.yishuxiyi I/System.out: map1在线程RxIoScheduler-2中
12-28 21:00:32.437 23980-24008/com.xlzg.yishuxiyi I/System.out: map2在线程RxComputationScheduler-1中
12-28 21:00:32.441 23980-24010/com.xlzg.yishuxiyi I/System.out: onNext在线程RxNewThreadScheduler-1中        
```
*每次使用observeOn，都会把这个observeOn下面的操作包装在该observeOn指定的线程池中运行。*

# 结论
如果我们有一段这样的序列
```
Observable
.map                    // 操作1
.flatMap                // 操作2
.subscribeOn(io)
.map                    //操作3
.flatMap                //操作4
.observeOn(main)
.map                    //操作5
.flatMap                //操作6
.subscribeOn(io)        //!!特别注意
.subscribe(handleData)
```
假设这里我们是在主线程上调用这段代码，
那么
>操作1，操作2是在io线程上，因为之后subscribeOn切换了线程
操作3，操作4也是在io线程上，因为在subscribeOn切换了线程之后，并没有发生改变。
操作5，操作6是在main线程上，因为在他们之前的observeOn切换了线程。
特别注意那一段，对于操作5和操作6是无效的
再简单点总结就是
subscribeOn的调用切换之前的线程。
observeOn的调用切换之后的线程。
observeOn之后，不可再调用subscribeOn 切换线程
