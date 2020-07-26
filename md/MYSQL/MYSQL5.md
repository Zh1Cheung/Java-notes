## 分库分表

- 一种常见的路由策略如下：

  - > １、中间变量　＝ user_id%（库数量*每个库的表数量）;
    >
    > ２、库序号　＝　取整（中间变量／每个库的表数量）;
    >
    > ３、表序号　＝　中间变量％每个库的表数量;





## 执行计划

- explain + 查询SQL - 用于显示SQL执行信息参数，根据参考信息可以进行SQL优化

- type

  - 查询时的访问方式，性能：all < index < range < index_merge < ref_or_null < ref < eq_ref < systemconst

  - ```sql
        ALL             全表扫描，对于数据表从头到尾找一遍
                        select * from tb1;
                        特别的：如果有limit限制，则找到之后就不在继续向下扫描
                               select * from tb1 where email = 'seven@live.com'
                               select * from tb1 where email = 'seven@live.com' limit 1;
                               虽然上述两个语句都会进行全表扫描，第二句使用了limit，则找到一个后就不再继续扫描。
    
        INDEX           全索引扫描，对索引从头到尾找一遍
                        select nid from tb1;
    
        RANGE          对索引列进行范围查找
                        select *  from tb1 where name < 'alex';
                        PS:
                            between and
                            in
                            >   >=  <   <=  操作
                            注意：!= 和 \> 符号
    
        INDEX_MERGE     合并索引，使用多个单列索引搜索
                        select *  from tb1 where name = 'alex' or nid in (11,22,33);
    
        REF             根据索引查找一个或多个值
                        select *  from tb1 where name = 'seven';
    
        EQ_REF          连接时使用primary key 或 unique类型
                        select tb2.nid,tb1.name from tb2 left join tb1 on tb2.nid = tb1.nid;
    
        CONST           常量
                        表最多有一个匹配行,因为仅有一行,在这行的列值可被优化器剩余部分认为是常数,const表很快,因为它们只读取一次。
                        select nid from tb1 where nid = 2 ;
    
        SYSTEM          系统
                        表仅有一行(=系统表)。这是const联接类型的一个特例。
                        select * from (select nid from tb1 where nid = 1) as A;
    ```





## BTree索引

- B-Tree
  - 数据结构
    - d为大于1的一个正整数，称为B-Tree的度。
    - h为一个正整数，称为B-Tree的高度。
    - 每个非叶子节点由n-1个key和n个指针组成，其中d<=n<=2d。
    - 每个叶子节点最少包含一个key和两个指针，最多包含2d-1个key和2d个指针，叶节点的指针均为null 。
    - 所有叶节点具有相同的深度，等于树高h。
    - key和指针互相间隔，节点两端是指针。
    - 一个节点中的key从左到右非递减排列。
- B+Tree
  - 内节点不存储data，只存储key；叶子节点不存储指针
  - 虽然B-Tree中不同节点存放的key和指针可能数量不一致，但是每个节点的域和上限是一致的，所以在实现中B-Tree往往对每个节点申请同等大小的空间
- 为什么使用B-Tree（B+Tree）
  - 局部性原理与磁盘预读
    - 当一个数据被用到时，其附近的数据也通常会马上被使用
    - 预读的长度一般为页（page）的整倍数
- B-/+Tree索引的性能分析
  - 一次检索最多需要h-1次I/O
    - 一般实际应用中，出度d是非常大的数字，通常超过100，因此h非常小（通常不超过3）
  - B+Tree内节点去掉了data域，因此可以拥有更大的出度，拥有更好的性能
- 使用自增字段作为主键则是一个很好的选择
  - 如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页





## MySQL语句执行顺序

- 写的顺序：select ... from... where.... group by... having... order by.. limit [offset,] 
- 顺序
  - from
  - on
  - join
  - where
  - group by
  - having
  - select
  - distinct
  - order by
  - limit

