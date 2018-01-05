## 什么是ORM 
ORM(Object/Relationship Mapping)：对象/关系映射   

## 为什么需要ORM
利用面向对象思想编写的数据库应用程序最终都是把对象信息保存在关系型数据库中，于是要编写很多和底层数据库相关的SQL语句。   

写SQL语句有什么不好吗？ 
1. 不同的数据库使用的SQL语法不同。比如：PL/SQL  
2. 同样的功能在不同的数据库中有不同的实现方式。 比如分页SQL 
3. 程序过分依赖SQL对程序的移植及扩展，维护带来很大的麻烦

## 什么是Hibernate  
Hibernate是Java领域的一款开源的ORM框架技术   
Hibernate对JDBC进行了非常轻量级的对象封装  

## 其他主流的ORM框架技术
1. MyBatis: 前身就是著名的iBatis
2. Toplink: 后被Oracle收购，并重新包装为Oracle AS TopLink
3. EJB: 本身是JavaEE的规范

## Hibernate初探 
使用Hibernate需要有以下4步：  
1. 创建Hibernate的配置文件hibernate.cfg.xml
2. 创建持久化类
3. 创建对象-关系映射文件
4. 通过Hibernate API编写访问数据库的代码  

## 编写第一个Hibernate例子  
### 创建hibernate的配置文件hibernate.cfg.xml 
```xml
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
    <session-factory>    	
    	<!-- 数据库帐号 -->
        <property name="connection.username">root</property>
        <!-- 数据库密码 -->
        <property name="connection.password">root</property>
        <!-- 数据库连接驱动 -->
        <property name="connection.driver_class">com.mysql.jdbc.Driver</property>
        <property name="connection.url">jdbc:mysql:///test?useUnicode=true&amp;characterEncoding=UTF-8</property>
        <!-- 方言 Hibernate可针对特定的数据库进行优化 -->
        <property name="dialect">org.hibernate.dialect.MySQLDialect</property>
		
		<!-- 输出SQL到控制台 -->
        <property name="show_sql">true</property>
        <!-- 输出SQL到控制台后是否进行排版 -->
        <property name="format_sql">true</property>
        <!-- 
        	create：每次生成新的表结构 -> 启动的时候先drop，再create
			create-drop: 先创建，再删除 
			update: 在原有的表的基础之上进行更新 -> 这个操作启动的时候会去检查schema是否一致，如果不一致会做scheme更新
			validate: 启动时验证现有schema与你配置的hibernate是否一致，如果不一致就抛出异常，并不做更新
         -->
        <property name="hbm2ddl.auto">create</property>
        
        <!-- 会给所有的表名前面加上前缀。如hibernate.STUDENTS 
        	<property name="default_schema">hibernate</property>
		-->
		
		<!-- 映射文件 -->
        <mapping resource="Students.hbm.xml"></mapping>
    </session-factory>
</hibernate-configuration>
```  
### 创建持久化类 
```java
public class Students {

    // 学号
	private int sid; 
    // 姓名
	private String sname; 
    // 性别
	private String gender; 
    // 出生日期
	private Date birthday; 
    // 地址
	private String address; 

	// 省略get/set方法

}
```
持久化类要遵循JavaBean的规范： 
1. JavaBean 必须申明为 public class 即：必须是公有的类
2. JavaBean 的所有属性必须申明为 private 即：属性必须私有
3. 通过 setter 方法和 getter 方法设值和取值
4. 必须有一个公有无参构造方法 

### 创建关系映射文件  
```xml
<!-- Students.hbm.xml -->
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
"http://hibernate.sourceforge.net/hibernate-mapping-3.0.dtd">
<hibernate-mapping>
    <class name="com.whyalwaysmea.hibernate.Students" table="STUDENTS">
        <id name="sid" type="int">
            <column name="SID" />
            <generator class="assigned" />
        </id>
        <property name="sname" type="java.lang.String">
            <column name="SNAME" />
        </property>
        <property name="gender" type="java.lang.String">
            <column name="GENDER" />
        </property>
        <property name="birthday" type="java.util.Date">
            <column name="BIRTHDAY" />
        </property>
        <property name="address" type="java.lang.String">
            <column name="ADDRESS" />
        </property>
    </class>
</hibernate-mapping>
```
这里的映射文件名和hibernate.cfg.xml中mapping需要对应

### 通过Hibernate API编写访问数据库代码 
```java
public class StudentsTest {

    private SessionFactory sessionFactory;
    private Session session;
    private Transaction transaction;//事务对象

    @Before
    public void init() {
        //创建配置对象
        Configuration config = new Configuration().configure();
        //创建服务注册对象
        ServiceRegistry serviceRegistry = new ServiceRegistryBuilder().applySettings(config.getProperties()).buildServiceRegistry();
        //创建会话工厂
        sessionFactory = config.buildSessionFactory(serviceRegistry);
        //创建会话对象
        session = sessionFactory.openSession();
        //开启事务
        transaction = session.beginTransaction();
    }

    @After
    public void destory() {
        transaction.commit(); //提交事务
        session.close(); //关闭会话
        sessionFactory.close(); //关闭会话工厂
    }

    @Test
    public void testSaveStudents() {
        Students s = new Students(1,"whyalwaysmea","boy",new Date(),"成都");
        session.save(s); //保存对象进入数据库
    }
}
``` 

## hibernate元素介绍  
### hibernate的执行流程 
1. 创建配置对象Configuration，配置对象的作用就是读取配置文档Hibernate.cfg.xml  
2. 获得Configuration的目的是创建SessionFactory对象。创建SessionFactory时就会读取相应的里面所加载的对象关系映射文件。  
3. 获得了SessionFactory对象后就可创建Session对象。Session对象类似于JDBC中的Connection。获得了Session对象，就表示获得了数据库连接对象。就可以执行Session中相应的方法，如：save、delecte、update、get。
4. 在执行Session的方法时，必须要开启事务。这些方法都要封装在事务当中。
5. 执行完方法后，要先提交事务，再关闭session。 

### session简介  
hibernate是对jdbc的封装，不建议直接使用jdbc的connection操作数据库，而是通过使用session操作数据库。

所以，session可以理解为操作数据库的对象。我们要使用Hibernate操作数据库之前，就必须获得一个session的实例。

session与connection，是多对一关系，每个session都有一个与之对应的connection，一个connection不同时刻可以提供多个session使用。

把对象保存在关系数据库中需要调用session的各种方法，如：save()，update()，delete()，createQuery()等。

如何获取session对象？  
1. openSession
2. getCurrentSession  
如果使用getCurrentSession需要在hibernate.cfg.xml文件中进行配置： 
如果是本地事务（jdbc事务） 
```xml
<property name="hibernate.current_session_context_class">thread</property> 
```
如何是全局事务(jta事务)  
```xml
<property name="hibernate.current_session_context_class">jta</property> 
```  
[全局事务和本地事务](http://blog.csdn.net/zhang_xiaomeng/article/details/55105700)    

openSession与getCurrentSession的区别:  
1. getCurrentSession在事务提交或者回滚之后会自动关闭，而openSession需要手动关闭。如果使用openSession而没有手动关闭，多次提交之后会导致连接池溢出。  
2. openSession每次创建新的session对象，getCurrentSession使用现有的session对象。 

### transaction简介    
hibernate对数据的操作都是封装在事务当中，并且默认是非自动提交的方式。所以用session保存对象时，如果不开启事务，并且手工提交事务，对象并不会真正保存在数据库中。  

如果想让hibernate像jdbc那样自动提交事务，必须调用session对象的doWork()方法，获得jdbc的connection后，设置其为自动提交事务模式，但通常并不推荐这样做。
```java
@Test
public void testSaveStudents() {
    Students s = new Students(2,"whyalwaysmea","boy",new Date(),"四川");
    session.doWork(new Work() {

        @Override
        public void execute(Connection connection) throws SQLException {
            connection.setAutoCommit(true);

        }

    });
    session.save(s); // 保存对象进入数据库
    session.flush(); // 必须调用该方法，将数据推送进数据库
}
``` 

## Hibernate单表操作  
### 单一主键  
单一主键是指表中由某一列充当主键。 
两种常见的生成但已逐渐策略：

1. assigned 由java应用程序负责生成（手工赋值）。
2. native 由底层数据库自动生成标识符，如果是MySQL就是increment，如果是Oracle就是sequence。 

与单一主键对应的还有联合主键 

### 组件属性  
实体类中的某个属性属于用户自定义的类的对象。  

在我们的Students类中有address属性，它是属于Address类的对象。而Address类又有三个字段。那么久把address这个属性称为组件属性。   
```java
// 改造一下Students.java
public class Students {

	private int sid; // 学号
	private String sname; // 姓名
	private String gender; // 性别
	private Date birthday; // 出生日期
	private Address address; // 地址

    // 省略get/set
}
```
新建Address.java
```java
public class Address {

	private String postcode;
	private String phone;
	private String address;

    // 省略get/set
}
```
改造Students.hbm.xml 
```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
"http://hibernate.sourceforge.net/hibernate-mapping-3.0.dtd">
<!-- Generated 2018-1-5 10:34:28 by Hibernate Tools 3.5.0.Final -->
<hibernate-mapping>
    <class name="com.whyalwaysmea.hibernate.Students" table="STUDENTS">
        <id name="sid" type="int">
            <column name="SID" />
            <generator class="assigned" />
        </id>
        <property name="sname" type="java.lang.String">
            <column name="SNAME" />
        </property>
        <property name="gender" type="java.lang.String">
            <column name="GENDER" />
        </property>
        <property name="birthday" type="java.util.Date">
            <column name="BIRTHDAY" />
        </property>
        <component name="address" class="com.whyalwaysmea.hibernate.Address">
             <property name="postcode"  column="POSTCODE" />
             <property name="phone"  column="PHONE" />
             <property name="address"  column="ADDRESS" />
        </component>
    </class>
</hibernate-mapping>
```
进行测试:
```java
public class StudentsTest {

	private SessionFactory sessionFactory;
	private Session session;
	private Transaction transaction;// 事务对象

	@Before
	public void init() {
		// 创建配置对象
		Configuration config = new Configuration().configure();
		// 创建服务注册对象
		ServiceRegistry serviceRegistry = new ServiceRegistryBuilder().applySettings(config.getProperties())
				.buildServiceRegistry();
		System.out.println("aaa");
		// 创建会话工厂
		sessionFactory = config.buildSessionFactory(serviceRegistry);
		System.out.println("ddd");
		// 创建会话对象
		session = sessionFactory.openSession();
		System.out.println("hhh");
		// 开启事务
		transaction = session.beginTransaction();
	}

	@Test
	public void testSaveStudents() {
		// 生成学生对象
		Address address = new Address("中国", "四川", "成都");
		Students s = new Students(3, "张三丰", "男", new Date(), address);
		// 保存对象进入数据库
		session.save(s);
	}

	@After
	public void destory() {
		transaction.commit(); // 提交事务
		session.close(); // 关闭会话
		sessionFactory.close(); // 关闭会话工厂
	}
}
```

### 单表CRUD 
- save
- update
- delete
- get/load （查询单个记录）  

get与load的区别：    
1. 在不考虑缓存的情况下，get方法会在调用之后立即向数据库发出sql语句，返回持久化对象。load方法会在调用后返回一个代理对象。该代理对象只保存了实体对象的id，直到使用对象的非主键属性时才会发出sql语句。
2. 查询数据库中不存在数据时，get方法返回null，load方法抛出异常org.hibernate.ObjectNotFoundException

## 参考 
[Hibernate初探之单表映射](http://blog.csdn.net/ljxljxljx747/article/details/77450841#comments)   
[慕课网-Hibernate初探之单表映射](https://www.imooc.com/learn/396) 