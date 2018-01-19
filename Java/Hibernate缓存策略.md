# 了解缓存 
## 什么是缓存
缓存是指为了降低应用程序对物理数据源访问的频次，从而提高应用程序的运行性能的一种策略。 

## 为什么使用缓存
1. ORM框架访问数据库的效率直接影响应用程序的运行速度，提升和优化ORM框架的执行效率至关重要  
2. Hibernate的缓存是提升和优化Hibernate执行效率的重要手段，所以学会Hibernate缓存的使用和配置是优化的关键 

# Hibernate一级缓存 
## 介绍Hibernate一级缓存
1. Hibernate一级缓存又称为“Session缓存”、“会话级缓存”
2. 通过Session从数据库查询实体时会把实体在内存中存储起来，下一次查询同一实体时不再从数据库获取，而从内存中获取
3. 一级缓存的生命周期和Session相同；Session销毁，它也销毁
4. 一级缓存中的数据可适用范围在当前会话之内

## Hibernate一级缓存的API
一级缓存无法取消，用两个方法管理 
`evict()`:用于将某个对象从Session的一级缓存中清楚   
`clear()`:用于将一级缓存中的所有对象全部清楚   

## 代码测试 
### 同一个Session进行重复查询
```java
@Test
public void oneLevelTest() {
    // 使用同一个Session进行重复查询
    Student student = (Student) session.get(Student.class, 1);
    System.out.println(student.getSname());
    
    Student student2 = (Student) session.get(Student.class, 1);
    System.out.println(student2.getSname());
}
```  
这里我们先观察控制台的输出:
```sql
Hibernate: select student0_.SID as SID1_1_0_, student0_.SNAME as SNAME2_1_0_, student0_.sex as sex3_1_0_, student0_.gid as gid4_1_0_ from student student0_ where student0_.SID=?;
Rose
Rose
```
可以看到，在使用同一个Session进行重复查询的时候，只进行了一次SQL查询，就可以打印出两次。可以说明第二次查询是使用了缓存的  

### 不同Session进行重复查询
```java
@Test
public void oneLevelTest2() {
    // 使用不同Session进行重复查询	
    Session session = HibernateUtil.getSession();
    Student student = (Student) session.get(Student.class, 1);
    System.out.println(student.getSname());
    HibernateUtil.closeSession(session);
    
    Session session2 = HibernateUtil.getSession();
    Student student2 = (Student) session2.get(Student.class, 1);
    System.out.println(student2.getSname());
    HibernateUtil.closeSession(session2);		
}
```
```sql
Hibernate: select student0_.SID as SID1_1_0_, student0_.SNAME as SNAME2_1_0_, student0_.sex as sex3_1_0_, student0_.gid as gid4_1_0_ from student student0_ where student0_.SID=?
Rose
Hibernate: select student0_.SID as SID1_1_0_, student0_.SNAME as SNAME2_1_0_, student0_.sex as sex3_1_0_, student0_.gid as gid4_1_0_ from student student0_ where student0_.SID=?
Rose
``` 
这里进行了两次查询，说明第二次的查询没有使用到缓存。这正是因为两次查询使用的不是同一个Session

### query.list() 和 query.iterate()
在这里分别使用query.list() 和 query.iterate()进行查询，然后再观察控制台的输出。
#### 单独使用query.list()
```java
@Test
public void oneLevelTest3() {
    // 使用query.list()进行查询	
    Session session = HibernateUtil.getSession();
    Query query = session.createQuery("FROM Student");
    List<Student> list = query.list();
    for (Student student : list) {
        System.out.println(student.getSname());
    }
    HibernateUtil.closeSession(session);	
}
```
```sql
Hibernate: select student0_.SID as SID1_1_, student0_.SNAME as SNAME2_1_, student0_.sex as sex3_1_, student0_.gid as gid4_1_ from student student0_
Rose
Jack
```
可以看到，这里只进行了一次查询就查询出了所有的Student。 

#### 单独使用query.iterate()
接下来看看query.iterate()进行查询
```java
@Test
public void oneLevelTest4() {
    // 使用query.iterate()进行查询	
    Session session = HibernateUtil.getSession();
    Query query = session.createQuery("FROM Student");
    Iterator iterate = query.iterate();
    while(iterate.hasNext()) {
        Student student = (Student) iterate.next();
        System.out.println(student.getSname());
    }
    HibernateUtil.closeSession(session);
}
```
```sql
Hibernate: select student0_.SID as col_0_0_ from student student0_
Hibernate: select student0_.SID as SID1_1_0_, student0_.SNAME as SNAME2_1_0_, student0_.sex as sex3_1_0_, student0_.gid as gid4_1_0_ from student student0_ where student0_.SID=?
Rose
Hibernate: select student0_.SID as SID1_1_0_, student0_.SNAME as SNAME2_1_0_, student0_.sex as sex3_1_0_, student0_.gid as gid4_1_0_ from student student0_ where student0_.SID=?
Jack
``` 
这里输出了三条语句。第一条是查询出所有学生的id，然后在遍历输出学生信息的时候再根据id去查询具体的学生信息。这里因为表中只有两个学生，所以这里就只做了两次单独的查询 

#### 先使用query.list()再使用query.iterate() 
```java
@Test
public void oneLevelTest5() {
    // 先使用query.list()再使用query.iterate()
    Session session = HibernateUtil.getSession();
    Query query = session.createQuery("FROM Student");
    List<Student> list = query.list();
    for (Student student : list) {
        System.out.println(student.getSname());
    }
    System.out.println("-------------");
    Iterator iterate = query.iterate();
    while(iterate.hasNext()) {
        Student student = (Student) iterate.next();
        System.out.println(student.getSname());
    }
    HibernateUtil.closeSession(session);	
}
```
```sql
Hibernate: select student0_.SID as SID1_1_, student0_.SNAME as SNAME2_1_, student0_.sex as sex3_1_, student0_.gid as gid4_1_ from student student0_
Rose
Jack
-------------
Hibernate: select student0_.SID as col_0_0_ from student student0_
Rose
Jack
```
这里一共输出了两行sql。第一行是采用query.list()进行查询时输出的，它查询了所有的Student信息。 第二条是query.iterate()进行查询时输出的，可以看到这条语句仅查询出了所有Student的id，然后便输出了学生的信息。 因为可以推断出，这是的查询是使用了缓存的。 

#### 先使用query.iterate()再使用query.list()
```java
@Test
public void oneLevelTest6() {
    // 先使用query.iterate()再使用query.list()
    Session session = HibernateUtil.getSession();
    Query query = session.createQuery("FROM Student");
    Iterator iterate = query.iterate();
    while(iterate.hasNext()) {
        Student student = (Student) iterate.next();
        System.out.println(student.getSname());
    }
    System.out.println("-------------");
    List<Student> list = query.list();
    for (Student student : list) {
        System.out.println(student.getSname());
    }
    HibernateUtil.closeSession(session);	
}
```
```sql
Hibernate: select student0_.SID as col_0_0_ from student student0_
Hibernate: select student0_.SID as SID1_1_0_, student0_.SNAME as SNAME2_1_0_, student0_.sex as sex3_1_0_, student0_.gid as gid4_1_0_ from student student0_ where student0_.SID=?
Rose
Hibernate: select student0_.SID as SID1_1_0_, student0_.SNAME as SNAME2_1_0_, student0_.sex as sex3_1_0_, student0_.gid as gid4_1_0_ from student student0_ where student0_.SID=?
Jack
-------------
Hibernate: select student0_.SID as SID1_1_, student0_.SNAME as SNAME2_1_, student0_.sex as sex3_1_, student0_.gid as gid4_1_ from student student0_
Rose
Jack
```
因为是先使用query.iterate()进行的查询，所以依照之前的分析会进行三次sql的输出。随后使用query.list()进行查询，它还是像原来一样输出了一行sql。所以我们可以推测query.list()进行查询的时候是没有使用缓存的。

#### query.iterate()和query.list()小结 
query.list()和query.iterate()区别： 
1. 返回的类型不同：  
list()返回List；iterate()返回Iterate。
2. 查询策略不同：  
list()直接发送sql语句，查询数据库；
iterate()发送sql语句，从数据库取出id，然后先从缓存中根据id查找对应信息，
有就返回结果，没有就根据id发送sql语句，查询数据库。  
3. 返回对象不同：    
list()返回持久化实体类对象；
iterate()返回代理对象。  
4. 缓存的关系不同：  
list()只缓存，但不使用缓存（查询缓存除外）；
iterate()会使用缓存。  

# Hibernate二级缓存 
## 环境准备  
要使用二级缓存，需要额外的引入几个jar包,和一个配置文件。  
我使用的版本是：hibernate-release-4.2.4.Final，可以去[官网下载](https://sourceforge.net/projects/hibernate/files/hibernate4/4.2.4.Final/)相应的zip文件。里面包含了跟Hibernate相关的所有jar包了。 
EHCache相关jar包：hibernate-4.2.4\hibernate-release-4.2.4.Final\lib\optional\ehcache下的所有包
EHCache配置文件：hibernate-4.2.4\hibernate-release-4.2.4.Final\project\etc下的ehcache.xml文件 

## 映射文件修改  
除了将相应的jar包和配置文件添加进来，还需要在Java实体类所对应的映射文件中添加如下配置:  
```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC 
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
<hibernate-mapping>
    <class name="com.whyalwaysmea.cache.Grade" table="grade">
        <!-- 配置二级缓存信息 -->
    	<cache usage="read-only" region="Grade"/>
        <id name="gid" column="gid" type="java.lang.Integer">
            <generator class="increment"></generator>
        </id>
        <property name="gname" type="java.lang.String">
            <column name="gname" length="20" not-null="true"></column>
        </property>
        <property name="gdesc" type="java.lang.String">
            <column name="gdesc"></column>
        </property>        
        <set name="students" table="student" inverse="true" cascade="all">
            <!-- 指定关联的外键列 -->
            <key column="gid"></key>
            <one-to-many class="com.whyalwaysmea.cache.Student"/>
        </set>
    </class>
</hibernate-mapping>
```
## 测试 
```java
@Test
public void twoLevelCacheTest() {
    // 使用不同Session重复查询		
    Session session = HibernateUtil.getSession();
    Grade grade = (Grade) session.get(Grade.class, 1);
    System.out.println(grade.getGname());
    HibernateUtil.closeSession(session);
    
    Session session2 = HibernateUtil.getSession();
    Grade grade2 = (Grade) session2.get(Grade.class, 1);
    System.out.println(grade2.getGname());
    HibernateUtil.closeSession(session2);
}
```
```sql
Hibernate: select grade0_.gid as gid1_0_0_, grade0_.gname as gname2_0_0_, grade0_.gdesc as gdesc3_0_0_ from grade grade0_ where grade0_.gid=?
Java大神班
Java大神班
```
通过控制台的输出可以发现，现在使用不同的Session进行查询，只会输出一次sql查询语句了。 

## 二级缓存相关配置 
`<cache />`标签的详情介绍：  
- usage: 指定缓存策略，可选的策略包括：transactional, read-write, nonstrict-read-write或read-only 
- region: 指定二级缓存区域名(用于配置指定缓存属性) 
- include: 指定是否缓存延迟加载的对象。all，表示缓存所有对象；non-lazy，表示不缓存延迟加载的对象

## 缓存策略配置 
最开始配置环境的时候，将ehcache.xml放入进来了，现在我们来看看这一个配置文件的使用
```xml
<cache name="Grade"
    maxElementsInMemory="10000"
    eternal="false"
    timeToIdleSeconds="300"
    timeToLiveSeconds="600"
    overflowToDisk="true"
    />
```
在映射文件中，我们指定了二级缓存区域名为"Grade"，在这里我们就是使用该区域名指定它的缓存策略的。 
这里对配置属性进行一下简单的介绍：  
- maxElementsInMemory:  cache 中最多可以存放的元素的数量  
- eternal : 是否永驻内存。如果值是true，cache中的元素将一直保存在内存中，不会因为时间超时而丢失，所以在这个值为true的时候，timeToIdleSeconds和timeToLiveSeconds两个属性的值就不起作用了。 
- timeToIdleSeconds:  访问这个cache中元素的最大间隔时间。如果超过这个时间没有访问这个cache中的某个元素，那么这个元素将被从cache中清除。  
- timeToLiveSeconds:  cache中元素的生存时间。意思是从cache中的某个元素从创建到消亡的时间，从创建开始计时，当超过这个时间，这个元素将被从cache中清除。  
- overflowToDisk:  溢出是否写入磁盘。  
 
# 一二级缓存的对比  
|  属性 |一级缓存 | 二级缓存 | 
| ------------- |:-------------:| -----:| 
| 缓存的范围 | 事务范围、每个事务都拥有单独一级缓存 | 应用范围，当前应用内所有事务共享 | 
| 并发访问策略 | 不会出现并发问题 | 必须提供适当的并发访问策略 | 
| 数据过期策略 | 没有数据过期策略 | 缓存对象的最大数目、最长时间、最长空闲时间等 | 
| 缓存的软件实现 | 框架包含 | 第三方提供、可插拔集成 | 
| 物理介质 | 内存 | 内存和硬盘 | 
| 启用方式 | 默认启用、不可关闭 | 默认不启用、选择性开启 | 


# 参考 
[hibernate-release-4.2.4.Final下载](https://sourceforge.net/projects/hibernate/files/hibernate4/4.2.4.Final/)   
[Hibernate缓存策略](https://www.imooc.com/learn/465)