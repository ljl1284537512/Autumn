## MyBatis应用分析与最佳实践(怎么用，如何用好？)

### 目录
- 为什么用MyBatis?
- 不用容器，MyBatis怎么用？
- MyBatis的高级用法与扩展
- MyBatis常见问题

### 核心主题
- 特性
- 编程式开发方法和核心对象
- 核心配置含义
- 高级用法与扩展方式

#### 为什么要使用MyBatis?（原生API的缺点）
1. 代码重复(注册驱动，获取连接等)
2. 结果处理集太复杂
3. 数据库操作（语句）与业务耦合在一起
4. 连接管理（手动管理资源）

#### Spring jdbc & 工具类(DbUtils)

##### 解决了什么问题?
1. 方法封装
如Spring jdbc的JdbcTemplate
2. 支持数据源
如Spring jdbc的dataSource
3. 映射结果集
如Spring jdbc的RowMapper.mapRow()方法

##### 没有解决什么问题？
1. SQL语句硬编码
2. 参数只能按照顺序传入(占位符)
3. 没有实现实体类到数据库记录的映射
如没有提供save(T t)来完成实体与记录之间的映射
5. 没有提供缓存等功能

### ORM介绍

O: 对象
R: 关系型数据库
M: 映射

ORM要完成的事情就是将对象与数据库进行映射(实体属性<->表字段，对象<->数据库记录)，操作对象就和操作数据库记录是一样的。

#### Hibernate

##### 使用方式
hbm.xml或者注解 + session.save(T)/update/get

##### 问题
1. 不能指定部分字段
使用save/get等操作，都是操作表里面的所有字段，不能指定其中某个字段。
3. 无法自定义SQL,优化比较困难
Hibernate自动生成sql语句，无法对其进行优化。
5. 不支持动态SQL
比如要动态变更表名（分表的情况），Hibernate就不支持了。

#### MyBatis
##### 特性
1. 使用连接池对连接进行管理
2. SQL和代码分离，集中管理
SQL都在配置文件中
4. 参数映射和动态SQL
5. 结果集映射
6. 缓存管理
7. 重复SQL的提取
重复的sql可以提取出"sql"的标签，可以进行复用
9. 插件机制

#### 如何选择ORM框架
1. 业务简单的项目可以使用Hiberante
不需要怎么优化，涉及的sql操作非常简单。
3. 需要灵活的SQL，可以用MyBatis
4. 对性能要求高，可以使用JDBC
涉及的表特别少，但是很要求性能
5. Spring JDBC 可以和ORM框架混用

### MyBatis编程式开发

#### 使用步骤
1. 引入Mybatis以及Mysql依赖。
2. 增加全局配置文件-mybatis-config.xml。
3. 创建SqlSession会话，通过SqlSessionFactory生成。
```java
SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder().build(inputStream<mybatis--config>);
SqlSession session = sessionFactory.openSession();
```
4. 使用拿到的session完成相关的数据库操作。
```java
PeopleMapper mapper = session.getMapper(PeopleMapper.class);
mapper.selectAll();
```
#### 核心对象
1. SqlSessionFactoryBuilder
用处：创建工厂类
生命周期：方法局部(method)
3. SqlSessionFactory
用处：
生命周期：全局生命周期(application,保持单例)
4. SqlSession
用处: 会话，用于操作数据库
生命周期：应用的每一次请求中存在(request/method)，请求关闭了之后，就应该关闭session。
5. Mapper
生命周期：因为它是由session生成的，所以它的生命周期和session是一致的(method)。//TODO

#### 核心配置

##### 全局配置文件(mybatis-config.xml)

configuration：对应Mybatis中的org.apache.ibatis.session.Configuration，其属性对应configuration标签下的二级标签。
properties：引用一些属性；
settings：Mybatis全局行为的一个控制；
tryAliases：定义用于简化拼写的别名；
typeHandlers：java数据类型与db数据类型的转换(String<->varchar)
> mybatis如何将BLOB类型转换成java对象? 自定义TypeHandler；包括如何实现将Java的Json对象转换成Mysql中的Json类型的字段；
> TypeHandler也可以ResultMap中进行映射，将varchar类型转换成Java的Json对象；

objectFactory: 对象工厂，Mybatis从db里面查出来数据之后，需要将其转换成java对象，首先需要创建对象，如果这个对象需要被定制化，就可以自定义创建工厂。
pulgins: 插件
environments: 环境配置（事务与数据源配置1                                                              ）
mappers: 定义Mapper的映射文件

##### 核心配置-Mapper.xml

cache:
cache-ref:
resultMap:
> resultMap和resultType的区别？

sql:
insert:
delete:
update:
select:

##### 核心配置-settings

cacheEnabled
proxyFactory
lazyLoadingEnabled
aggressiveLazyLoading
lazyLoadTriggerMethods
defaultExecutorType
localCachedScope
logImpl

##### 动态SQL配置

if：条件判断
choose(结合when,otherwise): 条件判断 
trim(放在where或者set里面): 去除匹配的字符串(如结尾的逗号)
foreach: 遍历/循环

##### 批量操作

>代码里面的for循环操作和MyBatis里面的foreach批量操作有什么区别？
>答：如果在代码中循环去做某些事，那么就会频繁的创建和关闭session，浪费性能。而如果使用Mybatis的foreach，则Mybatis会生成一条批量处理的sql语句，避免了频繁的session处理。
>但是如果使用批量操作，拼接的sql语句过长，会导致问题-Mybatis数据包确保不会超过4MB。
>如何解决？使用B
> statement和preparedStatement的区别？

1. 批量插入
2. 批量更新
3. Batch Executor: 会把sql都攒着，最后一次性发送；

#### 嵌套(关联查询) / N+1/ 延迟加载

1. 什么时候会出现关联查询？
答：如果要查询员工的部门信息，会关联部门表。
3. MyBatis关联查询的方式？
答：两种方式进行映射
- 嵌套结果：resultMap标签中提供association标签，定义关联的对象，Mybatis会把查询的结果映射到关联的对象上面去；
- 嵌套查询：同样内嵌assocation标签，使用这个标签提供的select属性，其值为对应查询的方法（当调用该属性的时候，会执行该方法）；
> assocation 和 collection的区别?

5. 什么是N+1问题?
答：当涉及到多表关联查询的时候，每次查询主表的时候(1)，会自动去查询其下对应的属性(N)，这就涉及多个关联子查询。然而这个子查询，也许我们根本就用不到；
7. 延迟加载配置及原理
答：设置setting属性的时候，设置lazyLoadingEnabled=true，这样就能保证关联查询的时候，暂时只查主表，关联表的查询是在获取属性的时候开始执行(对主表查询出来的对象执行equals，clone，hashCode，toString这些方法，会默认触发二次查询)。；
延迟加载还涉及到setting标签的aggressiveLazyLoading属性，如果这个属性设置为true，那么对主查询出来的对象执行任何方法，都会启动关联对象的查询。
疑问：在获取关联对象的时候，Mybatis做了什么？
答：主查询的结果为代理对象，再次调用的时候，就会执行二次查询。
8. Mybatis动态代理配置；
Mybatis可以配置setting标签下的proxyFactory的值来指定动态代理的方式（默认使用Javassist，可以修改成CGLIB）。

#### 翻页

1. 逻辑翻页与物理翻页的区别
逻辑翻页：将所有数据查询出来，在内存中完成翻页;
物理翻页：利用DB的支持，从数据库层面进行翻页（如Mysql的limit支持）；
3. 逻辑翻页的支持RowBounds
RowBounds内置offset和limit属性，查询的时候，可以指定offset和limit的值，并将RowBounds对象作为查询方法的参数就可以完成逻辑分页。
逻辑翻页的缺点: 一次性查询所有数据，如果数据量过大，会严重影响性能。
5. 物理翻页的几种实现方式
- 对Mapper中的SQL语句进行limit拼接（需要在Java代码中计算页数，以及条数）；
- 使用PageHelper等其他插件自动生成物理翻页的sql语句；

#### 通用Mapper的引入
如果字段发生了变化，怎么办？
解决思路：
1). 继承(Mapper.xml也支持继承)，但是继承会导致类和Mapper.xml都急剧增多。
2). 通用Mapper(tk mybatis)；
