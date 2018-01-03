# 什么是ORM 
ORM(Object/Relationship Mapping)：对象/关系映射   

# 为什么需要ORM
利用面向对象思想编写的数据库应用程序最终都是把对象信息保存在关系型数据库中，于是要编写很多和底层数据库相关的SQL语句。   

写SQL语句有什么不好吗？ 
1. 不同的数据库使用的SQL语法不同。比如：PL/SQL  
2. 同样的功能在不同的数据库中有不同的实现方式。 比如分页SQL 
3. 程序过分依赖SQL对程序的移植及扩展，维护带来很大的麻烦

# 什么是Hibernate  
Hibernate是Java领域的一款开源的ORM框架技术   
Hibernate对JDBC进行了非常轻量级的对象封装  

# 其他主流的ORM框架技术
1. MyBatis: 前身就是著名的iBatis
2. Toplink: 后被Oracle收购，并重新包装为Oracle AS TopLink
3. EJB: 本身是JavaEE的规范

