作者：java架构

链接：https://zhuanlan.zhihu.com/p/121646778

来源：知乎

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

### 1、什么是Mybatis？

1、Mybatis 是一个半 ORM（ 对象关系映射）框架，它内部封装了 JDBC，开发时只需要关注 SQL 语句本身， 不需要花费精力去处理加载驱动、创建连接、创建

statement 等繁杂的过程。程序员直接编写原生态 sql，可以严格控制 sql 执行性能， 灵活度高。

2、MyBatis 可以使用 XML 或注解来配置和映射原生信息， 将 POJO 映射成数据库中的记录， 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。

3、通过 xml 文件或注解的方式将要执行的各种 statement 配置起来， 并通过java 对象和 statement 中 sql 的动态参数进行映射生成最终执行的 sql 语句，最后由 mybatis 框架执行 sql 并将结果映射为 java 对象并返回。（ 从执行 sql 到返回 result 的过程）。

### 2、Mybaits 的优点：

1、基于 SQL 语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，SQL 写在 XML 里，解除 sql 与程序代码的耦合，便于统一管理；提供 XML 标签， 支持编写动态 SQL 语句， 并可重用。

2、与 JDBC 相比，减少了 50%以上的代码量，消除了 JDBC 大量冗余的代码，不需要手动开关连接；

3、很好的与各种数据库兼容（ 因为 MyBatis 使用 JDBC 来连接数据库，所以只要JDBC 支持的数据库 MyBatis 都支持）。

4、能够与 Spring 很好的集成；

5、提供映射标签， 支持对象与数据库的 ORM 字段关系映射； 提供对象关系映射标签， 支持对象关系组件维护。

### 3、MyBatis 框架的缺点：

1、SQL 语句的编写工作量较大， 尤其当字段多、关联表多时， 对开发人员编写SQL 语句的功底有一定要求。

2、SQL 语句依赖于数据库， 导致数据库移植性差， 不能随意更换数据库。

### 4、MyBatis 框架适用场合：

1、MyBatis 专注于 SQL 本身， 是一个足够灵活的 DAO 层解决方案。

2、对性能的要求很高，或者需求变化较多的项目，如互联网项目， MyBatis 将是不错的选择。

### 5、MyBatis 与Hibernate 有哪些不同？

1、Mybatis 和 hibernate 不同，它不完全是一个 ORM 框架，因为 MyBatis 需要程序员自己编写 Sql 语句。

2、Mybatis 直接编写原生态 sql， 可以严格控制 sql 执行性能， 灵活度高， 非常适合对关系数据模型要求不高的软件开发， 因为这类软件需求变化频繁， 一但需求变化要求迅速输出成果。但是灵活的前提是 mybatis 无法做到数据库无关性， 如果需要实现支持多种数据库的软件，则需要自定义多套 sql 映射文件，工作量大。

3、Hibernate 对象/关系映射能力强， 数据库无关性好， 对于关系模型要求高的软件， 如果用 hibernate 开发可以节省很多代码， 提高效率。

### 6、#{}和${}的区别是什么？

\#{}是预编译处理， ${}是字符串替换。

Mybatis 在处理#{}时，会将 sql 中的#{}替换为?号，调用 PreparedStatement 的set 方法来赋值；

Mybatis 在处理

{}替换成变量的值。使用#{}可以有效的防止 SQL 注入， 提高系统安全性。

### 7、当实体类中的属性名和表中的字段名不一样 ，怎么办 ？

第 1 种： 通过在查询的 sql 语句中定义字段名的别名， 让字段名的别名和实体类的属性名一致。

```text
<select id=”selectorder”  parametertype=”int”  resultetype=” me.gacl.domain.order”>

select order_id id, order_no orderno ,order_price price form orders where order_id=#{id};

</select>
```

第 2 种： 通过来映射字段名和实体类属性名的一一对应的关系。

```text
<select id="getOrder" parameterType="int" resultMap="orderresultmap">
select * from orders where order_id=#{id}
</select>
<resultMap type=”me.gacl.domain.order”  id=”orderresultmap”>
<!–用 id 属性来映射主键字段–>
<id property=”id”  column=”order_id”>
<!–用 result 属性来映射非主键字段，property 为实体类属性名，column 为数据表中的属性–>
<result property =  “orderno”  column =”order_no”/>
<result property=”price”  column=”order_price”  />
</reslutMap>
```

### 9、通常一个Xml 映射文件，都会写一个Dao 接口与之对应，

请问，这个Dao 接口的工作原理是什么？Dao 接口里的方法， 参数不同时，方法能重载吗？

Dao 接口即 Mapper 接口。接口的全限名，就是映射文件中的 namespace 的值； 接口的方法名， 就是映射文件中 Mapper 的 Statement 的 id 值； 接口方法内的参数， 就是传递给 sql 的参数.

Mapper 接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为 key 值， 可唯一定位一个 MapperStatement。在 Mybatis 中， 每一个 select、insert 、update、delete标签， 都会被解析为一个MapperStatement 对象. 举例： com.mybatis3.mappers.StudentDao.findStudentById， 可以唯一 找 到 namespace 为 com.mybatis3.mappers.StudentDao 下 面 id 为findStudentById 的 MapperStatement 。 Mapper 接口里的方法，是不能重载的，因为是使用 全限名+方法名 的保存和寻找策略。Mapper 接口的工作原理是 JDK 动态代理， Mybatis 运行时会使用 JDK 动态代理为 Mapper 接口生成代理对象 proxy， 代理对象会拦截接口方法， 转而执行 MapperStatement 所代表的 sql， 然后将 sql 执行结果返回。

### 10、Mybatis 是如何进行分页的？分页插件的原理是什么？

Mybatis 使用 RowBounds 对象进行分页， 它是针对 ResultSet 结果集执行的内存分页，而非物理分页。可以在 sql 内直接书写带有物理分页的参数来完成物理分页功能， 也可以使用分页插件来完成物理分页。

分页插件的基本原理是使用 Mybatis 提供的插件接口， 实现自定义插件， 在插件的拦截方法内拦截待执行的 sql，然后重写 sql，根据 dialect 方言，添加对应的物理分页语句和物理分页参数。

### 11、Mybatis 是如何将sql 执行结果封装为目标对象并返回的？ 都有哪些映射形式？

第一种是使用标签， 逐一定义数据库列名和对象属性名之间的映射关系。

第二种是使用 sql 列的别名功能， 将列的别名书写为对象属性名。

有了列名与属性名的映射关系后， Mybatis 通过反射创建对象， 同时使用反射给对象的属性逐一赋值并返回， 那些找不到映射关系的属性， 是无法完成赋值的。

### 13、如何获取自动生成的(主)键值?

insert 方法总是返回一个 int 值 ， 这个值代表的是插入的行数。

如果采用自增长策略，自动生成的键值在 insert 方法执行完后可以被设置到传入的参数对象中。

示例：

```text
<insert id=”insertname”  usegeneratedkeys=”true”  keyproperty=” id”>

insert into names (name) values (#{name})

</insert>

name name = new name(); name.setname(“fred”);
int rows = mapper.insertname(name);
// 完成后,id 已经被设置到对象中system.out.println(“rows inserted =  ”  + rows);
system.out.println(“generated key value =  ”  + name.getid());
```

### 14、在 mapper 中如何传递多个参数?

1、第一种： DAO 层的函数

public UserselectUser(String name,String area);

对应的 xml,#{0}代表接收的是 dao 层中的第一个参数，#{1}代表 dao 层中第二参数，更多参数一致往后加即可。

```text
<select id="selectUser"resultMap="BaseResultMap"> select *                  fromuser_user_t         whereuser_name = #{0}

anduser_area=#{1}

</select>
```

2、第二种： 使用 @param 注解:

```text
public interface usermapper {

user selectuser(@param(“username”) string username,@param(“hashedpassword”) string hashedpassword);

}
```

然后,就可以在 xml 像下面这样使用(推荐封装为一个 map,作为单个参数传递给mapper):

```text
<select id=”selectuser”  resulttype=”user”> select id, username, hashedpassword from some_table

where username = #{username}

and hashedpassword = #{hashedpassword}

</select>
```

3、第三种： 多个参数封装成 map

```text
try {

//映射文件的命名空间.SQL 片段的 ID，就可以调用对应的映射文件中的

SQL

//由于我们的参数超过了两个，而方法中只有一个 Object 参数收集，因此我们使用 Map 集合来装载我们的参数

Map < String, Object > map = new HashMap(); map.put("start", start);

map.put("end", end);

return sqlSession.selectList("StudentID.pagination", map);

} catch (Exception e) { e.printStackTrace(); sqlSession.rollback(); throw e;

} finally {

MybatisUtil.closeSqlSession();

}
```

### 15、Mybatis 动态sql 有什么用？执行原理？有哪些动态sql？

Mybatis 动态 sql 可以在 Xml 映射文件内，以标签的形式编写动态 sql，执行原理是根据表达式的值 完成逻辑判断并动态拼接 sql 的功能。

Mybatis 提供了 9 种动态 sql 标签：trim | where | set | foreach | if | choose

| when | otherwise | bind 。

16、Xml 映射文件中，除了常见的 select|insert|updae|delete 标签之外，还有哪些标签？

答： 、、、、

， 加上动态 sql 的 9 个标签， 其中为 sql 片段标签， 通过

标签引入 sql 片段，为不支持自增的主键生成策略标签。

17、Mybatis 的 Xml 映射文件中， 不同的 Xml 映射文件， id 是否可以重复？

不同的 Xml 映射文件， 如果配置了 namespace， 那么 id 可以重复； 如果没有配置 namespace， 那么 id 不能重复；

原因就是 namespace+id 是作为 Map<String, MapperStatement>的 key 使用的， 如果没有 namespace， 就剩下 id， 那么， id 重复会导致数据互相覆盖。有了 namespace，自然 id 就可以重复，namespace 不同，namespace+id 自然也就不同。

### 18、为什么说 Mybatis 是半自动 ORM 映射工具？它与全自动的区别在哪里？

Hibernate 属于全自动 ORM 映射工具， 使用 Hibernate 查询关联对象或者关联集合对象时， 可以根据对象关系模型直接获取， 所以它是全自动的。而 Mybatis 在查询关联对象或关联集合对象时，需要手动编写 sql 来完成，所以，称之为半自动 ORM 映射工具。

### 20、MyBatis 实现一对一有几种方式?具体怎么操作的？

有联合查询和嵌套查询,联合查询是几个表联合查询,只查询一次, 通过在resultMap 里面配置 association 节点配置一对一的类就可以完成；

嵌套查询是先查一个表，根据这个表里面的结果的 外键 id，去再另外一个表里面查询数据,也是通过 association 配置，但另外一个表的查询通过 select 属性配置。

### 21、MyBatis 实现一对多有几种方式,怎么操作的？

有联合查询和嵌套查询。联合查询是几个表联合查询,只查询一次,通过在

resultMap 里面的 collection 节点配置一对多的类就可以完成； 嵌套查询是先查一个表,根据这个表里面的 结果的外键 id,去再另外一个表里面查询数据,也是通过配置 collection,但另外一个表的查询通过 select 节点配置。

### 22、Mybatis 是否支持延迟加载？如果支持，它的实现原理是什么？

答： Mybatis 仅支持 association 关联对象和 collection 关联集合对象的延迟加载， association 指的就是一对一， collection 指的就是一对多查询。在 Mybatis 配置文件中， 可以配置是否启用延迟加载 lazyLoadingEnabled=true|false。

它的原理是， 使用 CGLIB 创建目标对象的代理对象， 当调用目标方法时， 进入拦截器方法， 比如调用 a.getB().getName()， 拦截器 invoke()方法发现 a.getB()是null 值， 那么就会单独发送事先保存好的查询关联 B 对象的 sql， 把 B 查询上来， 然后调用 a.setB(b)，于是 a 的对象 b 属性就有值了，接着完成 a.getB().getName()方法的调用。这就是延迟加载的基本原理。

当然了， 不光是 Mybatis， 几乎所有的包括 Hibernate， 支持延迟加载的原理都是一样的。

### 23、Mybatis 的一级、二级缓存:

1） 一级缓存: 基于 PerpetualCache 的 HashMap 本地缓存， 其存储作用域为Session， 当 Session flush 或 close 之后， 该 Session 中的所有 Cache 就将清空， 默认打开一级缓存。

2） 二级缓存与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap 存储， 不同在于其存储作用域为 Mapper(Namespace)， 并且可自定义存储源， 如 Ehcache。默认不打开二级缓存， 要开启二级缓存， 使用二级缓存属性类需要实现 Serializable 序列化接口(可用来保存对象的状态),可在它的映射文件中配置

3） 对于缓存数据更新机制， 当某一个作用域(一级缓存 Session/二级缓存

Namespaces)的进行了 C/U/D 操作后，默认该作用域下所有 select 中的缓存将被 clear 。

### 24、什么是 MyBatis 的接口绑定？有哪些实现方式？

接口绑定，就是在 MyBatis 中任意定义接口,然后把接口里面的方法和 SQL 语句绑定, 我们直接调用接口方法就可以,这样比起原来了 SqlSession 提供的方法我们可以有更加灵活的选择和设置。



接口绑定有两种实现方式,一种是通过注解绑定， 就是在接口的方法上面加上@Select、@Update 等注解， 里面包含 Sql 语句来绑定； 另外一种就是通过 xml 里面写 SQL 来绑定, 在这种情况下,要指定 xml 映射文件里面的 namespace 必须为接口的全路径名。当 Sql 语句比较简单时候,用注解绑定, 当 SQL 语句比较复杂时候,用 xml 绑定,一般用 xml 绑定的比较多。

### 25、使用 MyBatis 的mapper 接口调用时有哪些要求？

1、Mapper 接口方法名和 mapper.xml 中定义的每个 sql 的 id 相同；

2、Mapper 接口方法的输入参数类型和 mapper.xml 中定义的每个 sql 的parameterType 的类型相同；

3、Mapper 接口方法的输出参数类型和 mapper.xml 中定义的每个 sql 的resultType 的类型相同；

4、Mapper.xml 文件中的 namespace 即是 mapper 接口的类路径。

### 26、Mapper 编写有哪几种方式？

第一种： 接口实现类继承 SqlSessionDaoSupport： 使用此种方法需要编写mapper 接口， mapper 接口实现类、mapper.xml 文件。



1、在 sqlMapConfig.xml 中配置 mapper.xml 的位置

```text
<mappers>
<mapper resource="mapper.xml 文件的地址" />
<mapper resource="mapper.xml 文件的地址" />
</mappers>
```

1、定义 mapper 接口

3、实现类集成 SqlSessionDaoSupport

mapper 方法中可以 this.getSqlSession()进行数据增删改查。

4、spring 配置

```text
<bean id=" " class="mapper 接口的实现">

<property name="sqlSessionFactory" ref="sqlSessionFactory"></property>

</bean>
```

第二种： 使用 org.mybatis.spring.mapper.MapperFactoryBean：



1、在 sqlMapConfig.xml 中配置 mapper.xml 的位置， 如果 mapper.xml 和mappre 接口的名称相同且在同一个目录， 这里可以不用配置

```text
<mappers>

<mapper resource="mapper.xml 文件的地址" />

<mapper resource="mapper.xml 文件的地址" />

</mappers>
```

2、定义 mapper 接口：

1、mapper.xml 中的 namespace 为 mapper 接口的地址

2、mapper 接口中的方法名和 mapper.xml 中的定义的 statement 的 id 保持一致

3、Spring 中定义

```text
<bean id="" class="org.mybatis.spring.mapper.MapperFactoryBean">

<property name="mapperInterface"              value="mapper 接口地址" />

<property name="sqlSessionFactory" ref="sqlSessionFactory" />

</bean>
```

第三种： 使用 mapper 扫描器：

1、mapper.xml 文件编写：

mapper.xml 中的 namespace 为 mapper 接口的地址；

mapper 接口中的方法名和 mapper.xml 中的定义的 statement 的 id 保持一致；

如果将 mapper.xml 和 mapper 接口的名称保持一致则不用在 sqlMapConfig.xml 中进行配置。

2、定义 mapper 接口：

注意 mapper.xml 的文件名和 mapper 的接口名称保持一致， 且放在同一个目录

3、配置 mapper 扫描器：

```text
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">

<property name="basePackage" value="mapper 接口包地址"></property>

<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>

</bean>
```

4、使用扫描器后从 spring 容器中获取 mapper 的实现对象。

### 27、简述 Mybatis 的插件运行原理，以及如何编写一个插件。

答： Mybatis 仅可以编写针对 ParameterHandler、ResultSetHandler、

StatementHandler、Executor 这 4 种接口的插件， Mybatis 使用 JDK 的动态代理， 为需要拦截的接口生成代理对象以实现接口方法拦截功能， 每当执行这 4 种接口对象的方法时，就会进入拦截方法，具体就是 InvocationHandler 的 invoke() 方法， 当然， 只会拦截那些你指定需要拦截的方法。



## XML 映射配置文件

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

    



## **介绍**

- MyBatis-Plus(简称 MP),是一个 MyBatis 的增强工具包，只做增强不做改变. 为简化开发工作、提高生产率而生 我们的愿景是成为 Mybatis 最好的搭档，就像 魂斗罗 中的 1P、2P，基友搭配，效率翻倍。 

- 问题

  - 假设我们已存在一张 tbl_employee 表，且已有对应的实体类 Employee，实现tbl_employee 表的 CRUD 操作我们需要做什么呢？
  - 实现方式:
    - 基于 Mybatis 需要编写 EmployeeMapper 接口，并手动编写 CRUD 方法，提供 EmployeeMapper.xml 映射文件，并手动编写每个方法对应的 SQL 语句. 
    - 基于 MP 只需要创建EmployeeMapper 接口, 并继承 BaseMapper 接口.这就是使用 MP需要完成的所有操作，甚至不需要创建 SQL 映射文件。

- 每一个 mappedStatement 都表示 Mapper 接口中的一个方法与 Mapper 映射文件中的一个 SQL。 MP 在启动就会挨个分析 xxxMapper 中的方法，并且将对应的 SQL 语句处理好，保存到 configuration 对象中的 mappedStatements 中.

- MybatisPlus会默认使用实体类的类名到数据中找对应的表.

  - ```java
    @TableName(value="tbl_employee")
    // @TableId:
    //	value: 指定表中的主键列的列名， 如果实体属性名与列名一致，可以省略不指定. 
    //	type: 指定主键策略. 
    @TableId(value="id" , type =IdType.AUTO)
    
    ```

- service层

  - ```java
    public interface EmployeeService extends IService<Employee>{}
    
    	//不用再进行mapper的注入.
    	/**
    	 * EmployeeServiceImpl  继承了ServiceImpl
    	 * 1. 在ServiceImpl中已经完成Mapper对象的注入,直接在EmployeeServiceImpl中进行使用
    	 * 2. 在ServiceImpl中也帮我们提供了常用的CRUD方法， 基本的一些CRUD方法在Service中不需要我们自己定义.
    	 * 
    	 * 
    	 */ 
    @Service
    public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee> implements EmployeeService {}
    ```



## 条件构造器EntityWrapper

- 概述

  - Mybatis-Plus 通过 EntityWrapper（简称 EW，MP 封装的一个查询条件构造器）或者Condition（与 EW 类似） 来让用户自由的构建查询条件，简单便捷，没有额外的负担，能够有效提高开发效率
  - 实体包装器，主要用于处理 sql 拼接，排序，实体参数查询等
  - 注意: 使用的是**数据库字段**，不是 Java 属性!

- ```java
  Page<Employee> page = employee.selectPage(
      new Page<Employee>(1, 1),
      new EntityWrapper<Employee>().like("last_name", "老"));
  List<Employee> emps = page.getRecords();
  System.out.println(emps);
  
  // ------
  
  Page<Employee> userListCondition = employeeMapper.selectPage(new Page<Employee>(2,3), Condition.create()
  .eq("gender", 1)
  .eq("last_name", "MyBatisPlus")
  .between("age", 18, 50));
                                                      
  ```





## ActiveRecord(活动记录)

- Active Record(活动记录)，是一种**领域模型模式**，特点是一个模型类对应关系型数据库中的一个表，而模型类的一个实例对应表中的一行记录。

- 仅仅需要让实体类继承 Model 类且实现主键指定方法，即可开启 AR 之旅.

- ```java
  @TableName("tbl_employee")
  public class Employee extends Model<Employee>{
  // .. fields
  // .. getter and setter
  @Override
  protected Serializable pkVal() {
  return this.id; 
  }
  ```







## 自定义全局操作

- ```java
  /**
   * 自定义全局操作
   */
  public class MySqlInjector  extends AutoSqlInjector{
  
      /**
  	 * 扩展inject 方法，完成自定义全局操作
  	 */
      @Override
      public void inject(Configuration configuration, MapperBuilderAssistant builderAssistant, Class<?> mapperClass,Class<?> modelClass, TableInfo table) {
          //将EmployeeMapper中定义的deleteAll， 处理成对应的MappedStatement对象，加入到configuration对象中。
  
          //注入的SQL语句
          String sql = "delete from " +table.getTableName();
          //注入的方法名   一定要与EmployeeMapper接口中的方法名一致
          String method = "deleteAll" ;
  
          //构造SqlSource对象
          SqlSource sqlSource = languageDriver.createSqlSource(configuration, sql, modelClass);
  
          //构造一个删除的MappedStatement
          this.addDeleteMappedStatement(mapperClass, method, sqlSource);
  
      }
  }
  
  
  public interface EmployeeMapper extends BaseMapper<Employee> {
  	
  	int  deleteAll();
  }
  
  ```

  