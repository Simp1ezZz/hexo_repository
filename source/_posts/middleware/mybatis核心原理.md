---
title: MyBatis核心原理：映射机制、缓存与插件
date: 2023-04-19 12:42:01
tags:
  - 文章
categories: 学习
---


## 前言

MyBatis是Java生态中广泛使用的持久层框架，它封装了JDBC，简化了数据库操作。与Hibernate的全自动ORM不同，MyBatis采用半自动模式，开发者可以编写原生SQL，灵活控制执行效率。本文将深入剖析MyBatis的核心原理，包括映射机制、缓存实现、插件机制以及源码分析。

## MyBatis架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        MyBatis架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  接口层 (SqlSession)                                            │
│  ├─ SqlSessionFactory                                            │
│  ├─ SqlSession                                                   │
│  └─ Mapper接口                                                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  核心处理层                                                       │
│  ├─ Executor (SQL执行器)                                         │
│  │   ├─ SimpleExecutor                                          │
│  │   ├─ ReuseExecutor                                           │
│  │   └─ BatchExecutor                                           │
│  │                                                               │
│  ├─ StatementHandler (SQL语句处理)                               │
│  │   ├─ PreparedStatementHandler                                 │
│  │   ├─ CallableStatementHandler                                │
│  │   └─ SimpleStatementHandler                                  │
│  │                                                               │
│  ├─ ParameterHandler (参数处理)                                  │
│  ├─ ResultSetHandler (结果处理)                                  │
│  └─ TypeHandler (类型转换)                                       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  基础支撑层                                                       │
│  ├─ 数据源 (DataSource)                                          │
│  ├─ 事务管理 (Transaction)                                       │
│  ├─ 缓存 (一级缓存、二级缓存)                                     │
│  ├─ 插件 (Interceptor)                                           │
│  └─ 日志 (Logging)                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 1. 映射机制

### mapper.xml配置

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.mapper.UserMapper">

    <!-- 结果映射 -->
    <resultMap id="UserResultMap" type="User">
        <id property="id" column="id"/>
        <result property="name" column="name"/>
        <result property="email" column="email"/>
        <result property="createTime" column="create_time"
                typeHandler="org.apache.ibatis.type.JdbcTimestampHandler"/>
    </resultMap>

    <!-- 增删改查 -->
    <select id="selectById" resultMap="UserResultMap">
        SELECT id, name, email, create_time
        FROM users
        WHERE id = #{id}
    </select>

    <select id="selectByCondition" resultMap="UserResultMap">
        SELECT id, name, email, create_time
        FROM users
        <where>
            <if test="name != null">
                AND name LIKE #{name}
            </if>
            <if test="email != null">
                AND email = #{email}
            </if>
        </where>
        ORDER BY create_time DESC
        LIMIT #{offset}, #{limit}
    </select>

    <insert id="insert" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (name, email, create_time)
        VALUES (#{name}, #{email}, #{createTime})
    </insert>

    <update id="update">
        UPDATE users
        <set>
            <if test="name != null">name = #{name},</if>
            <if test="email != null">email = #{email},</if>
        </set>
        WHERE id = #{id}
    </update>

    <delete id="deleteById">
        DELETE FROM users WHERE id = #{id}
    </delete>

</mapper>
```

### 动态SQL原理

```java
// MyBatis动态SQL标签
// if, where, set, foreach, choose, when, otherwise, trim

// ${} vs #{} 的区别
// #{} : 使用PreparedStatement，参数替换为?，防止SQL注入
// ${} : 直接拼接字符串，有SQL注入风险

// 源码位置：org.apache.ibatis.mapping.BoundSql
// #{} 最终被替换为?占位符
// ${} 保持原样拼接
```

### 接口与XML的绑定

```java
// Mapper接口
public interface UserMapper {
    User selectById(Long id);
    List<User> selectByCondition(UserQuery query);
    int insert(User user);
}

// 绑定过程：
// 1. 解析mapper.xml，获取SQL信息
// 2. 根据namespace + methodName 找到对应的SQL
// 3. 创建MapperProxy代理对象
// 4. 调用方法时执行SQL

// 源码位置：org.apache.ibatis.binding.MapperProxy
public class MapperProxy<T> implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. 判断是否需要执行SQL
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        }

        // 2. 获取MapperMethod
        final MapperMethod mapperMethod = cachedMapperMethod(method);

        // 3. 执行SQL
        return mapperMethod.execute(sqlSession, args);
    }
}
```

## 2. 执行流程

### SQL执行过程

```java
// SqlSession执行SQL的完整流程

// 1. 获取SqlSession
SqlSession session = sqlSessionFactory.openSession();

// 2. 获取Mapper
UserMapper mapper = session.getMapper(UserMapper.class);

// 3. 调用方法
User user = mapper.selectById(1L);

// 实际执行流程：
/*
  UserMapper.selectById(1)
       │
       ▼
  MapperProxy.invoke()
       │
       ▼
  MapperMethod.execute()
       │
       ▼
  SqlSession.selectOne()
       │
       ▼
  Executor.query()
       │
       ▼
  SimpleExecutor/ReuseExecutor/BatchExecutor
       │
       ▼
  StatementHandler.query()
       │
       ▼
  PreparedStatementHandler
       │
       ▼
  PreparedStatement.execute()
       │
       ▼
  ResultSetHandler.handleResultSets()
*/
```

### StatementHandler

```java
// StatementHandler接口结构

public interface StatementHandler {
    // 获取Statement
    Statement prepare(Connection connection) throws SQLException;

    // 设置参数
    void parameterize(Statement statement) throws SQLException;

    // 执行查询
    <E> List<E> query(Statement statement, ResultHandler resultHandler)
        throws SQLException;

    // 执行更新
    int update(Statement statement) throws SQLException;

    // 执行批量
    List<BatchResult> batch(Statement statement) throws SQLException;
}

// 实现类：
// - PreparedStatementHandler：预处理SQL（常用）
// - SimpleStatementHandler：普通SQL
// - CallableStatementHandler：存储过程
```

### ParameterHandler

```java
// 参数处理器
public interface ParameterHandler {
    // 获取参数
    Object getParameterObject();
    // 设置参数到PreparedStatement
    void setParameters(PreparedStatement ps) throws SQLException;
}

// 内置参数处理器
// - DefaultParameterHandler
// - 注册自定义TypeHandler处理复杂类型

// TypeHandler示例
@MappedTypes(String.class)
public class StringTypeHandler implements TypeHandler<String> {

    @Override
    public void setParameter(PreparedStatement ps, int i,
                            String parameter, JdbcType jdbcType)
                            throws SQLException {
        ps.setString(i, parameter);
    }

    @Override
    public String getResult(ResultSet rs, String columnName)
                           throws SQLException {
        return rs.getString(columnName);
    }
}
```

### ResultSetHandler

```java
// 结果处理器
public interface ResultSetHandler {
    <E> List<E> handleResultSets(Statement statement) throws SQLException;
    <E> Cursor<E> handleCursorResultSets(Statement statement) throws SQLException;
    void handleOutputParameters(CallableStatement cs) throws SQLException;
}

// 处理过程
// 1. 读取ResultSet
// 2. 根据ResultMap进行映射
// 3. 处理嵌套结果（一对多、一对一）
// 4. 自动映射（开启autoMapping）

// 嵌套结果处理
<resultMap id="UserWithOrders" type="User">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <!-- 一对多：嵌套查询 -->
    <collection property="orders"
                ofType="Order"
                select="selectOrdersByUserId"
                column="id"/>
</resultMap>

<!-- 或嵌套结果 -->
<resultMap id="UserWithOrders" type="User">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <collection property="orders"
                ofType="Order"
                resultMap="OrderResultMap"/>
</resultMap>
```

## 3. 缓存机制

### 一级缓存

```java
// 一级缓存：SqlSession级别，默认开启，不可关闭

// 作用域：同一个SqlSession
// 存储： PerpetualCache（HashMap实现）

// 测试代码
SqlSession session = sqlSessionFactory.openSession();
UserMapper mapper = session.getMapper(UserMapper.class);

// 查询两次，只有第一次会执行SQL
User user1 = mapper.selectById(1L);  // 执行SQL，结果存入缓存
User user2 = mapper.selectById(1L);  // 命中缓存，不执行SQL

// 一级缓存失效场景：
// 1. SqlSession.close() 或 commit()
// 2. 执行update/delete/insert
// 3. clearCache()手动清空
session.clearCache();
```

### 二级缓存

```xml
<!-- mapper.xml中开启 -->
<cache
    eviction="LRU"
    flushInterval="60000"
    size="512"
    readOnly="true"/>

<!-- 或使用自定义缓存 -->
<cache type="com.github.pagehelper.PageHelper"/>
```

```java
// 二级缓存：Mapper级别（namespace）
// 需要开启才会生效

// 配置：
Configuration configuration = new Configuration();
configuration.setCacheEnabled(true);  // 全局开启

// mapper.xml中：
<cache/>  // 该mapper开启二级缓存

// 注意：
// 1. 二级缓存存储的是POJO对象，需实现序列化
// 2. 跨SqlSession共享
// 3. 事务提交后才会写入缓存

// 查询流程：
// SqlSession1.selectById(1)  -> 查缓存(未命中) -> 查DB -> 存入一级缓存
// SqlSession1.commit()       -> 一级缓存内容写入二级缓存
// SqlSession2.selectById(1)  -> 查二级缓存(命中) -> 直接返回
```

### 缓存源码

```java
// CachingExecutor - 二级缓存执行器
public class CachingExecutor implements Executor {

    private final Executor delegate;
    private final TransactionalCacheManager tcm = new TransactionalCacheManager();

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameterObject,
                            RowBounds rowBounds, ResultHandler resultHandler,
                            CacheKey key, BoundSql boundSql) throws SQLException {

        // 1. 获取二级缓存
        Cache cache = ms.getCache();
        if (cache != null) {
            // 2. 查询二级缓存
            List<E> cached = cache.getObject(key);
            if (cached != null) {
                return cached;  // 命中缓存
            }
        }

        // 3. 查询数据库
        List<E> list = delegate.query(ms, parameterObject,
                                       rowBounds, resultHandler, key, boundSql);

        // 4. 放入缓存
        if (cache != null) {
            cache.putObject(key, list);
        }

        return list;
    }
}
```

## 4. 插件机制

### 插件接口

```java
// MyBatis插件接口
@Intercepts({
    @Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class}
    )
})
public class MyPlugin implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 拦截逻辑
        // before: 执行业务逻辑前
        StatementHandler sh = (StatementHandler) invocation.getTarget();

        // 获取SQL信息
        BoundSql boundSql = sh.getBoundSql();
        String sql = boundSql.getSql();
        System.out.println("执行的SQL: " + sql);

        // proceed执行原方法
        Object result = invocation.proceed();

        // after: 执行业务逻辑后
        return result;
    }

    @Override
    public Object plugin(Object target) {
        // 生成代理对象
        // 通常使用Plugin.wrap()
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        // 获取配置文件中的参数
        String dbType = properties.getProperty("dbType");
    }
}
```

### 配置插件

```xml
<!-- mybatis-config.xml -->
<plugins>
    <plugin interceptor="com.example.plugin.MyPlugin">
        <property name="dbType" value="mysql"/>
    </plugin>
</plugins>
```

### 插件原理

```java
// 插件链调用
/*
  SqlSession.selectList()
       │
       ▼
  CachingExecutor.query()
       │
       ▼
  SimpleExecutor.query()
       │
       ▼
  PreparedStatementHandler.prepare()
       │
       ▼
  拦截器链.intercept()
       │
       ▼
  Plugin.invoke()
       │
       ▼
  实际Interceptor.intercept()
*/

// 多个插件的执行顺序
// 配置顺序从上到下，执行时按责任链模式
```

### 分页插件原理

```java
// PageHelper插件原理
@Intercepts({
    @Signature(type = StatementHandler.class,
               method = "prepare",
               args = {Connection.class, Integer.class})
})
public class PageInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler sh = (StatementHandler) invocation.getTarget();
        BoundSql boundSql = sh.getBoundSql();

        // 1. 拦截原SQL
        String originalSql = boundSql.getSql();

        // 2. 判断是否需要分页
        // PageHelper使用ThreadLocal传递分页参数
        Page<?> page = Page.getPage();

        if (page != null) {
            // 3. 改写SQL，添加分页
            String pageSql = page.getPageSql(originalSql);

            // 4. 修改BoundSql
            MetaObject metaObject = SystemMetaObject.forObject(boundSql);
            metaObject.setValue("sql", pageSql);
        }

        return invocation.proceed();
    }
}

// 原始SQL：SELECT * FROM users
// 改写后：SELECT * FROM users LIMIT 10, 20
```

## 5. 核心配置

### mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

    <!-- 引入外部配置文件 -->
    <properties resource="db.properties">
        <property name="username" value="root"/>
    </properties>

    <!-- 设置 -->
    <settings>
        <!-- 开启二级缓存 -->
        <setting name="cacheEnabled" value="true"/>
        <!-- 开启懒加载 -->
        <setting name="lazyLoadingEnabled" value="true"/>
        <!--  aggressiveLazyLoad关闭后，按需加载-->
        <setting name="aggressiveLazyLoading" value="false"/>
        <!-- 自动映射 -->
        <setting name="autoMappingBehavior" value="PARTIAL"/>
        <!-- 下划线转驼峰 -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
        <!-- 日志实现 -->
        <setting name="logImpl" value="SLF4J"/>
    </settings>

    <!-- 类型别名 -->
    <typeAliases>
        <package name="com.example.entity"/>
        <!-- 或手动注册 -->
        <typeAlias type="com.example.entity.User" alias="User"/>
    </typeAliases>

    <!-- 类型处理器 -->
    <typeHandlers>
        <package name="com.example.handler"/>
    </typeHandlers>

    <!-- 对象工厂 -->
    <objectFactory type="com.example.factory.MyObjectFactory">
        <property name="name" value="value"/>
    </objectFactory>

    <!-- 插件 -->
    <plugins>
        <plugin interceptor="com.github.pagehelper.PageInterceptor">
            <property name="helperDialect" value="mysql"/>
        </plugin>
    </plugins>

    <!-- 环境配置 -->
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC">
                <property name="..." value="..."/>
            </transactionManager>
            <dataSource type="UNPOOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>

    <!-- Mapper注册 -->
    <mappers>
        <mapper resource="mapper/UserMapper.xml"/>
        <package name="com.example.mapper"/>
    </mappers>

</configuration>
```

### Spring集成配置

```xml
<!-- Spring与MyBatis集成 -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations"
              value="classpath:mapper/**/*.xml"/>
    <property name="configLocation"
              value="classpath:mybatis-config.xml"/>

    <!-- 配置类型别名包 -->
    <property name="typeAliasesPackage"
             value="com.example.entity"/>

    <!-- 配置插件 -->
    <property name="plugins">
        <array>
            <bean class="com.github.pagehelper.PageInterceptor"/>
        </array>
    </property>
</bean>

<!-- Mapper扫描 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.example.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
</bean>
```

## 6. 常见面试题

**Q1：MyBatis的执行流程是什么？**

> 参考答案：1）加载配置，创建SqlSessionFactory；2）通过Factory创建SqlSession；3）获取Mapper代理对象；4）调用方法时，MapperProxy.invoke()；5）通过SqlSession执行SQL；6）Executor调用StatementHandler；7）StatementHandler调用PreparedStatement执行SQL；8）ParameterHandler设置参数；9）PreparedStatement.execute()执行；10）ResultSetHandler处理结果返回。

**Q2：MyBatis一级缓存和二级缓存的区别？**

> 参考答案：一级缓存是SqlSession级别的，默认开启，不可关闭，生命周期随SqlSession；二级缓存是Mapper（namespace）级别的，需要手动开启，跨SqlSession共享。一级缓存存储在PerpetualCache（HashMap）中，commit或close时会清空；二级缓存在事务提交后才写入。一级缓存命中率高但隔离性差，二级缓存隔离性好但可能出现脏读。

**Q3：#{}和${}的区别？**

> 参考答案：#{}使用PreparedStatement，会将参数替换为?占位符，防止SQL注入，是安全的；${}是直接字符串拼接，不做任何处理，存在SQL注入风险，应该尽量避免使用。但在某些场景下必须使用${}，如：动态表名、列名，order by #{column}会变成order by 'id'（字符串），而order by ${column}会变成order by id。

**Q4：MyBatis插件的原理？**

> 参考答案：MyBatis插件基于拦截器链模式，在创建Executor、StatementHandler、ParameterHandler、ResultSetHandler时会调用InterceptorChain.pluginAll()，遍历所有注册的插件，层层代理。插件使用@Intercepts注解标记要拦截的方法，使用@Signature指定拦截的类型和方法。执行时按照配置顺序形成责任链，依次调用各拦截器的intercept方法，通过invocation.proceed()控制执行流向。

## 总结

```
┌────────────────────────────────────────────────────────────────┐
│                      MyBatis核心原理                            │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  映射机制：                                                       │
│  ├─ XML配置: resultMap, SQL, 动态标签                            │
│  ├─ Mapper接口绑定: MapperProxy                                 │
│  └─ 动态SQL: SqlSource, BoundSql                                │
│                                                                │
│  执行流程：                                                       │
│  ├─ SqlSession -> Executor -> StatementHandler                 │
│  ├─ ParameterHandler: 参数设置                                 │
│  ├─ PreparedStatement: 执行SQL                                 │
│  └─ ResultSetHandler: 结果映射                                  │
│                                                                │
│  缓存：                                                           │
│  ├─ 一级: SqlSession级别, PerpetualCache                        │
│  └─ 二级: Mapper级别, CachingExecutor                           │
│                                                                │
│  插件：                                                           │
│  ├─ Interceptor接口                                            │
│  ├─ @Intercepts + @Signature                                   │
│  └─ 责任链模式层层代理                                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

理解MyBatis的核心原理有助于更好地使用框架、排查问题，以及进行自定义扩展。MyBatis的设计体现了职责分离、插件化、可扩展等优秀的设计原则。
