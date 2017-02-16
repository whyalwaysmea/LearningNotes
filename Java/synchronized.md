有时候在多线程操作的时候，会出现线程同步的问题。
```java
public class CounterThread extends Thread {
    private static int counter = 0;

    @Override
    public void run() {
        try {
            Thread.sleep((int)(Math.random()*100));
        } catch (InterruptedException e) {
        }
        counter ++;
    }


    public static void main(String[] args) throws InterruptedException {
        int num = 1000;
        Thread[] threads = new Thread[num];
        for(int i=0; i<num; i++){
            threads[i] = new CounterThread();
            threads[i].start();
        }

        for(int i=0; i<num; i++){
            threads[i].join();
        }

        System.out.println(counter);
    }
}
```
这段代码容易理解，有一个共享静态变量counter，初始值为0，在main方法中创建了1000个线程，每个线程就是随机睡一会，然后对counter加1，main线程等待所有线程结束后输出counter的值。    
为什么会这样呢？因为counter++这个操作不是原子操作，它分为三个步骤：   
1. 取counter的当前值
2. 在当前值基础上加1
3. 将新值重新赋值给counter
两个线程可能同时执行第一步，取到了相同的counter值，比如都取到了100，第一个线程执行完后counter变为101，而第二个线程执行完后还是101，最终的结果就与期望不符。

怎么解决这个问题呢？有多种方法：   
* 使用synchronized关键字
* 使用显式锁
* 使用原子变量


## 解决线程同步问题
针对上面的问题，我们可以使用synchronized关键字来解决线程同步问题。   
synchronized可以用于修饰类的实例方法、静态方法和代码块。
为了处理上面的问题，我们可以用synchronized来修饰代码块：
```java
// 这里我们只需要修改run
public void run() {
    try {
        Thread.sleep((int)(Math.random()*100));
    } catch (InterruptedException e) {
    }
    synchronized (CounterThread.class) {
    	counter ++;
	}
}
```
可以看到，我们用synchronized把`counter++`代码块给框起来了。  
synchronized括号里面的就是保护的对象   

synchronized保护的是对象而非代码，只要访问的是同一个对象的synchronized方法，即使是不同的代码，也会被同步顺序访问

## Q&A
### synchronized 和 volatile 关键字的作用
1. volatile 仅能使用在变量级别；  synchronized 则可以使用在实例方法、静态方法和代码块。
2. volatile 仅能实现变量的修改可见性，并不能保证原子性；synchronized 则可以保证变量的修改可见性和原子性
3. volatile 不会造成线程的阻塞；synchronized 可能会造成线程的阻塞。
4. volatile 标记的变量不会被编译器优化；  synchronized 标记的变量可以被编译器优化
