# 4. Elasticsearch之检索

## 4. 检索

### 4.1 样本测试数据

准备一份顾客银行账户信息的虚构的`JSON`  文档样本,每个文档下面都用下列的`schema`(模式)

```json
{
	"account_number": 1,
	"balance": 39225,
	"firstname": "Amber",
	"lastname": "Duke",
	"age": 32,
	"gender": "M",
	"address": "880 Holmes Lane",
	"employer": "Pyrami",
	"email": "amberduke@pyrami.com",
	"city": "Brogan",
	"state": "IL"
}
```



从 `[https://github.com/elastic/elasticsearch/blob/master/docs/src/test/resources/accounts.json] `  导入测试数据

`POST bank/account/_bulk`



### 4.2 检索

#### 4.2.1 `search api`

`ES` 支持两种基本方式检索

- 通过`REST request uri`发送搜索参数(uri+搜索参数)
- 通过`REST request body`  来发送参数(uri+请求体)

信息检索

>  一切检索从`_search`开始

- `GET /bank/_search`    

  > 检索`bank` 下所有信息,包括`type`和`docs`

- `GET` `bank/_search?q=*&sort=account_number:asc` 

  >  请求参数方式检索

  返回结果: 

  ```json
  {
    "took" : 887,
    "timed_out" : false,
    "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
    },
    "hits" : {
      "total" : {
        "value" : 1000,
        "relation" : "eq"
      },
      "max_score" : null,
      "hits" : [
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "0",
          "_score" : null,
          "_source" : {
            "account_number" : 0,
            "balance" : 16623,
            "firstname" : "Bradshaw",
            "lastname" : "Mckenzie",
            "age" : 29,
            "gender" : "F",
            "address" : "244 Columbus Place",
            "employer" : "Euron",
            "email" : "bradshawmckenzie@euron.com",
            "city" : "Hobucken",
            "state" : "CO"
          },
          "sort" : [
            0
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "1",
          "_score" : null,
          "_source" : {
            "account_number" : 1,
            "balance" : 39225,
            "firstname" : "Amber",
            "lastname" : "Duke",
            "age" : 32,
            "gender" : "M",
            "address" : "880 Holmes Lane",
            "employer" : "Pyrami",
            "email" : "amberduke@pyrami.com",
            "city" : "Brogan",
            "state" : "IL"
          },
          "sort" : [
            1
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "2",
          "_score" : null,
          "_source" : {
            "account_number" : 2,
            "balance" : 28838,
            "firstname" : "Roberta",
            "lastname" : "Bender",
            "age" : 22,
            "gender" : "F",
            "address" : "560 Kingsway Place",
            "employer" : "Chillium",
            "email" : "robertabender@chillium.com",
            "city" : "Bennett",
            "state" : "LA"
          },
          "sort" : [
            2
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "3",
          "_score" : null,
          "_source" : {
            "account_number" : 3,
            "balance" : 44947,
            "firstname" : "Levine",
            "lastname" : "Burks",
            "age" : 26,
            "gender" : "F",
            "address" : "328 Wilson Avenue",
            "employer" : "Amtap",
            "email" : "levineburks@amtap.com",
            "city" : "Cochranville",
            "state" : "HI"
          },
          "sort" : [
            3
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "4",
          "_score" : null,
          "_source" : {
            "account_number" : 4,
            "balance" : 27658,
            "firstname" : "Rodriquez",
            "lastname" : "Flores",
            "age" : 31,
            "gender" : "F",
            "address" : "986 Wyckoff Avenue",
            "employer" : "Tourmania",
            "email" : "rodriquezflores@tourmania.com",
            "city" : "Eastvale",
            "state" : "HI"
          },
          "sort" : [
            4
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "5",
          "_score" : null,
          "_source" : {
            "account_number" : 5,
            "balance" : 29342,
            "firstname" : "Leola",
            "lastname" : "Stewart",
            "age" : 30,
            "gender" : "F",
            "address" : "311 Elm Place",
            "employer" : "Diginetic",
            "email" : "leolastewart@diginetic.com",
            "city" : "Fairview",
            "state" : "NJ"
          },
          "sort" : [
            5
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "6",
          "_score" : null,
          "_source" : {
            "account_number" : 6,
            "balance" : 5686,
            "firstname" : "Hattie",
            "lastname" : "Bond",
            "age" : 36,
            "gender" : "M",
            "address" : "671 Bristol Street",
            "employer" : "Netagy",
            "email" : "hattiebond@netagy.com",
            "city" : "Dante",
            "state" : "TN"
          },
          "sort" : [
            6
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "7",
          "_score" : null,
          "_source" : {
            "account_number" : 7,
            "balance" : 39121,
            "firstname" : "Levy",
            "lastname" : "Richard",
            "age" : 22,
            "gender" : "M",
            "address" : "820 Logan Street",
            "employer" : "Teraprene",
            "email" : "levyrichard@teraprene.com",
            "city" : "Shrewsbury",
            "state" : "MO"
          },
          "sort" : [
            7
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "8",
          "_score" : null,
          "_source" : {
            "account_number" : 8,
            "balance" : 48868,
            "firstname" : "Jan",
            "lastname" : "Burns",
            "age" : 35,
            "gender" : "M",
            "address" : "699 Visitation Place",
            "employer" : "Glasstep",
            "email" : "janburns@glasstep.com",
            "city" : "Wakulla",
            "state" : "AZ"
          },
          "sort" : [
            8
          ]
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "9",
          "_score" : null,
          "_source" : {
            "account_number" : 9,
            "balance" : 24776,
            "firstname" : "Opal",
            "lastname" : "Meadows",
            "age" : 39,
            "gender" : "M",
            "address" : "963 Neptune Avenue",
            "employer" : "Cedward",
            "email" : "opalmeadows@cedward.com",
            "city" : "Olney",
            "state" : "OH"
          },
          "sort" : [
            9
          ]
        }
      ]
    }
  }
  
  ```

  响应结果解释: 

  - `took`: `es` 执行搜索的时间(毫秒)
  - `time_out`: 告诉我们搜索是否超时
  - `_shards`: 告诉我们多少个分片被搜索了,以及统计了成功/失败的搜索分片
  - `hits`: 搜索结果
  - `hits.total`: 搜索结果总条数
  - `hits.hits`: 实际的搜索结果(数组), 默认为前10 的文档 
  - `sort`: 结果的排序key(键), 没有则按照 `scort` 排序
  - `score` 和`max_score` : 相关性得分和最高得分(全文检索用)

  详细的字段信息，参照： [https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-search.html](https:_www.elastic.co_guide_en_elasticsearch_reference_current_getting-started-search)



### 4.3 `Query DSL`

#### 4.3.1 基本语法格式

`Elasticsearch`  提供了一个可以执行查询的`json` 风格的`DSL`, 这个被称为`Query DSL`, 该查询语句非常全面,一个查询语句的典型结构: 

```json
QUERY_NAME:{
   ARGUMENT:VALUE,
   ARGUMENT:VALUE,...
}
```

如果针对某个字段,那么它的结构如下: 

```json
{
  QUERY_NAME:{
     FIELD_NAME:{
       ARGUMENT:VALUE,
       ARGUMENT:VALUE,...
      }   
   }
}
```

```json
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0,
  "size": 5,
  "sort": [
    {
      "account_number": {
        "order": "desc"
      }
    }
  ]
}
```

`query` 定义了如何查询

- `match_all` : 查询类型(代表查询所有的数据), `es`中可以在`query` 中组合非常多的查询类型来完成复杂查询
- 除 了`query` 参数之后,我们也可以传递其他的参数以改变查询的结果,比如`sort`,`size`
- `form`+`szie` 限定, 完成分页功能
- `sort`排序,多字段排序,会在前序字段相等时后续字段内部排序,否则以前序为准



#### 4.3.2  返回部分字段

```json
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0,
  "size": 5,
  "sort": [
    {
      "account_number": {
        "order": "desc"
      }
    }
  ],
  "_source": ["balance","firstname"]
  
}
```

返回结果: 

```json
{
  "took" : 221,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "999",
        "_score" : null,
        "_source" : {
          "firstname" : "Dorothy",
          "balance" : 6087
        },
        "sort" : [
          999
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "998",
        "_score" : null,
        "_source" : {
          "firstname" : "Letha",
          "balance" : 16869
        },
        "sort" : [
          998
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "997",
        "_score" : null,
        "_source" : {
          "firstname" : "Combs",
          "balance" : 25311
        },
        "sort" : [
          997
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "996",
        "_score" : null,
        "_source" : {
          "firstname" : "Andrews",
          "balance" : 17541
        },
        "sort" : [
          996
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "995",
        "_score" : null,
        "_source" : {
          "firstname" : "Phelps",
          "balance" : 21153
        },
        "sort" : [
          995
        ]
      }
    ]
  }
}

```



#### 4.3.3 `match` 匹配查询

- 基本类型(非字符串), 精切控制

```json
GET bank/_search
{
  "query": {
    "match": {
      "account_number": "20"
    }
  }
}
```

`match` 返回 account_number=20的数据

返回结果; 

```json
{
  "took" : 774,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "20",
        "_score" : 1.0,
        "_source" : {
          "account_number" : 20,
          "balance" : 16418,
          "firstname" : "Elinor",
          "lastname" : "Ratliff",
          "age" : 36,
          "gender" : "M",
          "address" : "282 Kings Place",
          "employer" : "Scentric",
          "email" : "elinorratliff@scentric.com",
          "city" : "Ribera",
          "state" : "WA"
        }
      }
    ]
  }
}

```

- 字符串: 全文检索

  ```json
  GET bank/_search
  {
    "query": {
      "match": {
        "address": "kings"
      }
    }
  }
  ```

  全文检索,最后会按照评分进行排序,会对检索条件进行分词匹配

  查询结果:

  ```json
  {
    "took" : 790,
    "timed_out" : false,
    "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
    },
    "hits" : {
      "total" : {
        "value" : 2,
        "relation" : "eq"
      },
      "max_score" : 6.095661,
      "hits" : [
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "20",
          "_score" : 6.095661,
          "_source" : {
            "account_number" : 20,
            "balance" : 16418,
            "firstname" : "Elinor",
            "lastname" : "Ratliff",
            "age" : 36,
            "gender" : "M",
            "address" : "282 Kings Place",
            "employer" : "Scentric",
            "email" : "elinorratliff@scentric.com",
            "city" : "Ribera",
            "state" : "WA"
          }
        },
        {
          "_index" : "bank",
          "_type" : "account",
          "_id" : "722",
          "_score" : 6.095661,
          "_source" : {
            "account_number" : 722,
            "balance" : 27256,
            "firstname" : "Roberts",
            "lastname" : "Beasley",
            "age" : 34,
            "gender" : "F",
            "address" : "305 Kings Hwy",
            "employer" : "Quintity",
            "email" : "robertsbeasley@quintity.com",
            "city" : "Hayden",
            "state" : "PA"
          }
        }
      ]
    }
  }
  
  ```

#### 4.3.4 `match_phrase` 短句匹配

将需要匹配的值当成一整个单词(不分词) 进行检索

```json
GET bank/_search
{
  "query": {
    "match_phrase": {
      "address": "mill road"
    }
  }
}
```

查看 `address` 中包含 `mill road`的所有记录, 并且给出相关性得分.

返回结果: 

```json
{
  "took" : 6278,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 8.991257,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 8.991257,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```

`match`  和`match_phrase` 的区别,观察如下例子: 

```json
GET bank/_search
{
  "query": {
    "match_phrase": {
      "address": "990 Mill"
    }
  }
}
```

返回结果:

```json
{
  "took" : 48,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 10.919691,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 10.919691,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```

使用`match` 的`keyword`

```json
GET bank/_search
{
  "query": {
    "match": {
      "address.keyword": "990 Mill"
    }
  }
}
```

查看结果, 一条也没有匹配到

```json
{
  "took" : 34,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

```

修改匹配条件为: `990 Mill Road`

```json
GET bank/_search
{
  "query": {
    "match": {
      "address.keyword": "990 Mill Road"
    }
  }
}
```

查询出来一条数据

```json
{
  "took" : 133,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 6.685111,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 6.685111,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```



文本字段的匹配,使用`keyword`, 匹配的条件就是要显示字段的全部值,要进行精确匹配,`match_phrase` 是做短语匹配的,只要是文本中包含匹配条件,就能匹配到. 



#### 4.3.5 `multi_math` 多字段匹配

```json
GET bank/_search
{
  "query": {
    "multi_match": {
      "query": "mill",
      "fields": [
        "state",
        "address"
      ]
    }
  }
}
```

`state` 或者`address` 中包含`mill`, 并且在查询的过程中 , 会对于查询条件进行分词. 

查询结果:

```json
{
  "took" : 121,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 5.4598455,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 5.4598455,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 5.4598455,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 5.4598455,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "472",
        "_score" : 5.4598455,
        "_source" : {
          "account_number" : 472,
          "balance" : 25571,
          "firstname" : "Lee",
          "lastname" : "Long",
          "age" : 32,
          "gender" : "F",
          "address" : "288 Mill Street",
          "employer" : "Comverges",
          "email" : "leelong@comverges.com",
          "city" : "Movico",
          "state" : "MT"
        }
      }
    ]
  }
}

```



#### 4.3.6  `bool`  用来做复合查询

复合语句可以合并,任何其他查询语句,包括复合语句,这也就是意味着复合语句之间可以互相嵌套,可以表达非常复杂的逻辑. 

##### `must`:  必须达到`must` 所列举的所有条件

```json
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        },
        {
          "match": {
            "gender": "M"
          }
        }
      ]
    }
  }
}
```

 查看`address=mill` 和`gender=M`的数据

返回结果:

```json
{
  "took" : 21,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 6.1390967,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      }
    ]
  }
}

```



#####  `must not` 必须不匹配`must not` 所列举的所有条件

查询`gender=M` 并且`address =mill`的,但是`age!=38`的数据

```json
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        },
        {
          "match": {
            "gender": "M"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "age": "30"
          }
        }
      ]
    }
  }
}
```

返回结果: 

```json
{
  "took" : 90,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 6.1390967,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      }
    ]
  }
}

```



#####  `should`  应该达到`should`   列举的条件,如果达到就会增加相关文档的评分, 并不会改变查询的结果, 如果`query` 中只有`should`   且只有一种匹配规则,那么`should`  的条件就会被作为默认匹配条件去改变查询结果

例子: 匹配`lastName` 应该等于`Wallace`的数据

```json
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        },
        {
          "match": {
            "gender": "M"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "age": "30"
          }
        }
      ],
      "should": [
        {
          "match": {
            "lastname": "Wallace"
          }
        }
      ]
    }
  }
}

```



返回结果: 

```json
{
  "took" : 18145,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 12.824207,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 12.824207,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 6.1390967,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      }
    ]
  }
}

```

能够看到相关度越高,得分也就越高

#### 4.3.7 `Filter`  结果过滤

并不是所有的查询都需要产生分数,特别是哪些仅用于`filter` 过滤的文档 . 为了不计算分数,`es` 会自动检查场景并且优化查询的执行

```json
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        }
      ],
      "filter": [
        {
          "range": {
            "balance": {
              "gte": 10000,
              "lte": 20000
            }
          }
        }
      ]
    }
  }
}

```

这里先查询所有匹配`address=mill`的文档,然后再根据`10000<=balance<=20000`进行结果查询, 查询结果:

```json
{
  "took" : 66,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 5.4598455,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 5.4598455,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```

Each `must`, `should`, and `must_not` element in a Boolean query is referred to as a query clause. How well a document meets the criteria in each `must` or `should` clause contributes to the document’s _relevance score_. The higher the score, the better the document matches your search criteria. By default, Elasticsearch returns documents ranked by these relevance scores.
在boolean查询中，`must`, `should` 和`must_not` 元素都被称为查询子句 。 文档是否符合每个“must”或“should”子句中的标准，决定了文档的“相关性得分”。  得分越高，文档越符合您的搜索条件。  默认情况下，Elasticsearch返回根据这些相关性得分排序的文档。
The criteria in a `must_not` clause is treated as a _filter_. It affects whether or not the document is included in the results, but does not contribute to how documents are scored. You can also explicitly specify arbitrary filters to include or exclude documents based on structured data.

`“must_not”子句中的条件被视为“过滤器”。` 它影响文档是否包含在结果中，  但不影响文档的评分方式。  还可以显式地指定任意过滤器来包含或排除基于结构化数据的文档。

`filter` 在使用过程中,并不会计算相关性得分

#### 4.3.8 `term`

和`match` 一样,匹配某个属性的值,全文检索字段用`match`, 其他非`text`字段匹配用`term`

> Avoid using the `term` query for [`text`](https:_www.elastic.co_guide_en_elasticsearch_reference_7.6_text) fields.
> 避免对文本字段使用“term”查询
> By default, Elasticsearch changes the values of `text` fields as part of [analysis](). This can make finding exact matches for `text` field values difficult.
> 默认情况下，Elasticsearch作为[analysis]()的一部分更改' text '字段的值。这使得为“text”字段值寻找精确匹配变得困难。
> To search `text` field values, use the match.
> 要搜索“text”字段值，请使用匹配。
> [https://www.elastic.co/guide/en/elasticsearch/reference/7.6/query-dsl-term-query.html](

使用`term` 匹配查询

```json
GET bank/_search
{
  "query": {
    "term": {
      "address": {
        "value": "mill Road"
      }
    }
  }
}
```

返回结果: 

```json
{
  "took" : 26,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

```

一条也没有匹配到, 更换为`match`后, 就可以匹配到32个

也就是说, 全文检索字段用`match`, 其他非`text` 字段匹配用`match`

#### 4.3.9 `Aggregation` 执行聚合

聚合提供了从数据中分组和提取数据的能力,最简单的聚合方法大致等于`SQL Group by`和`SQL`的聚合函数, 在`es`中,执行搜索返回`hits`( 命中结果), 并且同时返回聚合结果,把响应中所有的`hits`(命中结果)分隔开的能力, 这是非常强大并且有效的, 你可以执行查询和多个聚合,并且在一次使用中得到各自的(任何一个)返回结果,使用一次简介和简化的API 就可以避免网络往返. 

`size:0` 不显示搜索数据

`aggs` 执行聚合,聚合语法如下: 

```json
"aggs":{
    "aggs_name这次聚合的名字，方便展示在结果集中":{
        "AGG_TYPE聚合的类型(avg,term,terms)":{}
     }
}，
```

#####  例子: 搜索`address` 中包含`mill` 的所有人的年龄分布以及平均年龄,但是不显示这些人的详情

```json
GET bank/_search
{
  "query": {
    "match": {
      "address": "mill"
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
  },"size": 0
}


```

返回结果: 

```json
{
  "took" : 1229,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 38,
          "doc_count" : 2
        },
        {
          "key" : 28,
          "doc_count" : 1
        },
        {
          "key" : 32,
          "doc_count" : 1
        }
      ]
    },
    "ageAvg" : {
      "value" : 34.0
    },
    "balanceAvg" : {
      "value" : 25208.0
    }
  }
}

```

复杂的查询： 

##### 按照年龄聚合,并且求这些年龄段的人的平均薪资

```json
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 100
      },
      "aggs": {
        "ageAvg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  },
  "size": 0
}

```

返回结果: 

```json
{
  "took" : 3307,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 31,
          "doc_count" : 61,
          "ageAvg" : {
            "value" : 28312.918032786885
          }
        },
        {
          "key" : 39,
          "doc_count" : 60,
          "ageAvg" : {
            "value" : 25269.583333333332
          }
        },
        {
          "key" : 26,
          "doc_count" : 59,
          "ageAvg" : {
            "value" : 23194.813559322032
          }
        },
        {
          "key" : 32,
          "doc_count" : 52,
          "ageAvg" : {
            "value" : 23951.346153846152
          }
        },
        {
          "key" : 35,
          "doc_count" : 52,
          "ageAvg" : {
            "value" : 22136.69230769231
          }
        },
        {
          "key" : 36,
          "doc_count" : 52,
          "ageAvg" : {
            "value" : 22174.71153846154
          }
        },
        {
          "key" : 22,
          "doc_count" : 51,
          "ageAvg" : {
            "value" : 24731.07843137255
          }
        },
        {
          "key" : 28,
          "doc_count" : 51,
          "ageAvg" : {
            "value" : 28273.882352941175
          }
        },
        {
          "key" : 33,
          "doc_count" : 50,
          "ageAvg" : {
            "value" : 25093.94
          }
        },
        {
          "key" : 34,
          "doc_count" : 49,
          "ageAvg" : {
            "value" : 26809.95918367347
          }
        },
        {
          "key" : 30,
          "doc_count" : 47,
          "ageAvg" : {
            "value" : 22841.106382978724
          }
        },
        {
          "key" : 21,
          "doc_count" : 46,
          "ageAvg" : {
            "value" : 26981.434782608696
          }
        },
        {
          "key" : 40,
          "doc_count" : 45,
          "ageAvg" : {
            "value" : 27183.17777777778
          }
        },
        {
          "key" : 20,
          "doc_count" : 44,
          "ageAvg" : {
            "value" : 27741.227272727272
          }
        },
        {
          "key" : 23,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 27314.214285714286
          }
        },
        {
          "key" : 24,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 28519.04761904762
          }
        },
        {
          "key" : 25,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 27445.214285714286
          }
        },
        {
          "key" : 37,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 27022.261904761905
          }
        },
        {
          "key" : 27,
          "doc_count" : 39,
          "ageAvg" : {
            "value" : 21471.871794871793
          }
        },
        {
          "key" : 38,
          "doc_count" : 39,
          "ageAvg" : {
            "value" : 26187.17948717949
          }
        },
        {
          "key" : 29,
          "doc_count" : 35,
          "ageAvg" : {
            "value" : 29483.14285714286
          }
        }
      ]
    }
  }
}

```

#####  查询所有年龄分布,并且这些年龄段中M的平均薪资和F的平均薪资以及这个年龄段的总体平均薪资

```json
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 100
      },
      "aggs": {
        "genderAgg": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "balanceAvg": {
              "avg": {
                "field": "balance"
              }
            }
          }
        },
        "ageBalanceAvg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  },
  "size": 0
}
```

返回结果： 

```json
{
    "_shards": {
        "failed": 0,
        "skipped": 0,
        "successful": 1,
        "total": 1
    },
    "aggregations": {
        "ageAgg": {
            "buckets": [
                {
                    "ageBalanceAvg": {
                        "value": 28312.918032786885
                    },
                    "doc_count": 61,
                    "genderAgg": {
                        "buckets": [
                            {
                                "balanceAvg": {
                                    "value": 29565.628571428573
                                },
                                "doc_count": 35,
                                "key": "M"
                            },
                            {
                                "balanceAvg": {
                                    "value": 26626.576923076922
                                },
                                "doc_count": 26,
                                "key": "F"
                            }
                        ],
                        "doc_count_error_upper_bound": 0,
                        "sum_other_doc_count": 0
                    },
                    "key": 31
                },
                {
                    "ageBalanceAvg": {
                        "value": 25269.583333333332
                    },
                    "doc_count": 60,
                    "genderAgg": {
                        "buckets": [
                            {
                                "balanceAvg": {
                                    "value": 26348.684210526317
                                },
                                "doc_count": 38,
                                "key": "F"
                            },
                            {
                                "balanceAvg": {
                                    "value": 23405.68181818182
                                },
                                "doc_count": 22,
                                "key": "M"
                            }
                        ],
                        "doc_count_error_upper_bound": 0,
                        "sum_other_doc_count": 0
                    },
                    "key": 39
                },
                {
                    "ageBalanceAvg": {
                        "value": 23194.813559322032
                    },
                    "doc_count": 59,
                    "genderAgg": {
                        "buckets": [
                            {
                                "balanceAvg": {
                                    "value": 25094.78125
                                },
                                "doc_count": 32,
                                "key": "M"
                            },
                            {
                                "balanceAvg": {
                                    "value": 20943.0
                                },
                                "doc_count": 27,
                                "key": "F"
                            }
                        ],
                        "doc_count_error_upper_bound": 0,
                        "sum_other_doc_count": 0
                    },
                    "key": 26
                }
            ],
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0
        }
    },
    "hits": {
        "hits": [],
        "total": {
            "relation": "eq",
            "value": 1000
        }
    },
    "timed_out": false,
    "took": 34
}
```

