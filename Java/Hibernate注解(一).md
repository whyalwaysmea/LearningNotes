# 类级别注解
## @Entity注解 
@Entity: 映射实体类  
@Entity(name = "tableName")  
name:可选，对应数据库中的一个表。若表名与实体类名相同，则可以省略。  

注意： 使用@Entity时必须指定实体类的主键属性，该注解只能使用在类上面。  

下面我们通过代码来测试一下效果。我们通过使用注解直接创建数据库的表数据。

首先创建一个Java Bean，并用@Entity注解和@Id注解 
```java
@Entity(name = "t_student")
public class Student {
	@Id
	private Integer id;
	private String name;
	private String sex;	
	private Date birthday;
    // 省略setter/getter
}
```
接下来需要进行配置文件的配置了，因为使用了注解，所以就不再需要映射文件配置了。
```xml
<!-- hibernate.cfg.xml -->
<!-- 配置映射的实体类 -->
<mapping class="com.whyalwaysmea.annotation.Student"/>
```  
这样进行简单的配置就可以测试了
```java
@Test
public void init() {
    // 创建配置对象
    Configuration config = new Configuration().configure();
    // 创建服务注册对象
    ServiceRegistry serviceRegistry = new ServiceRegistryBuilder().applySettings(config.getProperties())
            .buildServiceRegistry();
    // 创建会话工厂
    sessionFactory = config.buildSessionFactory(serviceRegistry);
    // 创建会话对象
    session = sessionFactory.openSession();
    // 开启事务
    transaction = session.beginTransaction();
    
}
```  
我们这里的测试代码只是简单的进行相关属性的配置，运行之后会发现如下的输出：
```sql
Hibernate: 
    drop table if exists t_student
Hibernate: 
    create table t_student (
        id integer not null,
        birthday datetime,
        name varchar(255),
        sex varchar(255),
        primary key (id)
    )
``` 
通过sql语句的输出，可以发现`@Entity(name = "t_student")`中的name属性发挥作用了，生成的表名就是t_student。如果不配置该参数，那么生成的表名应该是student。

## @Table 
@Table(name = "", catalog = "", schema = "")
- name: 可选，映射表的名称，默认表名和实体名称一致，只有在不一致的情况下才需要指定表名。 
- catalog: 可选，表示Catalog名称，默认为Catalog("")。
- schema: 可选，表示Schema名称，默认为Schema("")。 

## @Embeddable
@Embeddable表示一个非Entity类可以嵌入到另一个Entity类中作为属性而存在。  
这里还以Student类为基础，我们额外增加一个Address.java表示地址， 它是Student的一个属性 
```java
@Embeddable
public class Address {
	private String postCode;
	private String address;
	private String phone;
    // 省略setter/getter
}
```
```java
@Entity
@Table(name = "t_student")
public class Student {
	...
    private Address add;
    // 省略setter/getter
}
```  
测试用例依然和刚才一样，这里就直接输出控制台的语句了： 
```sql
Hibernate: 
    drop table if exists t_student
Hibernate: 
    create table t_student (
        id integer not null,
        address varchar(255),
        phone varchar(255),
        postCode varchar(255),
        birthday datetime,
        name varchar(255),
        sex varchar(255),
        primary key (id)
    )
``` 
这里把Address里面的属性都建立进入了t_student表中  

# 属性级别注解  
添加方式：  
1. 写在属性字段上面  
2. 写在属性的get方法上面  

## @Id  
@Id: 是实体类必须的。 定义了映射到数据库表的主键的属性，一个实体类可以有一个或者多个属性被映射为主键，可置于主键属性或者getXxx()前。  

**注意：** 如果有多个属性定义为主键属性，该实体类必须实现serializalbe接口。

## @GeneratedValue 
@GeneratedValue(strategy=GenerationType, generator=""): 可选， 用于定义主键生成策略。  
strategy表示主键生成策略，取值有： 
1. GenerationType.AUTO: 根据底层数据库自动选择(默认)
2. GenerationType.INDENTITY: 根据数据库的Identity字段生成 
3. GenerationType.SEQUENCE: 使用Sequence来决定主键的取值 
4. GenerationType.TABLE: 使用指定表来决定主键取值，结合@TableGenerator使用   

generator: 表示主键生成器的名称，这个属性通常和ORM框架相关  
例如：Hibernate可以指定uuid等主键生成方式  

```java
@Entity
@Table(name = "t_student")
public class Student {

	@Id
	@GeneratedValue(generator = "uuid")
	@GenericGenerator(name = "uuid", strategy="uuid")
	private String id;
	
	private String name;

}
``` 

## @GenericGenerator
@GenericGenerator注解是hibernate所提供的自定义主键生成策略生成器，由@GenericGenerator实现多定义的策略。所以，它要配合@GeneratedValue一起使用，并且@GeneratedValue注解中的”generator”属性要与@GenericGenerator注解中name属性一致，strategy属性表示hibernate的主键生成策略.  
这里列出常用的几种生成策略:  
1. assigned:  手工分配主键ID值。该策略要求程序员必须自己维护和管理主键，当有数据需要存储时，程序员必须自己为该数据分配指定一个主键ID值，如果该数据没有被分配主键ID值或分配的值存在重复，则该数据都将无法被持久化且会引起异常的抛出。       
2. uuid：32位16进制数的字符串。采用128位UUID算法生成主键，能够保证网络环境下的主键唯一性，也就能够保证在不同数据库及不同服务器下主键的唯一性。uuid 最终被编码成一个32位16进制数的字符串，占用的存储空间较大。用于为 String 类型生成唯一标识，适用于所有关系型数据库。 
3. identity：自然递增。 支持 DB2，MySQL，SQL Server，Sybase 和HypersonicSQL 数据库， 用于为 long 或 short 或 int 类型生成唯一标识。它依赖于底层不同的数据库，与 Hibernate 和 程序员无关。
4. increment：自然递增。与 identity 策略不同的是，该策略不依赖于底层数据库，而依赖于 hibernate 本身，用于为 long 或 short 或 int 类型生成唯一标识。主键计数器是由 hibernate 的一个实例来维护，每次自增量为 1，但在集群下不能使用该策略，否则将引起主键冲突的情况，该策略适用于所有关系型数据库使用。  


## @Column 
@Column：可将属性映射到列，使用该注解来覆盖默认值，@Column描述了数据库表中该字段的详细定义，这对于根据JPA注解生成数据库表结构的工具非常有作用。 
常用属性：  
name: 可选，表示数据库表中该字段的名称，默认情形属性名字一致  
nullable: 可选，表示该字段是否允许为null，默认为true  
unique: 可选，表示该字段是否是唯一标识，默认为false  
length: 可选，表示该字段的大小，仅对String类型的字段有效，默认值255.
insertable: 可选，表示在ORM框架执行插入操作时，该字段是否应出现在INSERT语句中，默认为true  
updateable: 可选，表示在ORM框架执行更新操作时，该字段是否应该出现在update语句中，默认为true。对于一经创建就不可以更改的字段，该属性非常有用  

## @Embedded 
@Embedded 是注释属性的，表示该属性的类是嵌入类  

**注意：** 同时嵌入类也必须标注@Embeddable注解  

## @EmbeddedId  
@EmbeddedId  使用嵌入式主键类实现复合主键  

**注意：** 嵌入式主键类必须实现Serializable接口、必须有默认的public无参数构造方法，必须覆盖equals和hashCode方法  

```java
/**
 * 嵌入式联合主键
 */
@Embeddable
public class ManPK implements Serializable {

	@Column(length=18)
	private String id;	// 身份证号
	@Column(length=8)
	private String sId;	// 用户编号

    // 省略getter/setter equals hashCode方法 
}
```
```java
@Entity
@Table(name = "t_man")
public class Man {

	@EmbeddedId
	private ManPK manPK;
	
	private String userName;
	
	private Integer age;
}
```
```java
@Test
public void idTest() {
    ManPK manPK = new ManPK();
    manPK.setId("321564561321");
    manPK.setsId("abcdefg");
    
    Man man = new Man();
    man.setManPK(manPK);
    man.setAge(21);
    man.setUserName("善良的空间");
    
    session.save(man);
}
```

## @Transient  
可选，表示该属性并非一个到数据库表的字段的映射，ORM框架将忽略该属性 