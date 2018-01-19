# 关系映射级别注解 
## 一对一单向外键 
- @OneToOne(cascade=CascadeType.ALL)  
- @JoinColumn(name="pid", unique=true)   

@OneToOne(cascade=)
表示级联关系  
@JoinColumn(name="pid", unique=true)     
表示外键关联， 对应关联类的列名

这里我们以学生和身份证号为模型，一个身份证号可以唯一的对应一个学生，一个学生有一个唯一的身份证号   


```java
@Entity
public class IDCard {

	@Id
	@GeneratedValue(generator="pid")
	@GenericGenerator(name="pid", strategy="assigned")
	@Column(length=18)
	private String pid;		// 身份证号码
	private String sname;	// 学生的姓名
}
```
```java
@Entity
public class Student {

	@Id
	@GeneratedValue
	private int sid;
	private Date birthday;
	
	@OneToOne(cascade=CascadeType.ALL)
	@JoinColumn(name="pid", unique=true)
	private IDCard idCard;
}
```  
因为是Student中有一个指向IDCard表主键的字段pid，所以主控方是Student，所谓主控方就是能改变关联关系的一方，Student只要改变pid就改变了关联关系


## 一对一双向外键 
1. 主控方的配置同一对一单向外键关联 
2. @OneToOne(mappedBy="idCard")   // 被控方 

mappedBy中的属性是，主控方中外键的属性。  
双向关联，必须设置mappedBy属性。因为双向关联只能交给一方去控制，不可能在双方都设置外键保存关联关系，否则双发都无法保存。  

```java
@Entity
public class IDCard {

	@Id
	@GeneratedValue(generator="pid")
	@GenericGenerator(name="pid", strategy="assigned")
	@Column(name = "pid", length=18)
	private String pid;		// 身份证号码
	private String sname;	// 学生的姓名
	
	@OneToOne(mappedBy="idCard")
	private Student student;
}
``` 
Student和上面一样没有任何改变  

## 多对一单向外键  
多方持有一方的引用， 比如：多个学生对应一个班级  

@ManyToOne(cascade={CascadeType.All}, fetch=FetchType.EAGER) 
fetch: 抓取策略。 


@JoinColumn(name="cid", referencedColumnName="id")
name：多方外键的列名 
referencedColumnName：一方主键的列名  

```java
/**
 * 教室，一的一方
 * @author whyalwaysmea
 */
@Entity
public class ClassRoom {

	@Id
	@Column(name="id")
	@GeneratedValue(generator="cid")
	@GenericGenerator(name="cid", strategy="uuid")
	private String classRoomId;
	
	private String classRoomName;

}
```
```java
/**
 * 学生类，多的一方
 * @author whyalwaysmea
 */
@Entity
public class Student {

	@Id
	@GeneratedValue
	private int sid;
	private String sname;
	
	@ManyToOne(cascade= {CascadeType.ALL}, fetch=FetchType.EAGER)
	@JoinColumn(name="cid", referencedColumnName="id")
	private ClassRoom classRoom;
}
```

## 一对多单向外键  
一方持有多方的集合，一个班级有多个学生   

@OneToMany(cascade={CascadeType.All}, fetch=FetchType.LAZY)
@JoinColumn(name="cid")


## 多对多单向外键 
学生和教师构成多对多的关联关系 
其中一个多方持有另一个多方的集合对象（学生持有教室的集合）  
创建中间表  

```java
@Entity
public class Teacher {

	@Id
	@GeneratedValue(generator="tid")
	@GenericGenerator(name="tid", strategy="uuid")
	private String tid;
	
	private String tname;
}
```
```java
@Entity
public class Student {

	@Id
	@GeneratedValue(generator="sid")
	@GenericGenerator(name="sid", strategy="uuid")
	private String sid;
	
	private String sname;
	
	@ManyToMany
	@JoinTable(name = "teachers_students", 
			inverseJoinColumns = {@JoinColumn(name = "teacherId", referencedColumnName="tid")},
					joinColumns = {@JoinColumn(name = "studentId", referencedColumnName="sid")} 
	)
	private Set<Teacher> teachers = new HashSet<>();
}
```

# 参考
[代码下载](https://gitee.com/whyalwaysmea/hibernate-learn)  
[Hibernate注解](https://www.imooc.com/learn/524)