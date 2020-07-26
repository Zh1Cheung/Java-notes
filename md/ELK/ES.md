## ElasticSearch主要功能

- 分布式搜索引擎
- 大数据实时分析引擎
- 分布式设计，方便支持水平扩展







## 索引，文档和 REST API

- 索引是文档的容器
  - es是面向文档的，文档是所有可搜索数据的最小单位
  - index体现了逻辑空间的概念 
    - Mapping定义字段类型
    - Setting定义不同的数据分布
- Shard体现了物理空间的概念，索引中的数据分散在Shard上





## 集群

- 每个节点上都保存了集群的状态，只有Master节点才能修改集群的状态信息
  - 集群状态 (Cluster State)，维护了一个集群中，必要的信息
    - 所有的节点信息
    - 所有的索引和其相关的Mapping与Setting 信息
    - 分片的路由信息
- Data Node
  - 可以保存数据的节点，叫做Data Node。负责保存分片数据。在数据扩展上起到了至关重要的作用
- Cordinating Node
  - 负责接受Client的请求，将请求分发到合适的节点，最终把结果汇集到一-起
  - 每个节点默认都起到了Coordinating Node的职责





## 分片

- 因为 ES 是个分布式的搜索引擎, 所以索引通常都会分解成不同部分, 而这些分布在不同节点的数据就是分片
- 主分片，用以解决数据水平扩展的问题。通过主分片，可以将数据分布到集群内的所有节点之上
  - 一个分片是一个运行的Lucene的实例
  - 主分片数在索引创建时指定，后续不允许修改，除非Reindex
  - 7.0开始，默认主分片数设置成1，解决了over-sharding的问题
    - 影响搜索结果的相关性打分，影响统计结果的准确性
    - 单个节点上过多的分片，导致资源浪费，影响性能
- 副本，用以解决数据高可用的问题。分片是主分片的拷贝
  - 副本分片数，可以动态题调整
  - 增加副本数，还可以在一定程度上提高服务的可用性(读 取的吞吐)

- 副本分片
  - 数据可⽤性
  - 提升系统的读取性能



## CRUD

- Bulk批量操作

  - ```json
    POST _bulk
    {"index":{"_index":"test","_id":"1"}}
    {"field1":"value1"}
    {"delete":{"_index":"test","_id":"2"}}
    {"create":{"_index":"test2","_id":"3"}}
    {"field1":"value3"}
    {"update":{"_index":"test"},"_id":"1"}
    {"doc":{"field2":"value2"}}
    ```

- > index->PUT
  >
  > ​     如果文档不存在，就索引新的文档。否则现有文档会被删除，新的文档被索引。版本信息+1
  >
  > create->PUT（指定ID） or POST(不指定ID，自动生成)
  >
  > read->GET
  >
  > update->POST
  >
  > delete->DELETE







## 倒排索引

- 包含两个部分
  - 单词词典
    - 记录所有文档的单词，记录单词到倒排索引的关联关系
  - 倒排列表
    - 文档ID
    - 词频TF
    - 位置
    - 偏移







## 分词

- Analyzer的组成

  - 分词器是专门处理分词的组件，由三部分组成：
    - Character Filter： 针对原始文本处理，例如去除html
    - Tokenizer： 按照规则切分为单词
    - Token Filter : 将切分的单词进行加工，小写，删除stopwords，增加同义词

- 内置分词器

  - 	standard Analyzer - 默认分词器，按词切分，小写处理
      	
      	Simple Analyzer – 按照非字母切分（符号被过滤），小写处理
      	
      	Stop Analyzer – 小写处理，停用词过滤（the，a，is）
      	
      	Whitespace Analyzer – 按照空格切分，不转小写
      	
      	Keyword Analyzer – 不分词，直接将输入当作输出
      	
      	Patter Analyzer – 正则表达式，默认 \W+ (非字符分隔)
      	
      	Language – 提供了30多种常见语言的分词器
      	
      	Cusromer Analyzer - 自定义分词器

- ```json
  PUT /artists/
  {
      "settings" : {
          "analysis" : {
              "analyzer" : {
                  "user_name_analyzer" : {
                      "tokenizer" : "whitespace",
                      "filter" : "pinyin_first_letter_and_full_pinyin_filter"
                  }
              },
              "filter" : {
                  "pinyin_first_letter_and_full_pinyin_filter" : {
                      "type" : "pinyin",
                      "keep_first_letter" : true,
                      "keep_full_pinyin" : false,
                      "keep_none_chinese" : true,
                      "keep_original" : false,
                      "limit_first_letter_length" : 16,
                      "lowercase" : true,
                      "trim_whitespace" : true,
                      "keep_none_chinese_in_first_letter" : true
                  }
              }
          }
      }
  }
  
  ```

  





## Search API

- 衡量搜索的相关性
  - Precision（查准率）：尽可能返回较少的无关文档	True Positive/全部返回的结果（True Positives and false Positives）
  - Recall（查全率）：尽量返回较多的相关文档	True Positive/所有应该返回的结果（True Positives and false Negtives）
  - Ranking：是否能够按照相关度进行排序
- 查询表达式：Match
- 短语搜索：Match Phrase
- Query String





## Dynamic Mapping

- Mapping中的字段一旦设定后，禁止直接修改。因为倒排索引生成后不允许直接修改。需要重新建立新的索引，做reindex操作。

- Dynamic Mappring

  - 在写入文档时候，如果索引不存在，会自动创建索引
  - 无需手动定义Mappings，es会自动根据文档信息，推算出字段的类型
  - 当类型如果设置不对时，会导致一些功能无法正常运行，例如Range查询

- 设定Null_value 需要对Null值实现搜索

  - 只有keyword类型支持设定Null_value

- Index Options

  - docs 记录doc id
  - freqs 
  - positions
  - offsets

- Copy to

  - copy_to将字段的数值拷贝到目标字段，实现类似_all的作用
  - copy_to的目标字段不出现在_source中

- 多字段类型

  - ```json
    "company": {
        "type": "text",
        "fields": {
            "keyword": {
                "type": "keyword",
                "ignore_above": 256
            }
        }
    }
    ```

- Exact Values：es中的keyword

  - Exact Values在索引时，不需要做特殊的分词处理

- Full Text：es中的text

- Index Template

  - 帮助你设定Mapping和Setting，并按照一定的规则，自动匹配到新创建的索引之上

  - 当一个索引被新创建时

    - 应用es默认的settings和mappings
    - 应用order数值低的index template中的设定
    - 应用order高的index Template中的设定，之前的设定会被覆盖
    - 应用创建索引时，用户指定的settings和mappings覆盖之前模板中的设定

  - ```json
    PUT /_template/template_test
    {
        "index_patterns" : ["test*"],
        "order" : 1,
        "settings" : {
        	"number_of_shards": 1,
            "number_of_replicas" : 2
        },
        "mappings" : {
        	"date_detection": false,
        	"numeric_detection": true
        }
    }
    ```

- Dynaminc Template

  - 根据es识别的数据类型，结合字段名称，来动态设定字段类型
    - 所有的字符串类型都设定成keyword，或者关闭keyword字段
    - is开头的字段都设置成boolean
    - long_开头的都设置成long类型
  - 根据path匹配
  - 根据Type和字段名





## 聚合分析

- 集合的分类
  - Bukcet Aggregation：一些列满足特定条件的文档的集合
    - Term&Range
  - Metric Aggregation：一些数学运算，可以对文档字段进行统计分析
  - Pipeline Aggregation：对其他的聚合结果进行二次聚合
  - Matrix Aggregation：支持对多个字段的操作并提供一个结果矩阵







## 基于词项和基于全文的搜索

- 基于词项特点

  - Term 是表达语意的最⼩单位。搜索和利⽤统计语⾔模型进⾏⾃然语⾔处理都需要处理 Term
  - 在 ES 中，Term 查询，对输⼊不做分词。会将输⼊作为⼀个整体，在倒排索引中查找准确的词项，并且使⽤相关度算分公式为每个包含该词项的⽂档进⾏相关度算分 – 例如“Apple Store”
  - 可以通过 Constant Score 将查询转换成⼀个 Filtering，避免算分，并利⽤缓存，提⾼性能

- 基于全文特点

  - 索引和搜索时都会进⾏分词，查询字符串先传递到⼀个合适的分词器，然后⽣成⼀个供查询的词项列表

  - 查询时候，先会对输⼊的查询进⾏分词，然后每个词项逐个进⾏底层的查询，最终将结果进⾏合并。并为每个⽂档⽣成⼀个算分。- 例如查 “Matrix reloaded”，会查到包括 Matrix 或者 reload的所有结果

    





## 结构化搜索

- 对结构化数据的搜索(⽇期，布尔类型和数字都是结构化的)

  - 结构化的⽂本可以做精确匹配或者部分匹配
  - 结构化结果只有“是”或“否”两个值

  

  



## 搜索的相关性算分

- 词频 TF

- 逆⽂档频率 IDF

  - Inverse Document Frequency ：简单说 = log(全部⽂档数/检索词出现过的⽂档总数)

- Lucene 中的 TF-IDF 评分公式

  - $$
    \begin{array}{c}
    \operatorname{score}(q, d)=\operatorname{coord}(q, d) * \text {queryNorm}(q) * \sum_{t \text { in } q}\left(t f(t \text { in } d) * \text { idf }(t)^{2} * \text { boost }(t) * \text { norm }(t, d)\right) \\
    \end{array}
    $$

  - > score(q,d)：相关性打分函数 q查询语句 d 匹配的文档
    > 累加公式t in q：t为分词后的词项
    > tf(t in d)：tf计算
    > idf(t)^2：idf计算
    > boost(t)：boost提升
    > norm(t,d)：field lenght norm计算，文档越短，相关性越高

- BM25
  - 和经典的TF-IDF相⽐，当 TF ⽆限增加时，BM 25算分会趋于⼀个数值	
  -  BM25还引入了平均文档长度的概念，单个文档长度对相关性的影响力与它和平均长度的比值有关系。
  - K默认值是1.2，数值越小，饱和度越高。
  - BM25的TF公式里，除了k外，引入另外两个参数：L和b。
    - L是文档长度与平均长度的比值。如果文档长度是平均长度的2倍，则L＝2。
    - 参数b的作用是设定L对评分的影响有多大。如果把b设置为0，则L完全失去对评分的影响力。b的值越大，L对总评分的影响力越大。此时，相似度最终的完整公式为：





## Query & Filtering 与多字符串多字段查询

- 复合查询： bool Query
  - must
    - 贡献算分
  - should
    - 贡献算分
  - must_not
    - 不贡献算分
  - filter
    - 不贡献算分





## 单字符串多字段查询

- Dis Max Query
  
  - 不应该将分数简单叠加，而是应该找到单个最佳匹配的字段的评分
- 最佳字段查询调优
  - 有⼀些情况下，同时匹配 title 和 body 字段的⽂档⽐只与⼀个字段匹配的⽂档的相关度更⾼
  - 但 disjunction max query 查询只会简单地使⽤单个最佳匹配语句的评分 _score 作为整体评分。怎么办？
- Tie Breaker
  - 获得最佳匹配语句的评分 _score. 
  - 将其他匹配语句的评分与 tie_breaker 相乘
  - 对以上评分求和并规范化

- Multi Match

  - 三种场景

    - 最佳字段（Best Fields）
    - 多数字段（Most Fields）
      - 无法使用Operator
      - 每个字段对于最终评分的贡献可以通过⾃定义值 boost 来控制
    - 混合字段（Cross Fields）
      - 可以在搜索时为单个字段提升权重

    





## Search Template 和 Index Alias 查询

- Search Template

  - 解耦程序 & 搜索 DSL

- Index Alias 实现零停机运维

  - ```json
    POST _aliases
    {
      "actions": [
        {
          "add": {
            "index": "movies-2019",
            "alias": "movies-latest"
          }
        }
      ]
    }
    
    POST movies-latest/_search
    {
      "query": {
        "match_all": {}
      }
    }
    ```

    



## Function Score Query 优化算分

- 可以在查询结束后，对每⼀个匹配的⽂档进⾏⼀系列的重新算分，根据新⽣成的分数进⾏排序。

- 提供了⼏种默认的计算分值的函数

  - Weight ：为每⼀个⽂档设置⼀个简单⽽不被规范化的权重
  - Field Value Factor：使⽤该数值来修改 _score，例如将 “热度”和“点赞数”作为算分的参考因素
  - Random Score：为每⼀个⽤户使⽤⼀个不同的，随机算分结果
  - 衰减函数： 以某个字段的值为标准，距离某个值越近，得分越⾼
  - Script Score：⾃定义脚本完全控制所需逻辑

- 引入factor

  - 新的算分 = ⽼的算分 * log( 1 + factor *投票数 )

  - Modifier 平滑曲线

  - Max Boost 可以将算分控制在⼀个最⼤值

  - Sum：算分与函数的和

  - ```json
    POST /blogs/_search
    {
      "query": {
        "function_score": {
          "query": {
            "multi_match": {
              "query":    "popularity",
              "fields": [ "title", "content" ]
            }
          },
          "field_value_factor": {
            "field": "votes",
            "modifier": "log1p" ,
            "factor": 0.1
          },
          "boost_mode": "sum",
          "max_boost": 3
        }
      }
    }
    ```

    





## Term & Phrase Suggester

- 搜索建议
  - 原理：将输⼊的⽂本分解为 Token，然后在索引的字典⾥查找相似的 Term 并返回
  - 根据不同的使⽤场景，Elasticsearch 设计了 4 种类别的 Suggesters
    - Term & Phrase Suggester
    - Complete & Context Suggester	
- ⼏种 Suggestion Mode	
  - Missing – 如索引中已经存在，就不提供建议，无法搜索到结果时，提供建议
  - Popular – 推荐出现频率更加⾼的词
  - Always – ⽆论是否存在，都提供建议

- Sorting by Frequency & Prefix Length
  - 默认按照 score 排序，也可以按照“frequency”
  - 默认⾸字⺟不⼀致就不会匹配推荐，但是如果将 prefix_length 设置为 0，就会为 hock 建议 rock

- Phrase Suggester

  - Phrase Suggester 在 Term Suggester 上增加了⼀些额外的逻辑
  - ⼀些参数
    - Suggest Mode ：missing, popular, always
    - Max Errors：最多可以拼错的 Terms 数
    - Confidence：限制返回结果数，默认为 1
    - direct_generator:候选生成器
      - phrase suggestion API接受关键字direct_generator下的生成器列表，该列表中的每个生成器在原始文本中均按术语被调用。
      - field	从中获取候选建议的字段。这是必需选项

  

  

  

## 自动补全与基于上下文的提示 

- Completion Suggester 
  - 提供了“⾃动完成” (Auto Complete) 的功能。⽤户每输⼊⼀个字符，就需要即时发送⼀个查询请求到后段查找匹配项。
  - 使⽤ Completion Suggester 的⼀些步骤
    - 定义 Mapping，使⽤ “completion” type
    - 索引数据
    - 运⾏ “suggest” 查询，得到搜索建议
- Context Suggester
  - Completion Suggester 的扩展
  - 可以在搜索中加⼊更多的上下⽂信息，例如，输⼊ “star”
    - 咖啡相关：建议 “Starbucks”
    - 电影相关：”star wars”
- 实现 Context Suggester 的具体步骤
  - 定制⼀个 Mapping
    - 增加contexts
      - type
      - name
  - 索引数据，并且为每个⽂档加⼊ Context 信息
  - 结合 Context 进⾏ Suggestion 查询
- 





## 跨集群搜索

- ```json
  PUT _cluster/settings
  {
    "persistent": {
      "cluster": {
        "remote": {
          "cluster0": {
            "seeds": [
              "127.0.0.1:9300"
            ],
            "transport.ping_schedule": "30s"
          },
          "cluster1": {
            "seeds": [
              "127.0.0.1:9301"
            ],
            "transport.compress": true,
            "skip_unavailable": true
          },
          "cluster2": {
            "seeds": [
              "127.0.0.1:9302"
            ]
          }
        }
      }
    }
  }
  ```







## ⽂档分布式存储

- ⽂档到分⽚的路由算法
  - shard = hash(_routing) % number_of_primary_shards	
  - Hash 算法确保⽂档均匀分散到分⽚中
  - 默认的 _routing 值是⽂档 id
  - 可以⾃⾏制定 routing数值，例如⽤相同国家的商品，都分配到指定的 shard
  - 设置 Index Settings 后， Primary 数，不能随意修改的根本原因







## 分⽚及其⽣命周期

- 倒排索引不可变性
  - 不可变性，带来了的好处如下
    - ⽆需考虑并发写⽂件的问题，避免了锁机制带来的性能问题
    - ⼀旦读⼊内核的⽂件系统缓存，便留在哪⾥。只要⽂件系统存有⾜够的空间，⼤部分请求就会直接请求内存，不会命中磁盘，提升了很⼤的性能
    - 缓存容易⽣成和维护 / 数据可以被压缩
  - 不可变更性，带来了的挑战：如果需要让⼀个新的⽂档可以被搜索，需要重建整个索引。
- Lucene Index
  - 在 Lucene 中，单个倒排索引⽂件被称为Segment。Segment 是⾃包含的，不可变更的。多个 Segments 汇总在⼀起，称为 Lucene 的Index，其对应的就是 ES 中的 Shard
  - 当有新⽂档写⼊时，会⽣成新 Segment，查询时会同时查询所有 Segments，并且对结果汇总。Lucene 中有⼀个⽂件，⽤来记录所有Segments 信息，叫做 Commit Point
  - 删除的⽂档信息，保存在“.del”⽂件中
- Refresh
  - 将 Index buffer 写⼊ Segment 的过程叫Refresh。Refresh 不执⾏ fsync 操作
  - Refresh 频率：默认 1 秒发⽣⼀次，可通过index.refresh_interval 配置。Refresh 后，数据就可以被搜索到了。这也是为什么Elasticsearch 被称为近实时搜索
  - 如果系统有⼤量的数据写⼊，那就会产⽣很多的 Segment
  - Index Buffer 被占满时，会触发 Refresh，默认值是 JVM 的 10%

- Transaction Log

  - Segment 写⼊磁盘的过程相对耗时，借助⽂件系统缓存，Refresh 时，先将Segment 写⼊缓存以开放查询

  - 为了保证数据不会丢失。所以在 Index ⽂档时，同时写 Transaction Log，⾼版本开始

    Transaction Log 默认落盘。每个分⽚有⼀个 Transaction Log

  - 在 ES Refresh 时，Index Buffer 被清空，Transaction log 不会清空

- Flush
  - 调⽤ Refresh，Index Buffer 清空并且 Refresh
  - 调⽤ fsync，将缓存中的 Segments写⼊磁盘
  - 清空（删除）Transaction Log
  - 默认 30 分钟调⽤⼀次
  - Transaction Log 满 （默认 512 MB）

- Merge
  - Segment 很多，需要被定期被合并
  - 减少 Segments / 删除已经删除的⽂档





## 分布式查询及相关性算分

- Query Then Fetch 潜在的问题
  - 性能问题
    - 每个分⽚上需要查的⽂档个数 = from + size
    - 最终协调节点需要处理：number_of_shard * ( from+size )
    - 深度分⻚
  - 相关性算分	
    - 每个分⽚都基于⾃⼰的分⽚上的数据进⾏相关度计算。这会导致打分偏离的情况，特别是数据量很少时。相关性算分在分⽚之间是相互独⽴。
    - 当⽂档总数很少的情况下，如果主分⽚⼤于 1，主分⽚数越多 ，相关性算分会越不准

- 解决算分不准的⽅法
  - 数据量不⼤的时候，可以将主分⽚数设置为 1 
  - 使⽤ DFS Query Then Fetch
    - 一般不建议使用





## 排序

- 排序的过程

  - 排序是针对字段原始内容进⾏的。 倒排索引⽆法发挥作⽤

  - Elasticsearch 有两种实现⽅法

    - Fielddata

    - Doc Values （列式存储，对 Text 类型⽆效）

      

  - |          | Doc Values                     | Field data                                    |
    | -------- | ------------------------------ | --------------------------------------------- |
    | 何时创建 | 索引时，和倒排索引⼀起创建     | 搜索时候动态创建                              |
    | 创建位置 | 磁盘⽂件                       | JVM Heap                                      |
    | 优点     | 避免⼤量内存占⽤               | 索引速度快，不占⽤额外的磁盘空间              |
    | 缺点     | 降低索引速度，占⽤额外磁盘空间 | ⽂档过多时，动态创建开销⼤，占⽤ 过多JVM Heap |

- 打开 text的 fielddata

  - 默认关闭，可以通过 Mapping 设置打开。修改设置后，即时⽣效，⽆需重建索引
  - 其他字段类型不⽀持，只⽀持对 Text 进⾏设定
  - 打开后，可以对 Text 字段进⾏排序。但是是对分词后的 term 排序，所以，结果往往⽆法满⾜预期，不建议使⽤

- 关闭 keyword的 doc values
  - 默认启⽤，可以通过 Mapping 设置关闭
  - 如果重新打开，需要重建索引
  - 明确不需要做排序及聚合分析时关闭







## 分⻚与遍历

- 深度分页

- Search After 避免深度分⻚的问题

  - 第⼀步搜索需要指定 sort，并且保证值是唯⼀的（可以通过加⼊ _id 保证唯⼀性）
    然后使⽤上⼀次，最后⼀个⽂档的 sort 值进⾏查询

  - ```json
    POST users/_search
    {
        "size": 1,
        "query": {
            "match_all": {}
        },
        "search_after":
            [
              10,
              "ZQ0vYGsBrR8X3IP75QqX"],
        "sort": [
            {"age": "desc"} ,
            {"_id": "asc"}    
        ]
    }
    
    ```

- Scroll API

  - 每次查询后，输⼊上⼀次的 Scroll Id

  - scroll=1m(保持游标查询窗口一分钟)

  - ```json
    POST /_search/scroll
    {
        "scroll" : "1m",
        "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAWAWbWdoQXR2d3ZUd2kzSThwVTh4bVE0QQ=="
    }
    ```





## 处理并发读写操作

- ES采用的是乐观并发控制

- ES 中的⽂档是不可变更的。如果你更新⼀个⽂档，会将就⽂档标记为删除，同时增加⼀个全新的⽂档。同时⽂档的 version 字段加 1 

  - > 内部版本控制
    > 	If_seq_no + If_primary_term
    > 使⽤外部版本(使⽤其他数据库作为主要数据存储) 
    > 	version + version_type=external







## Pipeline 聚合分析

- ⽀持对聚合分析的结果，再次进⾏聚合分析

- 通过buckets_path指定路径

  - ```json
    POST employees/_search
    {
      "size": 0,
      "aggs": {
        "jobs": {
          "terms": {
            "field": "job.keyword",
            "size": 10
          },
          "aggs": {
            "avg_salary": {
              "avg": {
                "field": "salary"
              }
            }
          }
        },
        "min_salary_by_job":{
          "min_bucket": {
            "buckets_path": "jobs>avg_salary"
          }
        }
      }
    }
    ```







## 聚合的作⽤范围及排序

- ES 聚合分析的默认作⽤范围是 query 的查询结果集
  - 同时 ES 还⽀持以下⽅式改变聚合的作⽤范围
    - Filter
      - 只对当前的子聚合语句生效
    - Post_Filter
      - 是对聚合分析后的⽂档进⾏再次过滤
    - Global
      - ⽆视 query，对全部⽂档进⾏统计

-  聚合的精准度问题
  - 数据量、实时性、精确度 只能满足两条







## Update By Query & Reindex API

- ⼀般在以下⼏种情况时，我们需要重建索引
  - 索引的 Mappings 发⽣变更：字段类型更改，分词器及字典更新
  - 索引的 Settings 发⽣变更：索引的主分⽚数发⽣改变
  - 集群内，集群间需要做数据迁移
- Elasticsearch 的内置提供的 API
  - Update By Query：在现有索引上重建
  - Reindex：在其他索引上重建索引







## Ingest Pipeline 与 Painless Script

- Ingest Node
  - 具有预处理数据的能⼒，可拦截 Index 或 Bulk API 的请求
  - 对数据进⾏转换，并重新返回给 Index 或 Bulk API
- Pipeline & Processor
  - Pipeline - 管道会对通过的数据（⽂档），按照顺序进⾏加⼯
  - Processor - Elasticsearch 对⼀些加⼯的⾏为进⾏了抽象包装
  - Pipeline就是一组Processors









## Hot & Warm 架构与 Shard Filtering

- Hot 节点（通常使用 SSD）：索引有不断有新文档写入。通常使用 SSD 

- Warm 节点（通常使用 HDD）：索引不存在新数据的写入；同时也不存在大量的数据查询

- 配置 Hot & Warm Architecture

  - 使用 Shard Filtering，步骤分为以下几步 

  - 标记节点 （Tagging）

    - node.attr.my_node_type=hot

  - 配置索引到 Hot Node 

    - ```json
      PUT logs-2019-06-27
      {
        "settings":{
          "number_of_shards":2,
          "number_of_replicas":0,
          "index.routing.allocation.require.my_node_type":"hot"
        }
      }
      
      ```

  - 配置索引到 Warm 节点

- Rack Awareness

  - 尽可能避免将同一个索引的主副分片同时分配在一个机架的节点上

  - node.attr.my_rack_id=rack1

  - ```josn
    PUT _cluster/settings
    {
      "persistent": {
        "cluster.routing.allocation.awareness.attributes": "my_rack_id"
      }
    }
    
    ```

- Shard Filtering 

  - “index.routing.allocation” – 分配索引到节点

  - > 设置 										                       分配索引到节点，节点的属性规则 
    > Index.routing.allocation.include.{attr} 	至少包含一个值 
    > Index.routing.allocation.exclude.{attr} 	不能包含任何一个值 
    > Index.routing.allocation.require.{attr} 	所有值都需要包含



