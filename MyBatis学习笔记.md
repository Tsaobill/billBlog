#MyBatis学习
##简介
####什么是 MyBatis？
1. MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。
2. MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。
3. MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

<!-- more -->

####现有的持久化技术对比
1）JDBC
	SQL和JAVA代码在一起，耦合度高。
	难以维护
2）Hibernate和JPA
	长难复杂SQL，不容易处理
	自动生成的SQL，不容易做特殊化处理
	基于全映射的全自动框架，难以对大量字段的POJO进行部分映射，导致数据库性能下降。
3）MyBatis
	SQL和JAVA代码分离，业务和数据的功能边界清晰
	核心SQL需自己优化

##使用MyBatis
####在项目中引入MyBatis
以IDEA为例，在SpringBoot项目中引入MyBatis
步骤 ：
1.在pom文件中引入maven依赖
2.在properties文件中配置mybatis
3.创建mapper.xml映射文件,并进行两个绑定：映射文件和Mapper接口通过namespace绑定，select等语句的id和Mapper接口中的方法名进行绑定。
4.创建Mapper接口

## 方法参数的提取

![image-20200601161339703](/Users/bill/Library/Application Support/typora-user-images/image-20200601161339703.png)

## XML映射文件

####XML 映射器
MyBatis 的真正强大在于它的语句映射，这是它的魔力所在。由于它的异常强大，映射器的 XML 文件就显得相对简单。如果拿它跟具有相同功能的 JDBC 代码进行对比，你会立即发现省掉了将近 95% 的代码。MyBatis 致力于减少使用成本，让用户能更专注于 SQL 代码。

- SQL 映射文件只有很少的几个顶级元素（按照应被定义的顺序列出）：

- cache – 该命名空间的缓存配置。
- cache-ref – 引用其它命名空间的缓存配置。
- resultMap – 描述如何从数据库结果集中加载对象，是最复杂也是最强大的元素。
- parameterMap – 老式风格的参数映射。此元素已被废弃，并可能在将来被移除！请使用行内参数映射。文档中不会介绍此元素。
- sql – 可被其它语句引用的可重用语句块。
- insert – 映射插入语句。
- update – 映射更新语句。
- delete – 映射删除语句。
- select – 映射查询语句。


下一部分将从语句本身开始来描述每个元素的细节。







![image-20200604105715198](/Users/bill/Library/Application Support/typora-user-images/image-20200604105715198.png)

### 标签描述

1、较顶级元素的标签

```xml
<select id=" "></select>
映射查询语句
属性：
id：唯一标识，应与mapper命名空间中对应的Mapper接口中对应的方法名保持一致
parameterType：对应方法的参数类型
resultType：对应方法的返回类型。
resultMap：单独定义返回结果的映射关系，与resultType二者同时只能使用一个。
完整的属性：
<select
  id="selectPerson"
  parameterType="int"
  parameterMap="deprecated"
  resultType="hashmap"
  resultMap="personResultMap"
  flushCache="false"
  useCache="true"
  timeout="10"
  fetchSize="256"
  statementType="PREPARED"
  resultSetType="FORWARD_ONLY"/>

<delete id="del"></delete>
映射删除语句
id：
parameterType：

<insert id="insert"></insert>
映射插入语句
id：
parameterType：对应方法的参数类型


<update id="update"></update>
映射更新语句
id：唯一标识
parameterType：

仅仅适用于update和insert的两个属性：
useGeneratedKeys：
使用 JDBC 的 getGeneratedKeys 方法来取出由数据库内部生成的主键（比如：像 MySQL 和 SQL Server 这样的关系型数据库管理系统的自动递增字段），默认值：false。
keyProperty：
指定能够唯一识别对象的属性，MyBatis 会使用 getGeneratedKeys 的返回值或 insert 语句的 selectKey 子元素设置它的值，默认值：未设置（unset）。如果生成列不止一个，可以用逗号分隔多个属性名称。


<sql id="sql">
用于包裹可被其他语句引用的可重用语句块
  <sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
  可以：
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns">
      	<property name="alias" value="t1"/>
  	</include>,
    <include refid="userColumns">
      	<property name="alias" value="t2"/>
  	</include>
  from some_table t1
    cross join some_table t2
</select>
```

2、结果映射resultMap

是用来对复杂查询结果和Java类进行自高级映射的标签

```xml
<resultMap id="detailedBlogResultMap" type="Blog">
  <constructor>
    <idArg column="blog_id" javaType="int"/>
  </constructor>
  <result property="title" column="blog_title"/>
  <association property="author" javaType="Author">
    <id property="id" column="author_id"/>
    <result property="username" column="author_username"/>
  </association>
  <collection property="posts" ofType="Post">
    <id property="id" column="post_id"/>
    <result property="subject" column="post_subject"/>
    <association property="author" javaType="Author"/>
    
    <collection property="tags" ofType="Tag" >
      <id property="id" column="tag_id"/>
    </collection>
    
    <discriminator javaType="int" column="draft"/>
    
  </collection>
</resultMap>
下级标签：
1. <constructor/> - 用于在实例化类时，将结果传递到构造方法中
		idArg - ID 参数；标记出作为 ID 的结果可以帮助提高整体性能
		arg - 将被注入到构造方法的一个普通结果
2. <id/> – 一个 主键和类属性的映射。 可以帮助提高整体性能
3. <result/> – 注入到字段或 JavaBean 属性的普通结果
4. <association/> – 一个复杂类型的关联；许多结果将包装成这种类型
			嵌套结果映射 – 关联可以是 resultMap 元素，或是对其它结果映射的引用
5. <collection/> – 一个复杂类型的集合
			嵌套结果映射 – 集合可以是 resultMap 元素，或是对其它结果映射的引用
6. <discriminator/> – 使用结果值来决定使用哪个 resultMap
			case – 基于某些值的结果映射
					嵌套结果映射 – case 也是一个结果映射，因此具有相同的结构和元素；或者引用其它的结果映射
下级标签的属性：
	column：查询结果的列名
	property：映射类的属性名


resultMap属性：
id	当前命名空间中的一个唯一标识，用于标识一个结果映射。
type 想要将结果映射到的类的完全限定名
autoMapping	如果设置这个属性，MyBatis 将会为本结果映射开启或者关闭自动映射。 能够覆盖全局的属性设置autoMappingBehavior。默认为：未设置（unset）。
```

