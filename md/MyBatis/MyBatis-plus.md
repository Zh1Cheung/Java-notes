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

  