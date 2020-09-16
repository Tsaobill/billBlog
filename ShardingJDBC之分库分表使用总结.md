## 摘要

实习入职第二周被要求实现一个Spring工程，其中一个要求是接入分库分表的数据源。因此开始学习。经过一番百度和Google，对分库分表的概念有了大致的了解，并最终选择使用sharding-jdbc来实现。



## 内容

### 1. 介绍

Sharding-JDBC最开始是从当当网内部架构ddframe中一个分库分表的模块脱离而出的，用来解决当当的分库分表问题，把相关的业务代码剥离后，就有了Sharding-JDBC，成为一个工作在客户端的分库分表解决方案。

2018 年 5 月， 因为增加了 Proxy 的版本和 Sharding-Sidecar（尚未发布）， Sharding-JDBC 更名为 Sharding Sphere，从一个客户端的组件变成了一个套件,版本变为3.x。

2018 年 11 月， Sharding-Sphere 正式进入 Apache 基金会孵化器， 这也是对 Sharding-Sphere 的质量和影响力的认可，这里有个时间点就是，3之前的版本groupID为io.shardingjdbc，3.X的版本为io.shardingsphere。

目前，ShardingSphere在maven仓库中的groupID已经变成org.apache.shardingshpere。版本4.X。

<img src="/Users/bill/Library/Application Support/typora-user-images/image-20200624133125489.png" alt="image-20200624133125489" style="zoom:50%;" />

Sharding-JDBC的更名和捐献前后groupID都不一样，在引入依赖时请注意区分。虽然各版本主题功能相同，但是某些接口和类的实现有所改变，因此更换版本后，import需要修改，类和接口的实现也需要对应修改。



### 使用

#### 1.在pom文件中引入依赖

```xml
<!-- Sharding JDBC进行分库分表-->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-core</artifactId>
    <version>4.1.0</version>
</dependency>
<!-- 使用spring命名空间进行配置的话，还需要此依赖，其官方文档有问题，artifactId错误的写成shardingsphere-jdbc-spring-namespace-->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-spring-namespace</artifactId>
    <version>4.1.0</version>
</dependency>
```

配置数据源和分库分表策略，编写对应的算法类以实现策略。

Sharding-JDBC提供了4中配置方式，详情看官网https://shardingsphere.apache.org/document/legacy/4.x/document/cn/manual/sharding-jdbc/configuration/

2.1 Java 配置

第一种是把数据源和分片策略都写在 Java Config 中，它的特点是非常灵活，我们可 以实现各种定义的分片策略。但是缺点是，如果把数据源、策略都配置在 Java Config 中，就出现了硬编码，在修改的时候比较麻烦。

 2.2 Spring Boot 配置

第二种是直接使用 Spring Boot 的 application.properties 来配置，这个要基于 starter 模块，org.apache.shardingsphere 的包还没有 starter，只有 io.shardingsphere 的包有 starter。把数据源和分库分表策略都配置在 properties 文件中。这种方式配置比较简单，但 是不能实现复杂的分片策略，不够灵活。

 2.3 yml 配置

第三种是使用 Spring Boot 的 yml 配置(shardingjdbc.yml)，也要依赖 starter 模块。当然我们也可以结合不同的配置方式，比如把分片策略放在 JavaConfig 中，数据 源配置在 yml 中或 properties 中。

 2.4 spring命名空间配置，以下为示例。

1、配置数据源：

Shardingdatasource.properties

```properties
sj_0.driver=com.mysql.jdbc.Driver
sj_0.url=jdbc:mysql://localhost:3306/users01?useUnicode=true&amp;characterEncoding=UTF-8
sj_0.username=root
sj_0.password=123456

sj_1.driver=com.mysql.jdbc.Driver
sj_1.url=jdbc:mysql://localhost:3306/users02?useUnicode=true&amp;characterEncoding=UTF-8
sj_1.username=root
sj_1.password=123456
```

2、单独编写一个分片配置文件：spring-sharding.xml。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:sharding="http://shardingsphere.apache.org/schema/shardingsphere/sharding"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://shardingsphere.apache.org/schema/shardingsphere/sharding
       http://shardingsphere.apache.org/schema/shardingsphere/sharding/sharding.xsd">

      <!--配置分库分表数据源-->
    <bean id="sj_ds_0" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <property name="driverClass" value="${sj_0.driver}"/>
        <property name="jdbcUrl" value="${sj_0.url}"/>
        <property name="user" value="${sj_0.username}"/>
        <property name="password" value="${sj_0.password}"/>
    </bean>
    <bean id="sj_ds_1" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <property name="driverClass" value="${sj_1.driver}"/>
        <property name="jdbcUrl" value="${sj_1.url}"/>
        <property name="user" value="${sj_1.username}"/>
        <property name="password" value="${sj_1.password}"/>
    </bean>
  
    <sharding:standard-strategy id="tableStrategy" sharding-column="id"
                                precise-algorithm-ref="tableAlgo"/>

    <sharding:standard-strategy id="databaseStrategy" sharding-column="id"
                                precise-algorithm-ref="databaseAlgo"/>
    <sharding:data-source id="shardingDataSource">
        <sharding:sharding-rule data-source-names="sj_ds_0,sj_ds_1">
            <sharding:table-rules>
                <sharding:table-rule logic-table="users"
                                     actual-data-nodes="sj_ds_$->{0..1}.user_$->{0..2}"
                                     database-strategy-ref="databaseStrategy"
                                     table-strategy-ref="tableStrategy" key-generator-ref="keyGenerate"
                />
            </sharding:table-rules>
        </sharding:sharding-rule>
    </sharding:data-source>

    <sharding:key-generator id="keyGenerate" column="id" type="SNOWFLAKE"/>
</beans>
```

3、还需要将实现的分片策略算法注入到spring容器中。

```xml
<bean id="tableAlgo" class="com.warush.service.shardingAlgo.TableShardingAlgorithm"/>
<bean id="databaseAlgo" class="com.warush.service.shardingAlgo.DatabaseShardingAlgorithm"/>
```

