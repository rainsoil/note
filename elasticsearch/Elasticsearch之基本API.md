# Elasticsearch的基本API

## 3. 基本API

### 3.1 `_cat`

#### 3.1.1  `GET`  `_cat` 查看所有的节点

如: `http://ip:9200/_cat/nodes`, 可以看到:

```text
172.29.0.3 21 86 55 8.33 8.23 9.08 dilm * 2162cacd6d03
```

注: `*` 表示集群中的主节点

#### 3.1.2 `GET`  `_cat/health` 查看es的健康状况

访问: `http://ip:9200/_cat/health`,可以看到

```text
1604546520 03:22:00 docker-cluster green 1 1 4 4 0 0 0 0 - 100.0%
```

注意: `green` 表示健康值正常

#### 3.1.3 `GET` `_cat/master` 查看主节点

访问: `http://ip:9200/_cat/master` , 可以看到

```text
osm0iaKBT9qrxc5mOIsRDQ 172.29.0.3 172.29.0.3 2162cacd6d03
```

#### 3.1.4  `GET` `_cat/indices`  查看所有的索引,等价于`mysql` 中的`show databases`

访问:  `http://ip:9200/_cat/indices`

可以看到

```text
green open .kibana_task_manager_1       SXTHIO_4RsWZmTwMNshnsA 1 0    2  0 49.2kb 49.2kb
green open kibana_sample_data_ecommerce caFiJ4VWSDmDkQFl6HXTRw 1 0 4675  0  5.1mb  5.1mb
green open .apm-agent-configuration     HPYrwMnOTB2s2vSeSMi_bQ 1 0    0  0   283b   283b
green open .kibana_1                    dIEll9o5TJWPNr95Bfv9pw 1 0   57 49  1.8mb  1.8mb
```



### 3.2  索引一个文档

保存一个数据,保存到哪个索引下面,指定用哪个唯一标识, 

如`PUT customer/external/1`, 表示 在`customer` 索引下的 `external` 类型下保存1号数据

```json
{
"name":"卢亚楠"
}
```

这里可以使用`POST` 请求, 也可以使用`PUT`请求,

`POST` 新增,如果不指定id, 会自动生成id, 指定id就会修改这个数据,并新增版本号

`PUT` 可以新增,也可以修改,但是`PUT` 必须指定id, 由于`PUT`必须指定id, 所以我们一般用来做更新操作,不指定`id` 就会报错

可以看到返回

```json
{
    "_index": "customer",
    "_type": "external",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

这些返回的`JSON`串的含义: 这些带有下划线开头的, 称为元数据,反映了当前的基本信息. 

`"_index": "customer"`: 表示该数据在哪个数据库下

`"_type": "external",` 表示该数据在哪个类型下

` "_id": "1",`:   说明被保存的数据的id

`"_version": 1,`:被保存的数据的版本

`"result": "created",`:  这里是创建了一条数据,如果 重新`put` 了一条数据, 该状态会变为`updated`, 并且版本号也会发生变化

###  3.3 查看文档

`GET ` `customer/external/1`

> `http://ip:9200/customer/external/1`

返回结果: 

```json
{
    "_index": "customer", // 在哪个索引中
    "_type": "external", // 在那个类型下
    "_id": "1", // 记录id
    "_version": 1, // 版本号
    "_seq_no": 0, // 并发控制字段,每次更新都会加1,用来做乐观锁
    "_primary_term": 1, // 同上,主分片重新分配,如重启,就会变化
    "found": true, 
    "_source": {
        "name": "卢亚楠"
    }
}
```

通过`if_seq_no=1&if_primary_term=1` ,当序列号匹配的时候,才进行修改,否则不修改. 

实例: 将`id=1`的数据, 修改为`name=1`,然后再次更新为`name=2`, 起始`seq_no=0`, `primary_term=1`

##### 1.  将`name`  更新为1

`http://ip:9200/customer/external/1?if_seq_no=0&if_primary_term=1`

查看返回结果

```json
{
    "_index": "customer",
    "_type": "external",
    "_id": "1",
    "_version": 2,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 1,
    "_primary_term": 1
}
```

可以看到, `_seq_no`加1了

##### 2.  将`name` 更新为2,更新过程中使用`_seq_no=0`

再次调用 `http://ip:9200/customer/external/1?if_seq_no=0&if_primary_term=1`

将参数更新为: 

```json
{
"name":"2"
}
```



返回结果: 

```json
{
    "error": {
        "root_cause": [
            {
                "type": "version_conflict_engine_exception",
                "reason": "[1]: version conflict, required seqNo [0], primary term [1]. current document has seqNo [1] and primary term [1]",
                "index_uuid": "wnn6v2lTQD23hl6ZTc2zpw",
                "shard": "0",
                "index": "customer"
            }
        ],
        "type": "version_conflict_engine_exception",
        "reason": "[1]: version conflict, required seqNo [0], primary term [1]. current document has seqNo [1] and primary term [1]",
        "index_uuid": "wnn6v2lTQD23hl6ZTc2zpw",
        "shard": "0",
        "index": "customer"
    },
    "status": 409
}
```

 可以看到更新出现了错误

##### 3. 查看新的数据

调用 `http://ip:9200/customer/external/1`

```json
{
    "_index": "customer",
    "_type": "external",
    "_id": "1",
    "_version": 2,
    "_seq_no": 1,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "name": "1"
    }
}
```

可以看到`_seq_no` 变成了1

##### 4. 再次更新,更新成功

将地址换为 `http://ip:9200/customer/external/1?if_seq_no=1&if_primary_term=1`, 

查看返回结果: 

```json
{
    "_index": "customer",
    "_type": "external",
    "_id": "1",
    "_version": 3,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 2,
    "_primary_term": 1
}
```

可以看到数据已经更新成功了

### 3.4 更新文档

可以使用

1. **`POST`** `customer/external/1/_update`

   ```json
   {
       "doc":{
           "name":"1"
       }
   }
   ```

2. **`POST`**   `customer/external/1`

   ```json
   {
           "name":"1"
       }
   ```

3. **`PUT`**  `customer/external/1`

   ```json
   {
           "name":"1"
       }
   ```

- 不同: `POST`操作会对比源文档数据,如果相同不会有什么操作,文档`Version 不增加,`PUT` 操作总会将数据重新保存并且增加`version` 版本

  带`_update` 对比元数据, 如果一样,则不进行任何操作. 

  使用场景: 

  - 对于大并发更新,不带`update`
  - 对于大并发查询偶尔更新,带`update`, 对比更新,重新计算分配规则

- 更新同时增加属性 **`POST`** `/customer/external/1_update`

  ```json
  {
      "doc":{
          "name":"2",
          "age":"22"
      }
  }
  ```

  

  

###  3.5 删除文档或者索引

  ```text
  DELETE customer/external/1
  DELETE customer
  ```

  注:  `elasticsearch` 并没有提供删除类型的操作, 只提供了删除索引和文档的操作,

  #### 3.5.1 根据id删除

  例:  删除`id=1`的数据,删除后继续查询`

  **`DELETE`**  `http://ip:9200/customer/external/1`

   返回结果: 

  ```json
  {
      "_index": "customer",
      "_type": "external",
      "_id": "1",
      "_version": 4,
      "result": "deleted",
      "_shards": {
          "total": 2,
          "successful": 1,
          "failed": 0
      },
      "_seq_no": 3,
      "_primary_term": 1
  }
  ```

  然后再查询, 可以看到返回

  ```josn
  {
      "_index": "customer",
      "_type": "external",
      "_id": "1",
      "found": false
  }
  ```

  

#### 3.5.2 删除整个索引

**`DELETE`** `http://ip:9200/customer`

返回结果: 

```json
{
    "acknowledged": true
}
```



### 3.6  批量操作 `bulk`

语法格式: 

```text
{action:{metadata}}\n
{request body  }\n
{action:{metadata}}\n
{request body  }\n
```

这里的批量操作,当发生某一条执行发生失败的时候,其他的数据仍然是可以继续执行,也就是说彼此之间是独立的,

`bulk api`依次按顺序执行所有的`action`(动作), 如果一个单个的动作因为任何原因失败,它将继续处理后面剩余的动作,当`bulk api`  返回的时候,它将提供每个动作的状态(与发送的顺序相同) , 通过这个可以检查一个指定的动作是否失败了. 

**例子**

1. 执行多条数据

   ```text
   POST customer/external/_bulk
   {"index":{"_id":"1"}}
   {"name":"John Doe"}
   {"index":{"_id":"2"}}
   {"name":"John Doe"}
   ```

   

    执行结果: 

   ```json
   #! Deprecation: [types removal] Specifying types in bulk requests is deprecated.
   {
     "took" : 491,
     "errors" : false,
     "items" : [
       {
         "index" : {
           "_index" : "customer",
           "_type" : "external",
           "_id" : "1",
           "_version" : 1,
           "result" : "created",
           "_shards" : {
             "total" : 2,
             "successful" : 1,
             "failed" : 0
           },
           "_seq_no" : 0,
           "_primary_term" : 1,
           "status" : 201
         }
       },
       {
         "index" : {
           "_index" : "customer",
           "_type" : "external",
           "_id" : "2",
           "_version" : 1,
           "result" : "created",
           "_shards" : {
             "total" : 2,
             "successful" : 1,
             "failed" : 0
           },
           "_seq_no" : 1,
           "_primary_term" : 1,
           "status" : 201
         }
       }
     ]
   }
   ```

2. 对于整个索引执行批量操作

   ```text
   POST /_bulk
   {"delete":{"_index":"website","_type":"blog","_id":"123"}}
   {"create":{"_index":"website","_type":"blog","_id":"123"}}
   {"title":"my first blog post"}
   {"index":{"_index":"website","_type":"blog"}}
   {"title":"my second blog post"}
   {"update":{"_index":"website","_type":"blog","_id":"123"}}
   {"doc":{"title":"my updated blog post"}}
   ```

   运行结果: 

   ```json
   #! Deprecation: [types removal] Specifying types in bulk requests is deprecated.
   {
     "took" : 608,
     "errors" : false,
     "items" : [
       {
         "delete" : {
           "_index" : "website",
           "_type" : "blog",
           "_id" : "123",
           "_version" : 1,
           "result" : "not_found",
           "_shards" : {
             "total" : 2,
             "successful" : 1,
             "failed" : 0
           },
           "_seq_no" : 0,
           "_primary_term" : 1,
           "status" : 404
         }
       },
       {
         "create" : {
           "_index" : "website",
           "_type" : "blog",
           "_id" : "123",
           "_version" : 2,
           "result" : "created",
           "_shards" : {
             "total" : 2,
             "successful" : 1,
             "failed" : 0
           },
           "_seq_no" : 1,
           "_primary_term" : 1,
           "status" : 201
         }
       },
       {
         "index" : {
           "_index" : "website",
           "_type" : "blog",
           "_id" : "MCOs0HEBHYK_MJXUyYIz",
           "_version" : 1,
           "result" : "created",
           "_shards" : {
             "total" : 2,
             "successful" : 1,
             "failed" : 0
           },
           "_seq_no" : 2,
           "_primary_term" : 1,
           "status" : 201
         }
       },
       {
         "update" : {
           "_index" : "website",
           "_type" : "blog",
           "_id" : "123",
           "_version" : 3,
           "result" : "updated",
           "_shards" : {
             "total" : 2,
             "successful" : 1,
             "failed" : 0
           },
           "_seq_no" : 3,
           "_primary_term" : 1,
           "status" : 200
         }
       }
     ]
   }
   ```

   
