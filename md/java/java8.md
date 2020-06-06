## 如何在项目中用上 Lambda 表达式和 Stream 操作？

- Lambda 表达式的初衷是，进一步简化匿名类的语法（不过实现上，Lambda 表达式并不是匿名类的语法糖），使 Java 走向函数式编程。

- 这里有个例子，分别使用匿名类和 Lambda 表达式创建一个线程打印字符串

  - ```java
    new Thread(new Runnable() {
    @Override
    	public void run() {
    		System.out.println(ss);
    }
    }).start();
    
    new Thread(() -> System.out.println("hello2")).start();
    ```

- Lambda 表达式如何匹配 Java 的类型系统呢？

  - 函数式接口是一种只有单一抽象方法的接口，使用 @FunctionalInterface 来描述，可以隐式地转换成 Lambda 表达式。使用 Lambda 表达式来实现函数式接口，不需要提供类名和方法定义，通过一行代码提供函数式接口的实例，就可以让函数成为程序中的头等公民，可以像普通数据一样作为参数传递，而不是作为一个固定的类中的固定方法。

  - ```java
    //Predicate接口是输入一个参数，返回布尔值。我们通过and方法组合两个Predicate条件
    Predicate<Integer> positiveNumber = i -> i > 0; 
    Predicate<Integer> evenNumber = i -> i % 2 == 0; 
    assertTrue(positiveNumber.and(evenNumber).test(2)); 
    
    //Consumer接口是消费一个数据。我们通过andThen方法组合调用两个Consumer，输出两行abcdefg 
    Consumer<String> println = System.out::println; 
    println.andThen(println).accept("abcdefg"); 
    
    //Function接口是输入一个数据，计算后输出一个数据。我们先把字符串转换为大写，然后通过andThen组合调用两个Function
    Function<String, String> upperCase = String::toUpperCase; 
    Function<String, String> duplicate = s -> s.concat(s); 
    assertThat(upperCase.andThen(duplicate).apply("test"), is("TESTTEST"));  
    
    //Supplier是提供一个数据的接口。这里我们实现获取一个随机数
    Supplier<Integer> random = ()->ThreadLocalRandom.current().nextInt(); 
    System.out.println(random.get()); 
    
    //BinaryOperator是输入两个同类型参数，输出一个同类型参数的接口。这里我们通过方法引用获得一个
    BinaryOperator<Integer> add = Integer::sum; 
    BinaryOperator<Integer> subtraction = (a, b) -> a - b; 
    assertThat(subtraction.apply(add.apply(1, 2), 3), is(0));
    ```

  - Predicate、Function 等函数式接口，还使用 default 关键字实现了几个默认方法。这样一来，它们既可以满足函数式接口只有一个抽象方法，又能为接口提供额外的功能

- 使用 Stream 简化集合操作

  - map 方法传入的是一个 Function，可以实现对象转换；
  - filter 方法传入一个 Predicate，实现对象的布尔判断，只保留返回 true 的数据；
  - mapToDouble 用于把对象转换为 double；
  - 通过 average 方法返回一个 OptionalDouble，代表可能包含值也可能不包含值的可空double。

- 使用 Optional 简化判空逻辑

  - 使用 Optional，不仅可以避免使用 Stream 进行级联调用的空指针问题；更重要的是，它提供了一些实用的方法帮我们避免判空逻辑。

- JDK8 结合 Lambda 和 Stream 对各种类的增强

  - 要通过 HashMap 实现一个缓存的操作，在 Java 8 之前我们可能会写出这样的getProductAndCache 方法：先判断缓存中是否有值；如果没有值，就从数据库搜索取值；最后，把数据加入缓存。

  - 而在 Java 8 中，我们利用 ConcurrentHashMap 的 computeIfAbsent 方法，用一行代码就可以实现这样的繁琐操作

  - ```java
    private Map<Long, Product> cache = new ConcurrentHashMap<>();
    private Product getProductAndCacheCool(Long id) { 
    	return cache.computeIfAbsent(id, i -> //当Key不存在的时候提供一个Function来代表
    		Product.getData().stream() // 拿入后存入cache
    			.filter(p -> p.getId().equals(i)) //过滤
    			.findFirst() //找第一个，得到Optional<Product> 
    			.orElse(null)); //如果找不到Product，则使用null 
    }
    ```

  - 并行流

    - 前面我们看到的 Stream 操作都是串行 Stream，操作只是在一个线程中执行，此外 Java 8还提供了并行流的功能：通过 parallel 方法，一键把 Stream 转换为并行操作提交到线程池处理。

      ```java
      IntStream.rangeClosed(1,100).parallel().forEach(xxxx)
      ```

- Stream 操作详解

  - flatMap:通过map把每一个元素转换为一个流，然后把所有流链接到一起扁平化展开

- 创建流

  - ```java
    // 通过 stream 方法把 List 或数组转换为流；
    Arrays.asList("a1", "a2", "a3").stream().forEach(System.out::println); 
    Arrays.stream(new int[]{1, 2, 3}).forEach(System.out::println);
    
    // 通过 Stream.of 方法直接传入多个元素构成一个流；
    String[] arr = {"a", "b", "c"};
    Stream.of(arr).forEach(System.out::println);
    Stream.of("a", "b", "c").forEach(System.out::println);
    Stream.of(1, 2, "a").map(item -> item.getClass().getName()).forEach(System.out::println);
    
    // 通过 Stream.iterate 方法使用迭代的方式构造一个无限流，然后使用 limit 限制流元素个数；
    Stream.iterate(2, item -> item * 2).limit(10).forEach(System.out::println);
    Stream.iterate(BigInteger.ZERO, n -> n.add(BigInteger.TEN)).limit(10).forEach(System.out::println);
    
    // 通过 Stream.generate 方法从外部传入一个提供元素的 Supplier 来构造无限流，然后使用 limit 限制流元素个数；
    Stream.generate(() -> "test").limit(3).forEach(System.out::println);
    Stream.generate(Math::random).limit(10).forEach(System.out::println);     
    
    // 通过 IntStream 或 DoubleStream 构造基本类型的流。
    IntStream.range(1, 3).forEach(System.out::println);
    IntStream.range(0, 3).mapToObj(i -> "x").forEach(System.out::println);
    IntStream.rangeClosed(1, 3).forEach(System.out::println);
    DoubleStream.of(1.1, 2.2, 3.3).forEach(System.out::println);
    
    //各种转换
    System.out.println(IntStream.of(1, 2).toArray().getClass()); //class [I
    System.out.println(Stream.of(1, 2).mapToInt(Integer::intValue).toArray().getClass()); //class [I
    System.out.println(IntStream.of(1, 2).boxed().toArray().getClass()); //class [Ljava.lang.Object;
    System.out.println(IntStream.of(1, 2).asDoubleStream().toArray().getClass()); //class [D
    System.out.println(IntStream.of(1, 2).asLongStream().toArray().getClass()); //class [J
    
    Arrays.asList("a", "b", "c").stream()   // Stream<String>
        .mapToInt(String::length)       // IntStream
        .asLongStream()                 // LongStream
        .mapToDouble(x -> x / 10.0)     // DoubleStream
        .boxed()                        // Stream<Double>
        .mapToLong(x -> 1L)             // LongStream
        .mapToObj(x -> "")              // Stream<String>
        .collect(Collectors.toList());
    }        
    ```

  - collect

    - toCollection 把流中的元素收集成指定的集合

    - joining 链接流中元素toString后的字符串

    - groupBy 根据元素的一个属性值对元素分组，属性值作为Key

    - partitionBy 用于分区，分区是特殊的分组，只有 true 和 false 两组。比如，我们把用户按照是否下单进行分区，给 partitioningBy 方法传入一个 Predicate 作为数据分区的区分，输出是 Map<Boolean, List>

      - ```java
        //根据是否有下单记录进行分区
        System.out.println(Customer.getData().stream().collect(
        	partitioningBy(customer -> orders.stream().mapToLong(Order::getCustomerId)
                                .anyMatch(id -> id == customer.getId()))));
        ```

  - 转化

    - ```java
      public class Main {
      
          public static void main(String[] args) {
      
              int[] data = {4, 5, 3, 6, 2, 5, 1};
              
              // int[] 转 List<Integer>
              List<Integer> list1 = Arrays.stream(data).boxed().collect(Collectors.toList());
      
              // Arrays.stream(arr) 可以替换成IntStream.of(arr)。
              // 1.使用Arrays.stream将int[]转换成IntStream。
              // 2.使用IntStream中的boxed()装箱。将IntStream转换成Stream<Integer>。
              // 3.使用Stream的collect()，将Stream<T>转换成List<T>，因此正是List<Integer>。
      
              // int[] 转 Integer[]
              Integer[] integers1 = Arrays.stream(data).boxed().toArray(Integer[]::new);
      
              // 前两步同上，此时是Stream<Integer>。
      
              // 然后使用Stream的toArray，传入IntFunction<A[]> generator。
      
              // 这样就可以返回Integer数组。
      
              // 不然默认是Object[]。
      
       
      
              // List<Integer> 转 Integer[]
      
              Integer[] integers2 = list1.toArray(new Integer[0]);
      
              //  调用toArray。传入参数T[] a。这种用法是目前推荐的。
      
              // List<String>转String[]也同理。
      
       
      
              // List<Integer> 转 int[]
      
              int[] arr1 = list1.stream().mapToInt(Integer::valueOf).toArray();
      
              // 想要转换成int[]类型，就得先转成IntStream。
      
              // 这里就通过mapToInt()把Stream<Integer>调用Integer::valueOf来转成IntStream
      
              // 而IntStream中默认toArray()转成int[]。
      
       
      
              // Integer[] 转 int[]
      
              int[] arr2 = Arrays.stream(integers1).mapToInt(Integer::valueOf).toArray();
      
              // 思路同上。先将Integer[]转成Stream<Integer>，再转成IntStream。
      
       
      
              // Integer[] 转 List<Integer>
      
              List<Integer> list2 = Arrays.asList(integers1);
      
              // 最简单的方式。String[]转List<String>也同理。
      
       
      
              // 同理
      
              String[] strings1 = {"a", "b", "c"};
      
              // String[] 转 List<String>
      
              List<String> list3 = Arrays.asList(strings1);
      
              // List<String> 转 String[]
      
              String[] strings2 = list3.toArray(new String[0]);
      
       
      
          }
      
      }
      
      ```

      