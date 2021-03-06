# 数据类型和变量 
数据类型用于对数据归类，以便于理解和操作。 
- 整数类型：byte/short/int/long， 分别有不同的取值范围 
- 小数类型：float/double，有不同的取值范围和精度
- 字符类型：char，表示单个字符
- 真假类型：boolean，表示真假

基本数据类型都有对应的数组类型，数组表示固定长度的同种数据类型的多条记录，这些数据在内存中连续存放。比如，一个自然数可以用一个整数类型数据表示，100个连续的自然数可以用一个长度为100的整数数组表示。  

为了操作数据，需要把数据存放到内存中。所谓内存在程序看来就是一块有地址编号的连续的空间，数据放到内存中的某个位置后，为了方便地找到和操作这个数据，需要给这个位置起一个名字。编程语言通过**变量**这个概念来表示这个过程。
声明一个变量，比如int a，其实就是在内存中分配了一块空间，这块空间存放int数据类型，a指向这块内存空间所在的位置，通过对a操作即可操作a指向的内存空间。 

# 赋值 
## 基本类型 
| 类型名| 取值范围 | 字节 |
| ------------- |:-------------:| :-------------:| 
| byte | -2^7 ~ 2^(7-1) |  1 |
| short| -2^15 ~ 2^(15-1)| 2 |
| int | -2^31 ~ 2^(31-1)| 4 |
| long| -2^63 ~ 2^(63-1)| 8 |


| 类型名| 取值范围 | 字节 | 有效数字位数 |
| ------------- |:-------------:| :-------------:| :-------------:| 
| float| 1.40E-45 <-> 3.40E+38 && -3.40E+38 <-> -1.4E-45 |  4 | 8位|
| double| 4.90E-324 <-> 1.7E+308 && -1.7E+308 <-> -4.9E-324 | 8 | 16位 | 

1.4E-45表示：1.4乘以10的045次方。
如何理解有效数字位数？
```java
double d = 123456789.123456789;
System.out.println(d);
 
float f = 123456789.123456789f;
System.out.println(f);
```
对应的输出是：
>1.2345678912345679E8
1.23456792E8

一般来说，CPU处理单精度浮点数的速度比处理双精度浮点数快

如果不声明，默认小数为double类型，所以如果要用float的话，必须进行强转
例如：float  a=1.3; 会编译报错，正确的写法 float a = (float)1.3;或者float a = 1.3f;（f或F都可以不区分大小写）

## 数组类型 
```java
// 预先知道数组的内容
int[] arr = {1,2,3};
int[] arr = new int[]{1,2,3};
// 先分配长度，然后再给每个元素赋值
int[] arr = new int[3];
arr[0] = 1;  arr[1] = 2;
``` 
即使没有给每个元素赋值，每个元素也都有一个默认值：数值类型的值为0，boolean为false，char为空字符。 

数组类型和基本类型有明显不同的，一个基本类型变量，内存中只会有一块对应的内存空间。但数组有两块：一块用于存储数组内容本身，另一块用于存储内容的位置。
![变量对应的内存地址和内容](http://img.blog.csdn.net/20180223105503605?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzQzNTg5Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 

基本类型a的内存地址是1000，这个位置存储的就是它的值100. 
数组类型arr的内存地址是2000，这个位置存储的值是一个位置3000，3000开始的位置存储的才是实际的数据“1，2，3”。

**为什么数组要用两块空间？** 
```java
int[] arrA = {1,2,3};
int[] arrB = {4,5,6,7};
arrA = arrB;
```
这个代码中，arrA初始的长度是3，arrB的长度是4，后来将arrB的值赋给了arrA。如果arrA对应的内存空间是直接存储的数组内容，那么它将没有足够的空间去容纳arrB的所有元素。 
用两块空间存储就简单得多，arrA存储的值就变成了和arrB的一样，存储的都是数组内容{4,5,6,7}的地址，此后访问arrA就和arrB一样的了，而arrA{1,2,3}的内存空间由于不再被引用会进行垃圾回收。 

# 基本运算 
## 算术运算 
算术运算符有加、减、乘、除，取模，自增，自减。  
取模运算适用于整数和字符类型，其他算术运算适用于所有数值类型和字符类型。

**注意：** 
1. 运算时要注意结果的范围，使用恰当的数据类型
```java
// 2147483647是int能表示的最大值
int a = 2147483647 * 2; 
```
a的结果是-2。为了避免这种情况，我们的结果类型应使用long，但是只改为long也是不够的，因为运算还是默认按照int类型进行，需要将至少一个数据表示为long：
```java
long a = 2147483647 * 2L;
```

2.整数相除不是四舍五入，而是直接舍去小数位 
```java
double d = 10 / 4;
```
结果是2。 如果要按小数进行运算，需要将至少一个数表示为小数形式，或者使用强制类型转换
```java
double d = 10 / 4.0;
double d = 10 / (double)4;
```

3.小数计算结果不精确
```java
float f = 0.1f * 0.1f;
```
理论上结果应该是：0.01。 但实际上输出是0.010000001。 
如果换成double，结果也不精确。
**为什么会这样？**，需要理解float和double的二进制表示。 

# 条件执行 

## switch
switch的表达式值的数据类型只能是：byte、short、int、char、枚举和String（Java7以后）

## 实现原理 
程序最终都是一条条的指令，CPU有一个指令指示器，指向下一条要执行的指令，CPU根据指示器的指示加载指令并且执行。  
但有一些特殊的指令，称为跳转指令，这些指令会修改指令指示器的值，让CPU跳到一个指定的地方执行。跳转有两种：一种是条件跳转；另一种是无条件跳转。

if/else实际上会转换为这些跳转指令 
```java
int a = 10;
if(a%2 == 0) {
	System.out.println("偶数");
}
// 其他代码
```
转换到转移指令可能是：
```java
1 int a = 10;
2 条件跳转：如果a%2==0，跳转到第4行
3 无条件跳转：跳转到第5行
4 System.out.println("偶数");
5 // 其他代码
```
可能会奇怪，为什么需要第3行的无条件跳转。因为指令是顺序执行下来的，如果没有它，那么括号中的输出语句就会执行。
当然，对应的跳转指令也可能是：
```java
1 int a = 10;
2 条件跳转：如果a%2!=0，跳转到第4行
3 System.out.println("偶数");
4 // 其他代码
```
这里就没有无条件跳转指令，具体怎么对应和编译器实现有关。  

switch的转换和具体系统实现有关。如果分支比较少，可能会转换为跳转指令。如果分支比较多，使用条件跳转会进行很多次的比较运算，效率比较低，可能会使用一种更为高效的方式，叫跳转表。  
跳转表是一个映射表，存储了可能的值以及要跳转到的地址

跳转表为什么会更高效呢？因为其中的值必须为整数，且按大小顺序排序。按大小排序的整数可以使用高效的二分查找。如果值是连续的，则跳转表还会进行特殊优化，优化为一个数组，连找都不用找了，值就是数组的下标索引。
之所以switch值的类型可以是byte、short、int、char、枚举和String。其中byte/short/int本来就是整数，char本质上也是整数，而枚举类型也有对应的整数，String用于switch时也会转换为整数。  

**不可以使用long是为什么呢？** 跳转表值的存储空间一般为32位，容纳不下long。

# 函数调用的基本原理 
## 栈 
之前谈过程序执行的基本原理：CPU有一个指令指示器，指向下一条要执行的指令，要么顺序执行，要么进行跳转（条件跳转或无条件跳转）。

程序从main函数开始顺序执行，函数调用可以看作一个无条件跳转，跳转到对应函数的指令处开始执行，碰到return语句或者函数结尾的时候，再执行一次无条件跳转，跳转回调用方，执行调用函数后的下一条指令。

但是这里面有几个问题： 
1. 参数如何传递？
2. 函数如何知道返回到什么地方？
3. 函数结果如何传给调用方？

解决思路是使用内存来存放这些数据，函数调用方和函数自己就如何存放和使用这些数据达成一个一致的协议，这个协议在各种计算机系统中都是类似的，存放这些数据的内存有一个相同的名字，叫做**栈** 

栈是一块内存，一般是先进后出。栈一般是从高位地址向低位地址扩展，换句话说，栈底的内存地址是最高的，栈顶的是最低的。 

## 函数执行的基本原理
我们直接看一个简单的例子：
```java
1 public class Sum {
2 
3   public static int sum(int a, int b) {
4       int c = a + b;
5       return c;
6   }    
7
8   public static void main(String[] args) {
9       int d = Sum.sum(1,2);
10      System.out.println(d);
11  }
12 }
``` 
main函数调用了sum函数，计算1和2的和，然后输出计算结果。  

我们从栈的角度来讨论下：  
1.当程序在main函数调用Sum.sum之前  
![main函数调用Sum.sum之前](http://img.blog.csdn.net/20180227093753388?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzQzNTg5Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 
栈中主要存放了两个变量，args和d 

2.在main函数调用Sum.sum时， 首先将参数1和2入栈，然后将返回地址入栈，接着进入sum函数内部，为局部变量c分配一个空间，而参数变量a和b则直接对应于入栈的数据1和2，在返回之前，返回值保存到了专门的返回值存储器中。
![在Sum.sum内部，准备返回之前的栈示意图](http://img.blog.csdn.net/20180227094347683?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzQzNTg5Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

main的下一条指令就是根据函数返回值给变量d复制，返回值从专门的返回值存储器中获得。  

从以上关于栈的描述可以看出，函数中的参数和函数内定义的变量都分配在栈中，这些变量只有在函数被调用的时候才分配，而且在调用结束后就被释放了。但这个主要针对基本数据类型

## 数组和对象的内存分配
对于数组和对象类型，它们都有两块内存：一块存放实际的内容，一块存放实际内容 的地址。 实际的内容是分配在**堆**上，存放地址的空间是分配在栈上。  
```java
public class ArrayMax {
	public static int max(int min, int[] arr) {
		int max = min;
		for(int a : arr) {
			if(a>max) {
				max = a;
			}
		}
		return max;
	}
	public static void main(String[] args) {
		int[] arr = new int[]{2,3,4};
		int ret = max(0, arr);
		System.out.println(ret);
	}
}
```
main函数新建一个数组，然后调用函数max计算0和数组中元素的最大值。 

在程序执行到max函数的return语句之前的时候:
![max函数的return语句之前](http://img.blog.csdn.net/20180227100628270?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzQzNTg5Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

对于数组arr，在栈中存放的是实际内容的地址，存放地址的栈空间会随着入栈分配，出栈释放。  
栈空间没有变量指向堆空间的时候，Java系统会进行辣鸡回收，进而释放这块堆空间
