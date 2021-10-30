#  Springboot 整合Elasticsearch



##   1. `elasticsearch-Rest-Client`

### `elasticserch` 客户端详解

1. `9300`  `tcp`

    `spring-data-elasticsearch:transport-api.jar`

   - `springboot` 版本不同,`ransport-api.jar` 不同,不能适配`es` 版本
   - `7.x` 已经不建议使用了,8以后就要废弃了

2. `9200` `HTTP`

   - `jestClient`:非官方,更新慢

   - `RestTemplate`: 模拟`HTTP`请求, `ES` 很多操作都需要自己封装

   - `HttpClient`: 同上

   - `Elasticsearch-Rest-Client`: 官方`RestClient`，封装了`ES`操作, `API`层次分明,上手简单

     最终选择Elasticsearch-Rest-Client（elasticsearch-rest-high-level-client）；
     [https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html](https:_www.elastic.co_guide_en_elasticsearch_client_java-rest_current_java-rest-high)



## 2. `SpringBoot` 整合`ElasticSearch`

###  2.1 导入依赖

这里的版本要和所安装的`es`的版本匹配

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.6.2</version>
</dependency>
```

修改 `elasticsearch`的版本, `<spring-boot-dependencies>` 中所依赖的版本为6.x,需要将版本改为安装的版本



### 2.2 编写测试类

#### 2.2.1 测试保存数据

[https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-document-index.html](https:_www.elastic.co_guide_en_elasticsearch_client_java-rest_current_java-rest-high-document-index)

```java
 public static RestHighLevelClient restHighLevelClient() {
        return new RestHighLevelClient(RestClient.builder(new HttpHost("122.152.226.81", 9200, "http")));
    }


    @Test
    public void initData() throws IOException {
        IndexRequest request = new IndexRequest("user");

        User user = new User();
        user.setName("张三");
        user.setAge(22);
        user.setCreateTime(new Date());
        // 设置要保存的内容
        request.source(JSON.toJSONString(user), XContentType.JSON);
        //执行创建索引和保存数据
        IndexResponse index = restHighLevelClient().index(request, RequestOptions.DEFAULT);
        System.out.println(index);

    }
```

返回结果 

```json
IndexResponse[index=user,type=_doc,id=9kuBoHUBGfMAKZWDXKSG,version=1,result=created,seqNo=0,primaryTerm=1,shards={"total":2,"successful":1,"failed":0}]

```



#####  获取数据

```java
  /**
     * <p>获取数据</p>
     *
     * @author luyanan
     * @since 2020/11/7
     */
    @Test
    public void getData() throws IOException {

        // 9kuBoHUBGfMAKZWDXKSG 是上步添加操作的时候返回的id
        GetRequest request = new GetRequest("user", "9kuBoHUBGfMAKZWDXKSG");
        GetResponse response = restHighLevelClient().get(request, RequestOptions.DEFAULT);
        System.out.println(response);
        String id = response.getId();
        System.out.println("id:" + id);

        if (response.isExists()) {
            System.out.println("version:" + response.getVersion());
            System.out.println("返回结果:" + response.getSourceAsString());

        }
    }

```

返回结果

```java
{"_index":"user","_type":"_doc","_id":"9kuBoHUBGfMAKZWDXKSG","_version":1,"_seq_no":0,"_primary_term":1,"found":true,"_source":{"age":22,"createTime":1604715631647,"name":"张三"}}
id:9kuBoHUBGfMAKZWDXKSG
version:1
返回结果:{"age":22,"createTime":1604715631647,"name":"张三"}
```



##### 复杂查询

> 搜索`address` 中包含`mill`的所有人的年龄分布以及平均年龄, 平均薪资

```json
GET bank/_search
{
  "query": {
    "match": {
      "address": "Mill"
    }
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 10
      }
    },
    "ageAvg": {
      "avg": {
        "field": "age"
      }
    },
    "balanceAvg": {
      "avg": {
        "field": "balance"
      }
    }
  }
}
```

`java`的实现

```java
/**
     * <p>复杂检索:在bank中搜索address中包含mill的所有人的年龄分布以及平均年龄，平均薪资</p>
     *   GET bank/_search
     * {
     *   "query": {
     *     "match": {
     *       "address": "Mill"
     *     }
     *   },
     *   "aggs": {
     *     "ageAgg": {
     *       "terms": {
     *         "field": "age",
     *         "size": 10
     *       }
     *     },
     *     "ageAvg": {
     *       "avg": {
     *         "field": "age"
     *       }
     *     },
     *     "balanceAvg": {
     *       "avg": {
     *         "field": "balance"
     *       }
     *     }
     *   }
     * }
     * @author luyanan
     * @since 2020/11/7
     */
    @Test
    public void getData2() {
        //1. 创建检索请求
        SearchRequest searchRequest = new SearchRequest();
        //1.1）指定索引
        searchRequest.indices("bank");
        //1.2）构造检索条件
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.matchQuery("address", "Mill"));
        //1.2.1)按照年龄分布进行聚合
        TermsAggregationBuilder ageAgg = AggregationBuilders.terms("ageAgg").field("age").size(10);
        sourceBuilder.aggregation(ageAgg);
        //1.2.2)计算平均年龄
        AvgAggregationBuilder ageAvg = AggregationBuilders.avg("ageAvg").field("age");
        sourceBuilder.aggregation(ageAvg);
        //1.2.3)计算平均薪资
        AvgAggregationBuilder balanceAvg = AggregationBuilders.avg("balanceAvg").field("balance");
        sourceBuilder.aggregation(balanceAvg);
        System.out.println("检索条件：" + sourceBuilder);
        searchRequest.source(sourceBuilder);
        //2. 执行检索
        SearchResponse searchResponse = restHighLevelClient().search(searchRequest, RequestOptions.DEFAULT);
        System.out.println("检索结果：" + searchResponse);
        //3. 将检索结果封装为Bean
        SearchHits hits = searchResponse.getHits();
        SearchHit[] searchHits = hits.getHits();
        for (SearchHit searchHit : searchHits) {
            String sourceAsString = searchHit.getSourceAsString();
            Account account = JSON.parseObject(sourceAsString, Account.class);
            System.out.println(account);
        }
        //4. 获取聚合信息
        Aggregations aggregations = searchResponse.getAggregations();
        Terms ageAgg1 = aggregations.get("ageAgg");
        for (Terms.Bucket bucket : ageAgg1.getBuckets()) {
            String keyAsString = bucket.getKeyAsString();
            System.out.println("年龄：" + keyAsString + " ==> " + bucket.getDocCount());
        }
        Avg ageAvg1 = aggregations.get("ageAvg");
        System.out.println("平均年龄：" + ageAvg1.getValue());
        Avg balanceAvg1 = aggregations.get("balanceAvg");
        System.out.println("平均薪资：" + balanceAvg1.getValue());
    }

```

