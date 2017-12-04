```java
public static void main(String[] args) {
	String s1 = new String("abc");
	String s2 = s1;
	String s3 = new String("abc");
	System.out.println("s1 == s2        " + (s1 == s2));
	System.out.println("s1 == s3        " + (s1 == s3));
	System.out.println("s2 == s3        " + (s2 == s3));


	s1 = "abc";
	System.out.println("s1 == s2        " + (s1 == s2));
	System.out.println("s1 == s3        " + (s1 == s3));
	System.out.println("s2 == s3        " + (s2 == s3));				

}
```
我们先看看这样的一个例子，会输出什么：
```java
s1 == s2        true
s1 == s3        false
s2 == s3        false
s1 == s2        false
s1 == s3        false
s2 == s3        false
```
执行第一行代码： 在堆里分配空间存放String对象，在常量区开辟空间存放常量“abc”，String对象指向常量，s1指向该对象。   
执行第二行代码：s2指向上一步new出来的string对象。   
执行第三行代码： 在堆里分配新的空间存放String对象，新对象指向常量“abc”，s3指向该对象。   
到这里，很明显，s1和s2指向的是同一个对象    

接着就很诡异了，我们让s1 依旧= “abc”,但是结果s1和s2指向的地址不同了。   
由于String类是不可改变的，所以String对象也是不可改变的，我们每次给String赋值都相当于执行了一次new String()，然后让变量指向这个新对象，而不是在原来的对象上修改。


我们查看源码可知：
```java
public final class String {}
```
所以可以很清楚的知道，fianl的话是改变不了的，所以，如果我们用String来操作字符串的时候，一旦我们字符串的值改变，就会在内存创建多一个空间来保存新的字符串   

StringBuffer和StringBuilder都集成了`AbstractStringBuilder`，而StringBuffer大部分方法都是synchronized，也就是线程安全的，而StringBuilder就没有，所以，我们查看API可以知道，StringBuilder可以操作StringBuffer，但是StringBuffer不可以操作StringBuilder，这也是线程的原因；


## 扩展
 
 [你真的了解String类的intern()方法吗](http://blog.csdn.net/seu_calvin/article/details/52291082)
