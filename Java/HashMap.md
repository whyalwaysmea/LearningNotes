## 继承关系
![HashMap继承结构](http://img.blog.csdn.net/20160313235311248)  
HashMap继承了AbstractMap，并且支持序列化和反序列化。由于实现了Clonable接口，也就支持clone()方法来复制一个对象。   
另外，HashMap是一个非线程安全的，因此适合运用在单线程环境下。如果是在多线程环境，可以通过Collections的静态方法synchronizedMap获得线程安全的HashMap，如下代码所示。
```java
Map<String, String> map = Collections.synchronizedMap(new HashMap<String, String>());
```

## 先了解哈希
Hash（哈希），又称“散列”。    

在某种程度上，散列是与排序相反的一种操作，排序是将集合中的元素按照某种方式比如字典顺序排列在一起，而散列通过计算哈希值，打破元素之间原有的关系，使集合中的元素按照散列函数的分类进行排列。  

我们通常使用数组或者链表来存储元素，一旦存储的内容数量特别多，需要占用很大的空间，而且在查找某个元素是否存在的过程中，数组和链表都需要挨个循环比较，而通过 哈希 计算，可以大大减少比较次数。

### 哈希冲突
选用哈希函数计算哈希值时，可能不同的 key 会得到相同的结果，一个地址怎么存放多个数据呢？这就是冲突。

常用的主要有两种方法解决冲突：   
1. 链接法（拉链法）。有点像字典的做法，A-Z，头字母相同，代表他们哈希冲突，然后在A中有一个链表，再填充具体的单词。
2. 开放定址法：当冲突发生时，使用某种探查(亦称探测)技术在散列表中寻找下一个空的散列地址，只要散列表足够大，空的散列地址总能找到。

## 内部结构
我们都知道HashMap的存储是需要键值对来进行存放的。那么在HashMap的内部，这样的键值对是以什么形式存在的呢？   
针对每个键值对，HashMap使用内部类`Entry`来存储
```java
static class Entry<K, V> implements Map.Entry<K, V> {
    final K key;
    V value;
    Entry<K, V> next;
    final int hash;

    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }
}
```
Entry类里包含了hashcode变量，key,value 和另外一个Entry对象。为什么要有一个Entry对象呢？我们看一下HashMap的存储结构，就清楚为什么了
![HashMap的存储结构](https://imgsa.baidu.com/baike/c0%3Dbaike80%2C5%2C5%2C80%2C26/sign=f197f18d087b020818c437b303b099b6/91ef76c6a7efce1b028711e2ad51f3deb48f656e.jpg)  
从整体上看，HashMap底层的存储结构是基于数组和链表实现的。对于每一个要存入HashMap的键值对（Key-Value Pair），通过计算Key的hash值来决定存入哪个数组单元（bucket），为了处理hash冲突，每个数组单元实际上是一条Entry单链表的头结点，其后引申出一条单链表。

## 相关属性
在学习HashMap的真正实现之前，我们先了解一下跟它相关的一些属性
```java
// 默认初始容量：16，必须是 2 的整数次方
static final int DEFAULT_INITIAL_CAPACITY = 16;
// 最大容量： 2^ 30 次方
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认加载因子的大小：0.75，可不是随便的，结合时间和空间效率考虑得到的
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 哈希表中的链表数组
transient Entry[] table;
// HashMap键值对的数量。
transient int size;
// 阈值，下次需要扩容时的值，等于 容量*加载因子
int threshold;
// 哈希表的加载因子
final float loadFactor;
```  
需要进一步解释的是loadFactor属性，loadFactor描述了HashMap发生扩容时的填充程度。如果loadFactor设置过大，意味着在HashMap扩容前发生hash冲突的机会越大，因此单链表的长度也就会越长，那么在执行查找操作时，会由于单链表长度过长导致查找的效率降低。如果loadFactor设置过小，那么HashMap的空间利用率会降低，导致HashMap在很多空间都没有被利用的情况下便开始扩容。

## 内部实现
我们先写一段HashMap常用的存取用法
```java
// 初始化HashMap
HashMap<String,String> hashMap = new　HashMap<String,String>();
// 存储数据
hashMap.put("one","hello1");
hashMap.put("two","hello2");
hashMap.put("three","hello3");
hashMap.put("four","hello4");
hashMap.put("five","hello5");
hashMap.put("six","hello6");
hashMap.put("seven","hello7");

// 取数据
String str = hashMap.get("one");

// 遍历
for (Entry<String, String> entry : hashMap.entrySet()) {
    entry.getKey();
    entry.getValue();
}
```
### 构造方法
我们先看看它的构造方法
```java
//创建一个空的哈希表，指定容量和加载因子
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and the default load factor (0.75).
 *
 * @param  initialCapacity the initial capacity.
 * @throws IllegalArgumentException if the initial capacity is negative.
 */
//创建一个空的哈希表，指定容量，使用默认的加载因子
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 */
// 创建一个空的哈希表，初始容量为 16，加载因子为 0.75
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/**
 * Constructs a new <tt>HashMap</tt> with the same mappings as the
 * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
 * default load factor (0.75) and an initial capacity sufficient to
 * hold the mappings in the specified <tt>Map</tt>.
 *
 * @param   m the map whose mappings are to be placed in this map
 * @throws  NullPointerException if the specified map is null
 */
//创建一个内容为参数 m 的内容的哈希表
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```
经过初始化之后，数组table还是一个空数组(排除第四种构造函数的情况)。主要是设置了一些相关属性，加载因子和初始化容量。

### put
```java
//添加指定的键值对到 Map 中，如果已经存在，就替换
public V put(K key, V value) {
     //先调用 hash() 方法计算位置
    return putVal(hash(key), key, value, false, true);
}
/**
 * Implements Map.put and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果table数组为空，新建。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //如果要插入的位置没有元素，新建个节点并放进去    
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        //如果要插入的数组已经有元素，替换
        // e 指向被替换的元素
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            //p 指向要插入的数组第一个 元素的位置，如果 p 的哈希值、键、值和要添加的一样，就停止找，e 指向 p
            e = p;
        else if (p instanceof TreeNode)
         //如果不一样，而且当前采用的还是 JDK 8 以后的树形节点，调用 putTreeVal 插入
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //否则还是从传统的链表数组查找、替换
            //遍历这个桶所有的元素
            for (int binCount = 0; ; ++binCount) {
                //没有更多了，就把要添加的元素插到后面得了
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //当这个桶内链表个数大于等于 8，就要树形化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //如果找到要替换的节点，就停止，此时 e 已经指向要被替换的节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
         //存在要替换的节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            //替换，返回
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //如果超出阈值，就得扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
根据代码可以总结插入逻辑如下:

1. 先调用 hash() 方法计算哈希值
2. 然后调用 putVal() 方法中根据哈希值进行相关操作
3. 如果当前 哈希表内容为空，新建一个哈希表
4. 如果要插入的数组中没有元素，新建个节点并放进去
5. 否则从数组中第一个元素开始查找哈希值对应位置
   1. 如果数组中第一个元素的哈希值和要添加的一样，替换，结束查找
   2. 如果第一个元素不一样，而且当前采用的还是 JDK 8 以后的树形节点，调用 putTreeVal() 进行插入
   3. 否则还是从传统的链表数组中查找、替换，结束查找
   4. 当这个数组内链表个数大于等于 8，就要调用 treeifyBin() 方法进行树形化
6. 最后检查是否需要扩容

插入过程中涉及到几个其他关键的方法 ：

* hash():计算对应的位置
* resize():扩容
* putTreeVal():树形节点的插入
* treeifyBin():树形化容器

这里涉及到了一些JDK1.8之后的优化，所以暂时就不具体分析了。

我们看看这里的这个hash()算法
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
通过将传入键的 hashCode 进行无符号右移 16 位，然后进行按位异或，得到这个键的哈希值。


## 相关链接
[Java 集合深入理解（16）：HashMap 主要特点和关键方法源码解读](http://blog.csdn.net/u011240877/article/details/53351188)   

[Java你可能不知道的事(3)HashMap](http://blog.csdn.net/yissan/article/details/50888070)

[HashMap源码分析](http://tinylcy.me/2016/12/04/HashMap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
