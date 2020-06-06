

## Spring声明式事务

- 小心 Spring 的事务可能没有生效

  - 在使用 @Transactional 注解开启声明式事务时， 第一个最容易忽略的问题是，很可能事务并没有生效。
  - @Transactional 生效原则 1，除非特殊配置（比如使用 AspectJ 静态织入实现AOP），否则只有定义在 public 方法上的 @Transactional 才能生效。
  - @Transactional 生效原则 2，必须通过代理过的类从外部调用目标方法才能生效。
    - this 自调用、通过 self 调用（自己的类内部注入自己调用自己的 方法 ），以及在 Controller 中调用UserService 三种实现的区别
      - 通过 this 自调用，没有机会走到 Spring 的代理类
      - 后两种改进方案调用的是 Spring 注入的 XXXService，通过代理调用才有机会对XXXService的方法进行动态增强。
      - CGLIB 通过继承方式实现代理类，private 方法在子类不可见，无法进行事务增强；
  - 一个小技巧，强烈建议你在开发时打开相关的 Debug 日志，以方便了解Spring 事务实现的细节，并及时判断事务的执行情况
    - 我们的 Demo 代码使用 JPA 进行数据库访问，可以这么开启 Debug 日志
    - logging.level.org.springframework.orm.jpa=DEBUG

- 事务即便生效也不一定能回滚

  - 通过 AOP 实现事务处理可以理解为，使用 try…catch…来包裹标记了 @Transactional 注解的方法，当方法出现了异常并且满足一定条件的时候，在 catch 里面我们可以设置事务回滚，没有异常则直接提交事务。

    - 第一，只有异常传播出了标记了 @Transactional 注解的方法，事务才能回滚。
    - 第二，默认情况下，出现 RuntimeException（非受检异常）或 Error 的时候，Spring才会回滚事务。

  - 在 createUserWrong1 方法中会抛出一个 RuntimeException，但由于方法内 catch 了所有异常，异常无法从方法传播出去，事务自然无法回滚

  - 现在，我们来看下修复方式，以及如何通过日志来验证是否修复成功。

    - 第一，如果你希望自己捕获异常进行处理的话，也没关系，可以手动设置让当前事务处于回滚状态

    - 第二，在注解中声明，期望遇到所有的 Exception 都回滚事务（来突破默认不回滚受检异常的限制）

    - ```java
      TransactionAspectSupprot.currentTransactionStatus().setRollbackonly();
      
      @Transaction(rollbackFor=Exception.class)
      ```

- 请确认事务传播配置是否符合自己的业务逻辑

  - 在有些业务逻辑中，可能会包含多次数据库操作，我们不一定希望将两次操作作为一个事务来处理，这时候就需要仔细考虑事务传播的配置了，否则也可能踩坑。

  - 因为运行时异常逃出了 @Transactional 注解标记的createUserWrong 方法，Spring 当然会回滚事务了。如果我们希望主方法不回滚，应该把子方法抛出的异常捕获了。

  - 看到这里，修复方式就很明确了，想办法让子逻辑在独立事务中运行，也就是改一下子用户的方法，为注解加上 propagation =Propagation.REQUIRES_NEW 来设置 REQUIRES_NEW 方式的事务传播策略，也就是执行到这个方法时需要开启新的事务，并挂起当前事务

  - ```java
    @Transactional(propagation = Propagation.REQUIRES_NEW) 
    ```

- @Transactional 与 @Async注解不能同时在一个方法上使用, 这样会导致事物不生效。





## 判等问题

- 注意 equals 和 == 的区别

  - 对基本类型，比如 int、long，进行判等，只能使用 ==，比较的是直接值。因为基本类型的值就是其数值。
  - 对引用类型，比如 Integer、Long 和 String，进行判等，需要使用 equals 进行内容判等。因为引用类型的直接值是指针，使用 == 的话，比较的是指针，也就是两个对象在内存中的地址，即比较它们是不是同一个对象，而不是比较对象的内容。
    - Integer默认情况下会缓存[-128,127]的数值
    - 使用 == 对一个值为 128 的直接赋值的 Integer 对象和另一个值为 128 的 int 基本类型判等,我们把装箱的 Integer 和基本类型 int 比较，前者会先拆箱再比较，比较的肯定是数值而不是引用
    - 对两个 new 出来的值都为 2 的 String 使用 == 判等：new 出来的两个 String 是不同对象，引用当然不同，所以得到 false 的结果。
  - 业务代码中滥用 intern，可能会产生性能问题
    - 其实，原因在于字符串常量池是一个固定容量的 Map。如果容量太小（Number of buckets=60013）、字符串太多（1000 万个字符串），那么每一个桶中的字符串数量会非常多，所以搜索起来就很慢。输出结果中的 Average bucket size=167，代表了 Map中桶的平均长度是 167

- 实现一个 equals 没有这么简单

  - 考虑到性能，可以先进行指针判等，如果对象是同一个那么直接返回 true；
  - 需要对另一方进行判空，空对象和自身进行比较，结果一定是 fasle（if (this == o) return true）；
  - 需要判断两个对象的类型，如果类型都不同，那么直接返回 false；
  - 确保类型相同的情况下再进行类型强制转换，然后逐一判断所有字段。

- hashCode 和 equals 要配对实现

  - 散列表需要使用 hashCode 来定位元素放到哪个桶。如果自定义对象没有实现自定义的 hashCode 方法，就会使用 Object 超类的默认实现，得到的两个hashCode 是不同的，导致无法满足需求。

- 注意 compareTo 和 equals 的逻辑一致性

  - binarySearch 方法内部调用了元素的 compareTo 方法进行比较；
  - 修复方式很简单，确保 compareTo 的比较逻辑和 equals 的实现一致即可。通过 Comparator.comparing 这个便捷的方法来实现两个字段的比较
  - 其实，这个问题容易被忽略的原因在于两方面：
    - 我们使用了 Lombok 的 @Data 标记了 Student,其实包含了 @EqualsAndHashCode 注解的作用，也就是默认情况下使用类型所有的字段（不包括 static 和 transient 字段）参与到 equals 和 hashCode 方法的实现中。因为这两个方法的实现不是我们自己实现的，所以容易忽略其逻辑。
    - compareTo 方法需要返回数值，作为排序的依据，容易让人使用数值类型的字段随意实现
  - 对于自定义的类型，如果要实现 Comparable，请记得 equals、hashCode、compareTo 三者逻辑一致。

- 小心 Lombok 生成代码的“坑”

  - 使用 @EqualsAndHashCode.Exclude 注解来修饰 name 字段，从 equals 和 hashCode的实现中排除 name 字段

  - 为解决这个问题，我们可以手动设置 callSuper 开关为 true，来覆盖这种默认行为

  - ```java
    @Data 
    @EqualsAndHashCode(callSuper = true) 
    class Employee extends Person
    ```

- 在实现 equals 时，我是先通过 getClass 方法判断两个对象的类型，你可能会想到还可以使用 instanceof 来判断。你能说说这两种实现方式的区别吗

  - getclass需要具体一种类型才能做比较
  - instanceof 涉及到继承的子类是都属于父类的判断

- 可以通过 HashSet 的 contains 方法判断元素是否在HashSet 中，同样是 Set 的 TreeSet 其 contains 方法和 HashSet 有什么区别吗

  - HashSet就是使用HashMap调用equals，判断两对象的HashCode是否相等
  - TreeSet因为是一个树形结构，则需要考虑树的左右。则需要通过compareTo计算正负值，看最后能否找到compareTo为0的值，找到则返回true

- quals比较的对象除了所谓的相等外，还有一个非常重要的因素，就是该对象的类加载器也必须是同一个，不然equals返回的肯定是false

  - 重启后，两个对象相等，
    结果是true，但是修改了某些东西后，热加载（不用重启即可生效）后，再次执行equals，返回就是false，因为热加载使用的类加载器和程序正常启动的类加载器不同

  

  

  

## 数值计算

- “危险”的 Double

  - 对 2.15-1.10 和 1.05 判等，结果判等不成立
    - 出现这种问题的主要原因是，计算机是以二进制存储数值的，浮点数也不例外
    - 对于计算机而言，0.1 无法精确表达，这是浮点数计算造成精度损失的根源
  - 我们大都听说过 BigDecimal 类型，浮点数精确表达和运算的场景，一定要使用这个类型。不过，在使用 BigDecimal 时有几个坑需要避开。
    - 浮点数运算避坑第一原则：使用 BigDecimal 表示和计算浮点数，且务必使用字符串的构造方法来初始化BigDecimal
    - 如果一定要用 Double 来初始化 BigDecimal 的话，可以使用 BigDecimal.valueOf 方法，以确保其表现和字符串形式的构造方法一致
    - BigDecimal 有 scale 和 precision 的概念，scale 表示小数点右边的位数，而 precision 表示精度，也就是有效数字的长度。
      - 调试一下可以发现，new BigDecimal(Double.toString(100)) 得到的 BigDecimal 的scale=1、precision=4；而 new BigDecimal(“100”) 得到的 BigDecimal 的 scale=0、precision=3。对于 BigDecimal 乘法操作，返回值的 scale 是两个数的 scale 相加。

- 考虑浮点数舍入和格式化的方式

  - String.format 采用四舍五入的方式进行舍入，取 1 位小数

    - 这就是由精度问题和舍入方式共同导致的，double 和 float 的 3.35 其实相当于 3.350xxx和 3.349xxx

    - 第二原则：浮点数的字符串格式化也要通过BigDecimal 进行。

      - ```java
        // 这次得到的结果是 3.3 和 3.4，符合预期
        BigDecimal num1 = new BigDecimal("3.35"); 
        BigDecimal num2 = num1.setScale(1, BigDecimal.ROUND_DOWN); 
        System.out.println(num2); 
        BigDecimal num3 = num1.setScale(1, BigDecimal.ROUND_HALF_UP); 
        System.out.println(num3);
        ```

- 用 equals 做判等，就一定是对的吗？

  - BigDecimal 的 equals 方法的注释中说明了原因，equals 比较的是 BigDecimal 的 value 和 scale，1.0 的 scale 是 1，1 的scale 是 0，所以结果一定是 false
  - 如果我们希望只比较 BigDecimal 的 value，可以使用 compareTo 方法
    - 你可能会意识到 BigDecimal 的 equals 和 hashCode 方法会同时考虑 value和 scale，如果结合 HashSet 或 HashMap 使用的话就可能会出现麻烦。比如，我们把值为 1.0 的 BigDecimal 加入 HashSet，然后判断其是否存在值为 1 的 BigDecimal，得到的结果是 false
    - 解决这个问题的办法有两个
      - 第一个方法是，使用 TreeSet 替换 HashSet。TreeSet 不使用 hashCode 方法，也不使用 equals 比较元素，而是使用 compareTo 方法，所以不会有问题	
      - 第二个方法是，把 BigDecimal 存入 HashSet 或 HashMap 前，先使用stripTrailingZeros 方法去掉尾部的零，比较的时候也去掉尾部的 0，确保 value 相同的BigDecimal，scale 也是一致的

- 小心数值溢出问题

  - 方法一是，考虑使用 Math 类的 addExact、subtractExact 等 xxExact 方法进行数值运算，这些方法可以在数值溢出时主动抛出异常。（执行后，可以得到 ArithmeticException，这是一个 RuntimeException）
  - 方法二是，使用大数类 BigInteger。BigDecimal 是处理浮点数的专家，而 BigInteger 则是对大数进行科学计算的专家。





## 集合类

- 使用 Arrays.asList 把数据转换为 List 的三个坑

  - 第一个坑是，不能直接使用 Arrays.asList 来转换基本类型数组

    - ```java
      int[] arr1 = {1, 2, 3};
      List list1 = Arrays.stream(arr1).boxed().collect(Collectors.toList());
      ```

  - 第二个坑，Arrays.asList 返回的 List 不支持增删操作

    - Arrays.asList 返回的 List 并不是我们期望的 java.util.ArrayList，而是 Arrays 的内部类 ArrayList。ArrayList 内部类继承自AbstractList 类，并没有覆写父类的 add 方法，而父类中 add 方法的实现，就是抛出UnsupportedOperationException

  - 第三个坑，对原始数组的修改会影响到我们获得的那个 List

    - 修复方式比较简单，重新 new 一个 ArrayList 初始化 Arrays.asList 返回的 List 即可

- 使用 List.subList 进行切片操作会导致 OOM

  - List.subList 返回的子List 不是一个普通的 ArrayList。这个子 List 可以认为是原始 List 的视图，会和原始 List 相互影响。如果不注意，很可能会因此产生 OOM 问题。
  - 既然 SubList 相当于原始 List 的视图，那么避免相互影响的修复方式有两种
    - 一种是，不直接使用 subList 方法返回的 SubList，而是重新使用 new ArrayList，在构造方法传入 SubList，来构建一个独立的 ArrayList；
    - 另一种是，对于 Java 8 使用 Stream 的 skip 和 limit API 来跳过流中的元素，以及限制流中元素的个数，同样可以达到 SubList 切片的目的。

- 一定要让合适的数据结构做合适的事情

  - 第一个误区是，使用数据结构不考虑平衡时间和空间。
    - ArrayList 在内存占用上性价比很高
    - 要对大 List 进行单值搜索的话，可以考虑使用 HashMap，其中 Key 是要搜索的值，Value 是原始对象，会比使用 ArrayList 有非常明显的性能优势。
  - 第二个误区是，过于迷信教科书的大 O 时间复杂度
    - 对于数组，随机元素访问的时间复杂度是 O(1)，元素插入操作是 O(n)； 
    - 对于链表，随机元素访问的时间复杂度是 O(n)，元素插入操作是 O(1)。 
    - 在大量的元素插入、很少的随机访问的业务场景下，
      - 在随机访问方面，我们看到了 ArrayList 的绝对优势，
      - 随机插入操作居然也是 LinkedList 落败
    - 翻看 LinkedList 源码发现，插入操作的时间复杂度是 O(1) 的前提是，你已经有了那个要插入节点的指针。但，在实现的时候，我们需要先通过循环获取到那个节点的 Node，然后再执行插入操作。前者也是有开销的，不可能只考虑插入操作本身的代价

- 与 ArrayList 在删除元素方面的坑有关

  - 调用类型是 Integer 的 ArrayList 的 remove 方法删除元素，传入一个 Integer 包装类的数字和传入一个 int 基本类型的数字，结果一样吗？
    - remove包装类数字是删除对象，基本类型的int数字是删除下标。
  

  
  
  
  
  
  

## 空值处理

- 空指针问题

  - NullPointerException 是 Java 代码中最常见的异常，我将其最可能出现的场景归为以下5 种：

    - 参数值是 Integer 等包装类型，使用时因为自动拆箱出现了空指针异常；

      - 对于 Integer 的判空，可以使用 Optional.ofNullable 来构造一个 Optional，然后使用orElse(0) 把 null 替换为默认值再进行 +1 操作。
      - Optional.ofNullable(i).orElse(0) + 1

    - 字符串比较出现空指针异常；

      - 对于 String 和字面量的比较，可以把字面量放在前面，比如"OK".equals(s)，这样即使s 是 null 也不会出现空指针异常；而对于两个可能为 null 的字符串变量的 equals 比较，可以使用 Objects.equals，它会做判空处理。
      - "OK".equals(s)
      - Objects.equals(s, t)

    - 诸如 ConcurrentHashMap 这样的容器不支持 Key 和 Value 为 null，强行 put null 的Key 或 Value 会出现空指针异常；

    - A 对象包含了 B，在通过 A 对象的字段获得 B 之后，没有对字段判空就级联调用 B 的方法出现空指针异常；

      - ```java
        // 对于类似 fooService.getBarService().bar().equals(“OK”) 的级联调用，需要判空的地方有很多，包括 fooService、getBarService() 方法的返回值，以及 bar 方法返回的字符串。如果使用 if-else 来判空的话可能需要好几行代码，但使用 Optional 的话一行代码就够了。        
        Optional.ofNullable(fooService)
                        .map(FooService::getBarService)
                        .filter(barService -> "OK".equals(barService.bar()))
                        .ifPresent(result -> log.info("OK"));
        ```

    - 方法或远程服务返回的 List 不是空而是 null，没有进行判空就直接调用 List 的方法出现空指针异常。

      - 对于 rightMethod 返回的 List，由于不能确认其是否为 null，所以在调用 size 方法获得列表大小之前，同样可以使用 Optional.ofNullable 包装一下返回值，然后通过.orElse(Collections.emptyList()) 实现在 List 为 null 的时候获得一个空的 List，最后再调用 size 方法。

      - ```java
        return Optional.ofNullable(rightMethod(test.charAt(0) == '1' ? null : new FooService(),
                        test.charAt(1) == '1' ? null : 1,
                        test.charAt(2) == '1' ? null : "OK",
                        test.charAt(3) == '1' ? null : "OK"))
                        .orElse(Collections.emptyList()).size();
        ```

  - 推荐使用阿里开源的 Java 故障诊断神器Arthas。Arthas 简单易用功能强大，可以定位出大多数的 Java 生产问题。

    - 通过 watch 命令监控 wrongMethod 方法的入参
    - watch 命令的参数包括类名表达式、方法表达式和观察表达式。这里，我们设置观察类为
      AvoidNullPointerExceptionController，观察方法为 wrongMethod，观察表达式为params 表示观察入参

- POJO 中属性的 null 到底代表了什么？

  - 明确 DTO 中null 的含义。

    - 对于 JSON 到 DTO 的反序列化过程，null 的表达是有歧义的，客户端不传某个属性，或者传 null，这个属性在 DTO 中都是 null

    - 传了null，意味着客户端希望重置这个属性。因为 Java 中的 null 就是没有这个数据，无法区
      分这两种表达，或许我们可以借助Optional 来解决这个问题。

    - UserDto 中只保留 id、name 和 age 三个属性，且 name 和 age 使用 Optional 来包装，以区分客户端不传数据还是故意传 null。

      - ```Java
        userEntity.setName(user.getName().orElse(""));
        userEntity.setNickname("guest" + userEntity.getName());
        ```

  - POJO 中的字段有默认值。

    - 如果客户端不传值，就会赋值为默认值，导致创建时间也被更新到了数据库中

  - 注意字符串格式化时可能会把 null 值格式化为 null 字符串。

  - DTO 和 Entity 共用了一个 POJO

    - 对于用户昵称的设置是程序控制的，我们不应该把它们暴露在 DTO 中，否则很容易把客户端随意设置的值更新到数据库中。此外，创建时间最好让数据库设置为当前时间，不用程序控制，可以通过在字段上设置columnDefinition 来实现。
    - 使用 Hibernate 的 @DynamicUpdate 注解实现更新 SQL 的动态生成，实现只更新修改后的字段，不过需要先查询一次实体，让 Hibernate 可以“跟踪”实体属性的当前状态，以确保有效。
      - 对于MyBatis 框架如何实现类似的动态 SQL 功能，实现插入和修改 SQL 只包含 POJO 中的非空字段
      - if标签
    - 然后，由于 DTO 中已经巧妙使用了 Optional 来区分客户端不传值和传 null 值，那么业务逻辑实现上就可以按照客户端的意图来分别实现逻辑。如果不传值，那么 Optional 本身为null，直接跳过 Entity 字段的更新即可，这样动态生成的 SQL 就不会包含这个列；如果传了值，那么进一步判断传的是不是 null。

  - 数据库字段允许保存 null，会进一步增加出错的可能性和复杂度

    - 因为如果数据真正落地的时候也支持 NULL 的话，可能就有 NULL、空字符串和字符串 null 三种状态。
    - 在 UserEntity 的字段上使用 @Column 注解，把数据库字段 name、nickname、age和 createDate 都设置为 NOT NULL，并设置 createDate 的默认值为CURRENT_TIMESTAMP，由数据库来生成创建时间。

- 小心 MySQL 中有关 NULL 的三个坑

  - MySQL 中 sum 函数没统计到任何记录时，会返回 null 而不是 0，可以使用 IFNULL函数把 null 转换为 0；
  - MySQL 中 count 字段不统计 null 值，COUNT(*) 才是统计所有记录数量的正确方式。
  - MySQL 中 =NULL 并不是判断条件而是赋值，对 NULL 进行判断只能使用 IS NULL 或者 IS NOT NULL。

- ConcurrentHashMap 的 Key 和 Value 都不能为 null，而 HashMap 却可以，你知道这么设计的原因是什么吗？TreeMap、Hashtable 等 Map 的 Key 和 Value 是否支持null 呢？

  - hashmap计算hash值的时候return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);会进行判断，所以key最多只能一个为null。hashtable计算hash值的时候int hash = key.hashCode();直接取的hashcode，key为null报错

  - Doug给出的回答是：在ConcurrentMaps (ConcurrentHashMaps, ConcurrentSkipListMaps)这些考虑并发安全的容器中不允许null值的出现的主要原因是他可能会在并发的情况下带来难以容忍的二义性

    