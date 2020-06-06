## **XML 映射配置文件**

- MyBatis 的配置文件包含了影响 MyBatis 行为甚深的设置（settings）和属性（properties）信息。
- properties
  - 这些属性都是可外部配置且可动态替换的，既可以在典型的 Java 属性文件中配置，亦可通过 properties 元素的子元素来传递。
  - 将按照下面的顺序来加载：通过方法参数传递的属性具有最高优先级，resource/url 属性中指定的配置文件次之，最低优先级的是 properties 属性中指定的属性。
- settings
  - 它们会改变 MyBatis 的运行时行为

| 设置参数                  | 描述                                                         | 有效值                                                       | 默认值                                                       |
| :------------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| cacheEnabled              | 该配置影响的所有映射器中配置的缓存的全局开关。               | true,false                                                   | true                                                         |
| lazyLoadingEnabled        | 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。 特定关联关系中可通过设置fetchType属性来覆盖该项的开关状态。 | true,false                                                   | false                                                        |
| aggressiveLazyLoading     | 当启用时，对任意延迟属性的调用会使带有延迟加载属性的对象完整加载；反之，每种属性将会按需加载。 | true,false,true                                              |                                                              |
| multipleResultSetsEnabled | 是否允许单一语句返回多结果集（需要兼容驱动）。               | true,false                                                   | true                                                         |
| useColumnLabel            | 使用列标签代替列名。不同的驱动在这方面会有不同的表现， 具体可参考相关驱动文档或通过测试这两种不同的模式来观察所用驱动的结果。 | true,false                                                   | true                                                         |
| useGeneratedKeys          | 允许 JDBC 支持自动生成主键，需要驱动兼容。 如果设置为 true 则这个设置强制使用自动生成主键，尽管一些驱动不能兼容但仍可正常工作（比如 Derby）。 | true,false                                                   | False                                                        |
| autoMappingBehavior       | 指定 MyBatis 应如何自动映射列到字段或属性。 NONE 表示取消自动映射；PARTIAL 只会自动映射没有定义嵌套结果集映射的结果集。 FULL 会自动映射任意复杂的结果集（无论是否嵌套）。 | NONE, PARTIAL, FULL                                          | PARTIAL                                                      |
| defaultExecutorType       | 配置默认的执行器。SIMPLE 就是普通的执行器；REUSE 执行器会重用预处理语句（prepared statements）； BATCH 执行器将重用语句并执行批量更新。 | SIMPLE REUSE BATCH                                           | SIMPLE                                                       |
| defaultStatementTimeout   | 设置超时时间，它决定驱动等待数据库响应的秒数。               | Any positive integer                                         | Not Set (null)                                               |
| safeRowBoundsEnabled      | 允许在嵌套语句中使用分页（RowBounds）。                      | true,false                                                   | False                                                        |
| mapUnderscoreToCamelCase  | 是否开启自动驼峰命名规则（camel case）映射，即从经典数据库列名 A_COLUMN 到经典 Java 属性名 aColumn 的类似映射。 | true, false                                                  | False                                                        |
| localCacheScope           | MyBatis 利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询。 默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。 若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据。 | SESSION,STATEMENT                                            | SESSION                                                      |
| jdbcTypeForNull           | 当没有为参数提供特定的 JDBC 类型时，为空值指定 JDBC 类型。 某些驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER。 | JdbcType enumeration. Most common are: NULL, VARCHAR and OTHER | OTHER                                                        |
| lazyLoadTriggerMethods    | 指定哪个对象的方法触发一次延迟加载。                         | A method name list separated by commas                       | equals,clone,hashCode,toString                               |
| defaultScriptingLanguage  | 指定动态 SQL 生成的默认语言。                                | A type alias or fully qualified class name.                  | org.apache.ibatis.scripting.xmltags.XMLDynamicLanguageDriver |
| callSettersOnNulls        | 指定当结果集中值为 null 的时候是否调用映射对象的 setter（map 对象时为 put）方法，这对于有 Map.keySet() 依赖或 null 值初始化的时候是有用的。注意基本类型（int、boolean等）是不能设置成 null 的。 | true,false                                                   | false                                                        |
| logPrefix                 | 指定 MyBatis 增加到日志名称的前缀。                          | Any String                                                   | Not set                                                      |
| logImpl                   | 指定 MyBatis 所用日志的具体实现，未指定时将自动查找。        | SLF4J,LOG4J,LOG4J2,JDK_LOGGING,COMMONS_LOGGING,STDOUT_LOGGING,NO_LOGGING | Not set                                                      |
| proxyFactory              | 指定 Mybatis 创建具有延迟加载能力的对象所用到的代理工具。    | CGLIB JAVASSIST                                              |                                                              |



- TypeHandler

  - 无论是 MyBatis 在预处理语句（PreparedStatement）中设置一个参数时，还是从结果集中取出一个值时， 都会用类型处理器将获取的值以合适的方式转换成 Java 类型。下表描述了一些默认的类型处理器。

- typeAlias

  - 类型别名。使用它们你可以不用输入类的全路径

  - > <typeAlias type="com.someapp.model.User" alias="User"/>

- mappers 

  - 注册一个sql映射 

  - ```xml
    <!-- <mapper resource="mybatis/mapper/EmployeeMapper.xml"/> -->
    <!-- <mapper class="EmployeeMapperAnnotation"/> -->
    
    <!-- 批量注册： -->
    <package name="conf.com.Zh1Cheung.mybatis.dao"/>
    ```

    



## MyBatis XML映射文件

- SQL 映射文件有很少的几个顶级元素（按照它们应该被定义的顺序）

  - > `cache` – 给定命名空间的缓存配置。
    >
    > `cache-ref` – 其他命名空间缓存配置的引用。
    >
    > `resultMap` – 是最复杂也是最强大的元素，用来描述如何从数据库结果集中来加载对象。
    >
    > `sql` – 可被其他语句引用的可重用语句块。
    >
    > `insert` – 映射插入语句
    >
    > `update` – 映射更新语句
    >
    > `delete` – 映射删除语句
    >
    > `select` – 映射查询语句

- Result Maps

  - ResultMap 的设计就是简单语句不需要明确的结果映射,而很多复杂语句确实需要描述它们 的关系。

  - 如果列名没有精确匹配,你可以在列名上使用 select 字句的别名(一个基本的 SQL特性)来匹配标签（select user_Id as "id" xxx）

  - 外部的 resultMap 是什么样子的,这也是解决列名不匹配的另外一种方式。

    - ```xml
      <resultMap id="userResultMap" type="User">
        <id property="id" column="user_id" />
        <result property="username" column="username"/>
        <result property="password" column="password"/>
      </resultMap> 
      
      <select id="selectUsers" resultMap="userResultMap">
        select user_id, user_name, hashed_password
        from some_table
        where id = #{id}
      </select>
      
      ```

  - > - `id` – 一个 ID 结果;标记结果作为 ID 可以帮助提高整体效能
    > - `result` – 注入到字段或 JavaBean 属性的普通结果
    > - `association` 一个复杂的类型关联;许多结果将包成这种类型 
    >   - 嵌入结果映射 – 结果映射自身的关联,或者参考一个
    >
    > - `collection` 复杂类型的集 
    >   - 嵌入结果映射 – 结果映射自身的集,或者参考一个

  - ```xml
    <association property="author" column="blog_author_id" javaType="Author">
      <id property="id" column="author_id"/>
      <result property="username" column="author_username"/>
    </association>
    
    ---
    
    <resultMap id="blogResult" type="Blog">
      <association property="author" column="author_id" javaType="Author" select="selectAuthor"/>
    </resultMap>
    
    ----
    
    
    <resultMap id="blogResult" type="Blog">
      <id property="id" column="blog_id" />
      <result property="title" column="blog_title"/>
      <association property="author" column="blog_author_id" javaType="Author" resultMap="authorResult"/>
    </resultMap>
    
    <resultMap id="authorResult" type="Author">
      <id property="id" column="author_id"/>
      <result property="username" column="author_username"/>
      <result property="password" column="author_password"/>
      <result property="email" column="author_email"/>
      <result property="bio" column="author_bio"/>
    </resultMap>
    
    ```

    ```xml
    <collection property="posts" ofType="domain.blog.Post">
      <id property="id" column="post_id"/>
      <result property="subject" column="post_subject"/>
      <result property="body" column="post_body"/>
    </collection>
    
    ---
    
    <!--
    
    在 Post 类型的 ArrayList 中的 posts 的集合
    "ofType"属性。这个属性用来区分 JavaBean(或字段)属性类型和集合包含的类型来说是很重要的
    javaType 属性是不需要的,因为 MyBatis 在很多情况下会为你算出来。
    
    -->
    
    <resultMap id="blogResult" type="Blog">
      <collection property="posts" select="selectPostsForBlog" column="id" ofType="Post" />
    </resultMap>
    
    	<resultMap type="Department" id="MyDeptStep">
    		<id column="id" property="id"/>
    		<id column="dept_name" property="departmentName"/>
    		<collection property="emps" 
    			select="EmployeeMapperPlus.getEmpsByDeptId"
    			column="{deptId=id}" fetchType="lazy"></collection>
    	</resultMap>
    	<!-- public Department getDeptByIdStep(Integer id); -->
    	<select id="getDeptByIdStep" resultMap="MyDeptStep">
    		select id,dept_name from tbl_dept where id=#{id}
    	</select>
    ```

    

    

## 动态SQL

- 标签

  - > - if
    > - choose (when, otherwise)
    > - trim (where, set)
    >   - where 元素知道只有在一个以上的if条件有值的情况下才去插入"WHERE"子句。
    >   - set 元素可以被用于动态包含需要更新的列，而舍去其他的。
    > - foreach

  - ```xml
    <where> 
        <if test="state != null">
            state = #{state}
        </if> 
        <if test="title != null">
            AND title like #{title}
        </if>
        <if test="author != null and author.name != null">
            AND author_name like #{author.name}
        </if>
    </where>
    
    <set>
        <if test="username != null">username=#{username},</if>
        <if test="password != null">password=#{password},</if>
        <if test="email != null">email=#{email},</if>
        <if test="bio != null">bio=#{bio}</if>
    </set>
    
    <foreach item="item" index="index" collection="list"
             open="(" separator="," close=")">
        #{item}
    </foreach>
    ```

    



## MyBatis 原理

- > 1.
  > SQLSessionFactory初始化（根据配置文件创建SqlSessionFactory）
  > 	拿到Mybatis全局配置
  > 	创建SqlSessionFactoryBuilder
  > 		build()
  > 			创建解析器XMLConifigBuilder
  > 				paser()解析configuration里的每一个标签并保存在Configuration中
  > 			    解析mapper.xml封装成一个MappedStatement
  > 			    	一个MappedStatement就代表一个增删改查标签的详细信息
  > 			    build(Configuration)->new DefaultSqlSession()
  > 	创建DefaultSqlSession并返回，包含了保存全局配置信息的Configuration
  >
  > 2.
  > openSession()获取SqlSession对象
  > 	调用DefaultSqlSessionFasctory的openSession()
  > 		调用OpenSessionFromDataSource
  > 			创建事务
  > 			new Executor()
  > 				根据Executor在全局配置中的类型，创建出SimpleExecutor/ReuseExecutor/BatchExecutor
  > 				如果有二级缓存配置开启，创建CachingExecutor
  > 				使用每一个拦截器重新包装executor并返回
  > 					interceptorChain.pluginAll(executor)
  > 				创建DefaultSqlSession并返回，包含了Configuration和Executor
  >
  > 3.
  > getMapper获取到接口的代理对象
  > 	调用DefaultSqlSessionFasctory的getMapper()
  > 		MapperRegistry调用getMapper()
  > 			根据接口类型获取MapperProxyFactory
  > 				newInstance(sqlSession)
  > 					创建MapperProxy，他是一个InvocationHandler
  > 						创建并返回MapperProxy代理对象
  >
  > 4. 查询实现
  > MapperProxy调用invoke()
  > 		MapperMethod判断增删改查类型并包装参数
  > 			SqlSession.selectOne()
  > 				selectList()
  > 				获取MappedStatement
  > 				executor.query()
  > 					获取BoundSql，他代表sql语句的详细信息
  > 					查看本地缓存是否有数据，没有就调用queryFromDatabase，查出以后保存在本地缓存
  > 						doQuery()
  > 							select查询标签里有statementType就是statementHandler 因为是ResultType 所以是PreparedstatementHandler
  > 							创建StatementHandler对象（其实是PreparedStatementHandler对象）(interceptorChain.pluginAll(statementHandler))
  > 								创建ParameterHander
  > 									interceptorChain.pluginAll(parameterHandler)
  > 								创建ResultSetHander
  > 									interceptorChain.pluginAll(resultSetHandler)
  > 								预编译sql产生PreparedStatement对象
  > 								调用parameterHandler设置参数	
  > 								查出数据使用ResultSetHandler处理结果，结果使用TypeHandler进行数据类型映射获取value值
  > 			返回list的第一个
  >
  > 
  >
  > 查询流程总结：
  > 代理对象
  > 	包含了DefaultSqlSession
  > 		使用Executot执行增删改查
  > 			创建statementHander处理Sql语句预编译
  > 				通过ParameterHander设置参数
  > 				通过ResultSetHander处理结果
  > 				通过TypeHander进行数据类型映射
  >
  > 所有的底层操作都是调用JDBC获得PrepareStatement

  

- > 参数映射：ParameterHandler
  > SQL解析：SqlSource
  > SQL执行：Executor
  > 结果映射和处理；ResultSetHandler
  >
  > 
  >
  > 四大对象：
  > 	Executor StatementHander ParameterHander ResultSetHander





## Mybatis插件原理

- 在四大对象创建的时候
  - 每个创建出来的对象不是直接返回的
    - 而是调用interceptorChain.pluginAll(parameterHandler)返回的;
    - 如果在注解中则调用plugin创建代理对象，不在就直接返回
  - 获取到所有的Interceptor（拦截器）（插件需要实现的接口）；
    - 调用interceptor.plugin(target);返回target包装后的对象
  - 插件机制，我们可以使用插件为目标对象创建一个代理对象	
    - 我们的插件可以为四大对象创建出代理对象；
    - 代理对象就可以拦截到四大对象的每一个执行；

-  插件编写：

  - 编写Interceptor的实现类

  - 使用@Intercepts注解完成插件签名

  - 将写好的插件注册到全局配置文件中

  - ```java
    @Intercepts(
    		{
    			@Signature(type=StatementHandler.class,method="parameterize",args=java.sql.Statement.class)
    		})
    public class MyFirstPlugin implements Interceptor{
    ```

    





## SSM

- applicationContext.xml

  - ```xml
    <!-- 
      整合mybatis 
      目的：1、spring管理所有组件。mapper的实现类。
       				 service==>Dao   @Autowired:自动注入mapper；
      	   2、spring用来管理事务，spring声明式事务
     -->
    
    <!--创建出SqlSessionFactory对象  -->
    <bean id="sqlSessionFactoryBean" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"></property>
        <!-- configLocation指定全局配置文件的位置 -->
        <property name="configLocation" value="classpath:conf/mybatis-config.xml"></property>
        <!--mapperLocations: 指定mapper文件的位置-->
        <property name="mapperLocations" value="classpath:conf/mybatis/mapper/*.xml"></property>
    </bean>
    
    <!--配置一个可以进行批量执行的sqlSession  -->
    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
        <constructor-arg name="sqlSessionFactory" ref="sqlSessionFactoryBean"/>
        <constructor-arg name="executorType" value="BATCH"/>
    </bean>
    
    <!-- 扫描所有的mapper接口的实现，让这些mapper能够自动注入；
     base-package：指定mapper接口的包名
      -->
    <mybatis-spring:scan base-package="Zh1Cheung.dao"/>
    <!-- <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
      <property name="basePackage" value="Zh1Cheung.dao"></property>
     </bean> -->
    
    ```

    