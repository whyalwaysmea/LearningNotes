## 前言
主要分析Just和From两个操作符的区别。以及尝试分析一下源码。  
基于[RxJava2.0.9](https://github.com/ReactiveX/RxJava)

## [Just](http://reactivex.io/documentation/operators/just.html)
创建一个发射指定值的Observable  
![Just](http://reactivex.io/documentation/operators/images/just.c.png)

Just将单个数据转换为发射那个数据的Observable。
```Java
Integer[] items = {0, 1, 2, 3, 4, 5};
Observable.just(items, items)
        .subscribe(new Consumer<Integer[]>() {
            @Override
            public void accept(@NonNull Integer[] integers) throws Exception {
                Stream.of(integers)
                        .forEach(integer -> System.out.println(integer));
            }
        });
```


## [From](http://reactivex.io/documentation/operators/from.html)
将其它种类的对象和数据类型转换为Observable
![Frome](http://reactivex.io/documentation/operators/images/from.c.png)  
当你使用Observable时，如果你要处理的数据都可以转换成展现为Observables，而不是需要混合使用Observables和其它类型的数据，会非常方便。这让你在数据流的整个生命周期中，可以使用一组统一的操作符来管理它们。

例如，Iterable可以看成是同步的Observable；Future，可以看成是总是只发射单个数据的Observable。通过显式地将那些数据转换为Observables，你可以像使用Observable一样与它们交互。

在RxJava中，from操作符可以转换Future、Iterable和数组。对于Iterable和数组，产生的Observable会发射Iterable或数组的每一项数据。

在RxJava1中，`from`操作符就可以接收Future、Iterable和数组作为参数。  
在RxJava2中，不再有单独的`from`操作符了，而是变为了`fromFuture`，`fromIterable`，`fromArray`
```Java
Integer[] items = { 0, 1, 2, 3, 4, 5 };
Observable.fromArray(items)
          .subscribe(System.out::println);
```
```Java
ArrayList<String> arrayList = new ArrayList<String>() {
    {
        add("str01");
        add("str02");
    }
};
Observable.fromIterable(arrayList)
        .subscribe(System.out::println);
```

## 区别
Just类似于From，但是From会将数组或Iterable的数据取出然后逐个发射，而Just只是简单的原样发射，将数组或Iterable当做单个数据。
