MyBatis是一个支持SQL映射和查询的**持久层框架**。
取代解耦JDBC
兼容各种数据库
与Spring很好集成

# JDBC 缺点
- 硬编码
	- 注册驱动
	- 连接
	- SQL语句
- 操作繁琐
	- 设置参数
	- 封装为结果集并返回

# Mapper代理
MyBatis还是会有硬编码的问题
通过Mapper可以动态代理的方式进行sql执行
```java
List<User>users sqlSession.selectList("test.selectAll"); // 写死了"test.selectAll" 

// Mapper 代理
UserMapperuserMapper sqlSession.getMapper(UserMapper.class);
List<User>users userMapper.selectALl();
```

## 包扫描简化映射文件

```xml
<mappers>
	<!--sql映射-->
	<mapper resource="com/packege0/mapper/UserMapper.xml"/>
	<!--Mapper 代理方式, 可以加载到所有的Mapper文件-->
	<package ame="com.packege0.mapper"/>
</mappers>

```
# 分页
## RowBounds
使用`RowBounds` 对象进行分页，它是针对 `ResultSet` 结果集执行的内存(逻辑)分页
```java
RowBounds rowBounds = new RowBounds((pageNum - 1) * pageSize, pageSize);
List<YourObject> resultList = sqlSession.selectList("mapper.selectSome", null, rowBounds);
```
## limit
在`sql`中加入`limit` 实现物理分页
```sql
SELECT * FROM table_name
LIMIT #{offset}, #{limit}
```

## 分页插件
实现物理分页
MyBatis 分页可通过 MyBatis-PageHelper 插件实现。首先，在项目中添加 MyBatis-PageHelper 依赖，然后使用 RowBounds 对象传递 offset 和 limit 参数来实现分页查询。例如：

```java
PageHelper.startPage(pageNum, pageSize);
List<YourEntity> result = yourMapper.selectYourEntities();
```

# 查询
## 封装
如果数据库的字段名和类的属性名不一样 -> 不自动封装
> [!info] 解决方案
> 1. 写别名
> 2. 写映射
### 别名
```xml
<select id="selectAll"resultType="brand">
	select id,brand_name as brandName,company_name as companyName
	from tb_brand;
</select>
```

缺点-> 还是**写死的** 
### resultMap映射
```xml
<resultMap id="brandResultMap" type="brand">
	<!--
		id: DB主键
		property: 是实体类属性名
	-->
	<result column="brand_name" property="brandName"/>
	<result column="company_name" property="companyName"/>
</resultMap>

<select id="selectALl" resultMap="brandResultMap">
	select *
	from tb_brand;
</select>
```

## 参数
```xml
<select id="selectById" resultMap="brandResultMap">
	select *
	from tb_brand where id =#{id};
</select>
```

```java
Brand brand brandMapper.selectById(id);
System.out.println(brand);
```

参数占位符的两种方式
```xml
#{id}  <!--会将其替换为`?`占位符，为了防止SQL注入-->
${id}  <!--直接拼在sql上。会存在SQL注入问题-->
```

## 条件查询
```java
List<Brand>selectByCondition(@Param("status") int status, @Param("companyName") String
companyName, @Param("brandName") String brandName);
List<Brand>selectByCondition(Brand brand);
List<Brand>selectByCondition(Map map);
```

函数参数`@Param("companyName")`需要和`xml`的参数名一致

```xml
<select id="selectByCondition"resultMap="brandResultMap">
	select
	from tb brand
	where
	status =#{status}
	and company_name like #{companyName}
	and brand name like #{brandName}
</select>
```

# 面试题

5. MyBatis 是什么以及它的主要功能？
6. MyBatis 与 Hibernate 的区别？
10. MyBatis 如何处理关联查询和一对多关系？
11. 如何使用 MyBatis 进行事务管理？
12. MyBatis 缓存是什么，它的作用是什么？
13. MyBatis 插件的作用及如何实现？
14. 如何解决 MyBatis 的参数传入问题？
15. 

使用 MyBatis 进行事务管理的方法包括：
1. 使用 Spring 框架的声明式事务管理，将 MyBatis 与 Spring 集成。
2. 在 MyBatis 的映射文件中使用 `<transaction>` 标签指定事务管理器。
3. 使用 MyBatis 的 `SqlSession` 接口手动管理事务，调用 `commit()` 和 `rollback()` 方法。



java开发一个商城项目,你对sql进行优化,对用户浏览商品的行为做了商品数据库的添加了多个索引,你会怎么设计这个索引?


针对用户浏览商品的行为，可以设计以下索引：
1. 对商品分类进行索引，加快根据分类筛选商品的速度。
2. 对商品价格进行索引，以便快速根据价格区间查找商品。
3. 对商品名称进行全文搜索索引，提高搜索速度和准确性。
4. 对商品库存和销量进行索引，以便快速筛选出有货或热销商品。
5. 如果涉及地域信息，对商品所在地进行索引，加快根据地区筛选商品的速度。

