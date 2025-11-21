---
title: elasticSearch 高级查询语法Query
date: 2024-03-10 13:22:14
tags:
---

# **ES倒排索引**

当数据写入 ES 时，数据将会通过 分词 被切分为不同的 term，ES 将 term 与其对应的文档列表建立一种映射关系，这种结构就是 倒排索引。如下图所示：

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/5EB9470E077F45AE8CE1207F0F0AAE45/50922)

为了进一步提升索引的效率，ES 在 term 的基础上利用 term 的前缀或者后缀构建了 term index, 用于对 term 本身进行索引，ES 实际的索引结构如下图所示：

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/EA7D94D695C44052BDAA20BE52E71D18/50923)

这样当我们去搜索某个关键词时，ES 首先根据它的前缀或者后缀迅速缩小关键词的在 term dictionary 中的范围，大大减少了磁盘IO的次数。

- 单词词典（Term Dictionary) ：记录所有文档的单词，记录单词到倒排列表的关联关系

- - 常用字典数据结构：https://www.cnblogs.com/LBSer/p/4119841.html

- 倒排列表(Posting List)-记录了单词对应的文档结合，由倒排索引项组成

- 倒排索引项(Posting)：

- - 文档ID
  - 词频TF–该单词在文档中出现的次数，用于相关性评分
  - 位置(Position)-单词在文档中分词的位置。用于短语搜索（match phrase query)
  - 偏移(Offset)-记录单词的开始结束位置，实现高亮显示

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/6ECEB2E891C34BB097892ACBED30950A/50921)

Elasticsearch 的JSON文档中的每个字段，都有自己的倒排索引。

可以指定对某些字段不做索引：

- 优点︰节省存储空间
- 缺点: 字段无法被搜索

# **文档映射Mapping**

Mapping类似数据库中的schema的定义，作用如下：

- 定义索引中的字段的名称
- 定义字段的数据类型，例如字符串，数字，布尔等
- 字段，倒排索引的相关配置(Analyzer)

ES中Mapping映射可以分为动态映射和静态映射。

**动态映射：**

在关系数据库中，需要事先创建数据库，然后在该数据库下创建数据表，并创建表字段、类型、长度、主键等，最后才能基于表插入数据。而Elasticsearch中不需要定义Mapping映射（即关系型数据库的表、字段等），在文档写入Elasticsearch时，会根据文档字段自动识别类型，这种机制称之为动态映射。

**静态映射：**

静态映射是在Elasticsearch中也可以事先定义好映射，包含文档的各字段类型、分词器等，这种方式称之为静态映射。

动态映射（Dynamic Mapping）的机制，使得我们无需手动定义Mappings，Elasticsearch会自动根据文档信息，推算出字段的类型。但是有时候会推算的不对，例如地理位置信息。当类型如果设置不对时，会导致一些功能无法正常运行，例如Range查询

**Dynamic Mapping类型自动识别：**

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/F05D4F42AAE64AF4AEBC56C88733999C/49290)

示例

```
#删除原索引
DELETE /user

#创建文档(ES根据数据类型, 会自动创建映射)
PUT /user/_doc/1
{
  "name":"fox",
  "age":32,
  "address":"长沙麓谷"
}

#获取文档映射
GET /user/_mapping
```

​     

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/7908E48E58A5476A9FAF7F07934C3E2C/49288)

 思考：能否后期更改Mapping的字段类型？

两种情况：

- **新增加字段**

- - dynamic设为true时，一旦有新增字段的文档写入，Mapping 也同时被更新
  - dynamic设为false，Mapping 不会被更新，新增字段的数据无法被索引，但是信息会出现在_source中
  - dynamic设置成strict(严格控制策略)，文档写入失败，抛出异常

|               | true | false | strict |
| ------------- | ---- | ----- | ------ |
| 文档可索引    | yes  | yes   | no     |
| 字段可索引    | yes  | no    | no     |
| Mapping被更新 | yes  | no    | no     |

- **对已有字段，一旦已经有数据写入，就不再支持修改字段定义**

- - Lucene 实现的倒排索引，一旦生成后，就不允许修改
  - 如果希望改变字段类型，可以利用 reindex API，重建索引

原因：

- 如果修改了字段的数据类型，会导致已被索引的数据无法被搜索
- 但是如果是增加新的字段，就不会有这样的影响

测试

```
PUT /user
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": {
        "type": "text"
      },
      "address": {
        "type": "object",
        "dynamic": "true"
      }
    }
  }
}
# 插入文档报错，原因为age为新增字段,会抛出异常
PUT /user/_doc/1
{
  "name":"fox",
  "age":32,
  "address":{
    "province":"湖南",
    "city":"长沙"
  }
}
```

dynamic设置成strict，新增age字段导致文档插入失败

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/00FAF4D5599D40DFBEA73AE0407C05A3/49291)

修改dynamic后再次插入文档成功

```
#修改daynamic
PUT /user/_mapping
{
  "dynamic":true
}
```

**对已有字段的mapping修改**

具体方法：

   1）如果要推倒现有的映射, 你得重新建立一个静态索引		

   2）然后把之前索引里的数据导入到新的索引里		

   3）删除原创建的索引		

   4）为新索引起个别名, 为原索引名				

```
PUT /user2
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "address": {
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}

POST _reindex
{
"source": {
"index": "user"
},
"dest": {
"index": "user2"
}
}

DELETE /user

PUT /user2/_alias/user

GET /user
```

   注意: 通过这几个步骤就实现了索引的平滑过渡,并且是零停机

## **常用Mapping参数配置**

- index: 控制当前字段是否被索引，默认为true。如果设置为false，该字段不可被搜索

```
DELETE /user
PUT /user
{
  "mappings" : {
      "properties" : {
        "address" : {
          "type" : "text",
          "index": false
        },
        "age" : {
          "type" : "long"
        },
        "name" : {
          "type" : "text"
        }
      }
    }
}

PUT /user/_doc/1
{
  "name":"fox",
  "address":"广州白云山公园",
  "age":30
}

GET /user

GET /user/_search
{
  "query": {
    "match": {
      "address": "广州"
    }
  }
}
```

​         

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/4572842C21DC4028AA2331126B7E756F/49289)

- 有四种不同基本的index options配置，控制倒排索引记录的内容：

- - docs :  记录doc id
  - freqs：记录doc id 和term frequencies（词频）
  - positions: 记录doc id / term frequencies / term position
  - offsets:  doc id / term frequencies / term posistion / character offsets

text类型默认记录postions，其他默认为 docs。记录内容越多，占用存储空间越大

```
DELETE /user
PUT /user
{
  "mappings" : {
      "properties" : {
        "address" : {
          "type" : "text",
          "index_options": "offsets"
        },
        "age" : {
          "type" : "long"
        },
        "name" : {
          "type" : "text"
        }
      }
    }
}
```



- null_value: 需要对Null值进行搜索，只有keyword类型支持设计Null_Value

```
DELETE /user
PUT /user
{
  "mappings" : {
      "properties" : {
        "address" : {
          "type" : "keyword",
          "null_value": "NULL"
        },
        "age" : {
          "type" : "long"
        },
        "name" : {
          "type" : "text"
        }
      }
    }
}

PUT /user/_doc/1
{
  "name":"fox",
  "age":32,
  "address":null
}

GET /user/_search
{
  "query": {
    "match": {
      "address": "NULL"
    }
  }
}

```

​          

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/C9BE6F291156404BB6C6C5A130EB1C14/49292)

- copy_to设置：将字段的数值拷贝到目标字段，满足一些特定的搜索需求。copy_to的目标字段不出现在_source中。

```
# 设置copy_to
DELETE /address
PUT /address
{
  "mappings" : {
      "properties" : {
        "province" : {
          "type" : "keyword",
          "copy_to": "full_address"
        },
        "city" : {
          "type" : "text",
          "copy_to": "full_address"
        }
      }
    },
    "settings" : {
        "index" : {
            "analysis.analyzer.default.type": "ik_max_word"
        }
    }
}

PUT /address/_bulk
{ "index": { "_id": "1"} }
{"province": "湖南","city": "长沙"}
{ "index": { "_id": "2"} }
{"province": "湖南","city": "常德"}
{ "index": { "_id": "3"} }
{"province": "广东","city": "广州"}
{ "index": { "_id": "4"} }
{"province": "湖南","city": "邵阳"}

GET /address/_search
{
  "query": {
    "match": {
      "full_address": {
        "query": "湖南常德",
        "operator": "and"
      }
    }
  }
}
```

## **Index Template**

Index Templates可以帮助你设定Mappings和Settings，并按照一定的规则，自动匹配到新创建的索引之上

- 模版仅在一个索引被新创建时，才会产生作用。修改模版不会影响已创建的索引
- 你可以设定多个索引模版，这些设置会被“merge”在一起
- 你可以指定“order”的数值，控制“merging”的过程

```
PUT /_template/template_default
{
  "index_patterns": ["*"],
  "order": 0,
  "version": 1,
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}


PUT /_template/template_test
{
  "index_patterns": ["test*"],
  "order": 1,
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  },
  "mappings": {
    "date_detection": false,
    "numeric_detection": true
  }
}
```

​      

**lndex Template的工作方式**

当一个索引被新创建时：

- 应用Elasticsearch 默认的settings 和mappings 
- 应用order数值低的lndex Template 中的设定
- 应用order高的 Index Template 中的设定，之前的设定会被覆盖
- 应用创建索引时，用户所指定的Settings和 Mappings，并覆盖之前模版中的设定

```
#查看template信息
GET /_template/template_default
GET /_template/temp*

PUT /testtemplate/_doc/1
{
  "orderNo": 1,
  "createDate": "2022/01/01"
}
GET /testtemplate/_mapping
GET /testtemplate/_settings

PUT /testmy
{
  "mappings": {
    "date_detection": true
  }
}

PUT /testmy/_doc/1
{
  "orderNo": 1,
  "createDate": "2022/01/01"
}

GET /testmy/_mapping
```

​     

## **Dynamic Template**

Dynamic Tempate定义在某个索引的Mapping中。

```
#Dynaminc Mapping 根据类型和字段名
DELETE my_index
PUT my_index/_doc/1
{
  "firstName":"Ruan",
  "isVIP":"true"
}

GET my_index/_mapping
DELETE my_index
PUT my_index
{
  "mappings": {
    "dynamic_templates": [
            {
        "strings_as_boolean": {
          "match_mapping_type":   "string",
          "match":"is*",
          "mapping": {
            "type": "boolean"
          }
        }
      },
      {
        "strings_as_keywords": {
          "match_mapping_type":   "string",
          "mapping": {
            "type": "keyword"
          }
        }
      }
    ]
  }
}

#结合路径
PUT /my_test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "full_name":{
          "path_match": "name.*",
          "path_unmatch": "*.middle",
          "mapping":{
            "type": "text",
            "copy_to": "full_name"
          }
        }
      }
      ]
  }
}

PUT /my_test_index/_doc/1
{
  "name":{
    "first": "John",
    "middle": "Winston",
    "last": "Lennon"
  }
}


GET /my_test_index/_search
{
  "query": {
    "match": {
      "full_name": "John"
    }
  }
}
```

​       

# **ES高级查询Query DSL**

ES中提供了一种强大的检索数据方式,这种检索方式称之为Query DSL（Domain Specified Language） , Query DSL是利用Rest API传递JSON格式的请求体(RequestBody)数据与ES进行交互，这种方式的丰富查询语法让ES检索变得更强大，更简洁。

[ https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl.html]( https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl.html)

语法:

```
GET /es_db/_doc/_search {json请求体数据}
可以简化为下面写法
GET /es_db/_search {json请求体数据}
```

示例

```
#无条件查询，默认返回10条数据
GET /es_db/_search
{
"query":{
"match_all":{}
}
}
```

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/0D5EDFDB694E4BD2B229706AE1EF9D9D/47342)

示例数据

```
#指定ik分词器
PUT /es_db
{
  "settings" : {
      "index" : {
          "analysis.analyzer.default.type": "ik_max_word"
      }
  }
}

# 创建文档,指定id
PUT /es_db/_doc/1
{
"name": "张三",
"sex": 1,
"age": 25,
"address": "广州天河公园",
"remark": "java developer"
}
PUT /es_db/_doc/2
{
"name": "李四",
"sex": 1,
"age": 28,
"address": "广州荔湾大厦",
"remark": "java assistant"
}

PUT /es_db/_doc/3
{
"name": "王五",
"sex": 0,
"age": 26,
"address": "广州白云山公园",
"remark": "php developer"
}

PUT /es_db/_doc/4
{
"name": "赵六",
"sex": 0,
"age": 22,
"address": "长沙橘子洲",
"remark": "python assistant"
}

PUT /es_db/_doc/5
{
"name": "张龙",
"sex": 0,
"age": 19,
"address": "长沙麓谷企业广场",
"remark": "java architect assistant"
}	
	
PUT /es_db/_doc/6
{
"name": "赵虎",
"sex": 1,
"age": 32,
"address": "长沙麓谷兴工国际产业园",
"remark": "java architect"
}			
```

​           

## **查询所有match_all**

使用match_all，默认只会返回10条数据。

原因：_search查询默认采用的是分页查询，每页记录数size的默认值为10。如果想显示更多数据，指定size

```
GET /es_db/_search
等同于
GET /es_db/_search
{
"query":{
"match_all":{}
}
}
```

   

### **返回指定条数size**

size 关键字: 指定查询结果中返回指定条数。 默认返回值10条

```
GET /es_db/_search
等同于
GET /es_db/_search
{
"query":{
"match_all":{}
}
}
```

思考： size可以无限增加吗？

测试

```
GET /es_db/_search
{
  "query": {
    "match_all": {}
  },
  "size": 20000
}
```

出现异常：

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/9C62678DC109403EAA9A9634B87FA00E/47254)

异常原因：

1、查询结果的窗口太大，from + size的结果必须小于或等于10000，而当前查询结果的窗口为20000。

2、可以采用scroll api更高效的请求大量数据集。

3、查询结果的窗口的限制可以通过参数index.max_result_window进行设置。

```
PUT /es_db/_settings
{ 
  "index.max_result_window" :"20000"
}
#修改现有所有的索引，但新增的索引，还是默认的10000
PUT /_all/_settings
{ 
  "index.max_result_window" :"20000"
}

#查看所有索引中的index.max_result_window值
GET /_all/_settings/index.max_result_window
```

​         

注意：参数index.max_result_window主要用来限制单次查询满足查询条件的结果窗口的大小，窗口大小由from + size共同决定。不能简单理解成查询返回给调用方的数据量。这样做主要是为了限制[内存](https://so.csdn.net/so/search?q=内存&spm=1001.2101.3001.7020)的消耗。

比如：from为1000000，size为10，逻辑意义是从满足条件的数据中取1000000到（1000000 + 10）的记录。这时ES一定要先将（1000000 + 10）的记录（即result_window）加载到内存中，再进行分页取值的操作。尽管最后我们只取了10条数据返回给客户端，但ES进程执行查询操作的过程中确需要将（1000000 + 10）的记录都加载到内存中，可想而知对内存的消耗有多大。这也是ES中不推荐采用（from + size）方式进行深度分页的原因。

同理，from为0，size为1000000时，ES进程执行查询操作的过程中确需要将1000000 条记录都加载到内存中再返回给调用方，也会对ES内存造成很大压力。

### **分页查询form**

from 关键字: 用来指定起始返回位置，和size关键字连用可实现分页效果

```
GET /es_db/_search
{
  "query": {
    "match_all": {}
  },
  "size": 5,
  "from": 0
}
```

​         

### **深分页查询Scroll**

改动index.max_result_window参数值的大小，只能解决一时的问题，当索引的数据量持续增长时，在查询全量数据时还是会出现问题。而且会增加ES服务器内存大结果集消耗完的风险。最佳实践还是根据异常提示中的采用scroll api更高效的请求大量数据集。

```
#查询命令中新增scroll=1m,说明采用游标查询，保持游标查询窗口一分钟。
#这里由于测试数据量不够，所以size值设置为2。
#实际使用中为了减少游标查询的次数，可以将值适当增大，比如设置为1000。
GET /es_db/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "size":  2
}
```

  

查询结果：

除了返回前2条记录，还返回了一个游标ID值_scroll_id。

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/1246ECE42197421FB4C5857B4FF02E2D/47287)

采用游标id查询：

```
# scroll_id 的值就是上一个请求中返回的 _scroll_id 的值
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "FGluY2x1ZGVfY29udGV4dF91dWlkDXF1ZXJ5QW5kRmV0Y2gBFmNwcVdjblRxUzVhZXlicG9HeU02bWcAAAAAAABmzRY2YlV3Z0o5VVNTdWJobkE5Z3MtXzJB"
}
```

​      

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/3B3D4829A7B84AC5B9D71A66EBE9A34D/47295)

多次根据scroll_id游标查询，直到没有数据返回则结束查询。采用游标查询索引全量数据，更安全高效，限制了单次对内存的消耗。

### **指定字段排序sort**

注意：会让得分失效

```
GET /es_db/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "age": "desc"
    }
  ]
}

#排序，分页
GET /es_db/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "age": "desc"
    }
  ],
  "from": 10,
  "size": 5
}
```

  

### **返回指定字段_source**

_source 关键字: 是一个数组,在数组中用来指定展示那些字段

```
GET /es_db/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["name","address"]
}
```

​          

**match**

match在匹配时会对所查找的关键词进行分词，然后按分词匹配查找

match支持以下参数：

- query : 指定匹配的值
- operator : 匹配条件类型

- - and : 条件分词后都要匹配
  - or : 条件分词后有一个匹配即可(默认)

- minmum_should_match : 最低匹配度，即条件在倒排索引中最低的匹配度

```
#模糊匹配 match   分词后or的效果
GET /es_db/_search
{
  "query": {
    "match": {
      "address": "广州白云山公园"
    }
  }
}

# 分词后 and的效果
GET /es_db/_search
{
  "query": {
    "match": {
      "address": {
        "query": "广州白云山公园",
        "operator": "AND"
      }
    }
  }
}

```

​         

在match中的应用： 当operator参数设置为or时，minnum_should_match参数用来控制匹配的分词的最少数量。

```
# 最少匹配广州，公园两个词
GET /es_db/_search
{
  "query": {
    "match": {
      "address": {
        "query": "广州公园",
        "minimum_should_match": 2
      }
    }
  }
}
```

 

## **短语查询match_phrase**

match_phrase查询分析文本并根据分析的文本创建一个短语查询。match_phrase 会将检索关键词分词。match_phrase的分词结果必须在被检索字段的分词中都包含，而且顺序必须相同，而且默认必须都是连续的。

```
GET /es_db/_search
{
  "query": {
    "match_phrase": {
      "address": "广州白云山"
    }
  }
}
GET /es_db/_search
{
  "query": {
    "match_phrase": {
      "address": "广州白云"
    }
  }
}
```

​           

思考：为什么查询广州白云山有数据，广州白云没有数据？

​    ![0](https://note.youdao.com/yws/public/resource/924a9d435d78784455143b1dda4a874a/xmlnote/8CA0149CA6BC4136AE70F8211D0BB9BC/47467)

分析原因：

先查看广州白云山公园分词结果，可以知道广州和白云不是相邻的词条，中间会隔一个白云山，而match_phrase匹配的是相邻的词条，所以查询广州白云山有结果，但查询广州白云没有结果。

```
POST _analyze
{
    "analyzer":"ik_max_word",
    "text":"广州白云山"
}
#结果
{
  "tokens" : [
    {
      "token" : "广州",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "白云山",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "白云",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "云山",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 3
    }
  ]
}
```

​     

如何解决词条间隔的问题？可以借助slop参数，slop参数告诉match_phrase查询词条能够相隔多远时仍然将文档视为匹配。

```
#广州云山分词后相隔为2，可以匹配到结果
GET /es_db/_search
{
  "query": {
    "match_phrase": {
      "address": {
        "query": "广州云山",
        "slop": 2
      } 
    }
  }
}
```

​        

## **多字段查询multi_match**

可以根据字段类型，决定是否使用分词查询，得分最高的在前面

```
GET /es_db/_search
{
  "query": {
    "multi_match": {
      "query": "长沙张龙",
      "fields": [
        "address",
        "name"
      ]
    }
  }
}
```

​         

注意：字段类型分词,将查询条件分词之后进行查询，如果该字段不分词就会将查询条件作为整体进行查询。

## **query_string**

允许我们在单个查询字符串中指定AND | OR | NOT条件，同时也和 multi_match query 一样，支持多字段搜索。和match类似，但是match需要指定字段名，query_string是在所有字段中搜索，范围更广泛。

注意: 查询字段分词就将查询条件分词查询，查询字段不分词将查询条件不分词查询

- **未指定字段查询**

```
GET /es_db/_search
{
  "query": {
    "query_string": {
      "query": "张三 OR 橘子洲"
    }
  }
}
```



- **指定单个字段查询**

```
#Query String
GET /es_db/_search
{
  "query": {
    "query_string": {
      "default_field": "address",
      "query": "白云山 OR 橘子洲"
    }
  }
}
```

​         

- **指定多个字段查询**

```
GET /es_db/_search
{
  "query": {
    "query_string": {
      "fields": ["name","address"],
      "query": "张三 OR (广州 AND 王五)"
    }
  }
}
```

​    

### **simple_query_string**

类似Query String，但是会忽略错误的语法,同时只支持部分查询语法，不支持AND OR NOT，会当作字符串处理。支持部分逻辑：

- \+ 替代AND
- | 替代OR
- \- 替代NOT

```
#simple_query_string 默认的operator是OR
GET /es_db/_search
{
  "query": {
    "simple_query_string": {
      "fields": ["name","address"],
      "query": "广州公园",
      "default_operator": "AND"
    }
  }
}

GET /es_db/_search
{
  "query": {
    "simple_query_string": {
      "fields": ["name","address"],
      "query": "广州 + 公园"
    }
  }
}
```

​     

## **关键词查询Term**

Term用来使用关键词查询(精确匹配),还可以用来查询没有被进行分词的数据类型。Term是表达语意的最小单位，搜索和利用统计语言模型进行自然语言处理都需要处理Term。match在匹配时会对所查找的关键词进行分词，然后按分词匹配查找，而term会直接对关键词进行查找。一般模糊查找的时候，多用match，而精确查找时可以使用term。

- ES中默认使用分词器为标准分词器(StandardAnalyzer),标准分词器对于英文单词分词,对于中文单字分词。

- 在ES的Mapping Type 中 keyword , date ,integer, long , double , boolean or ip 这些类型不分词，只有text类型分词。

```
#关键字查询 term
# 思考： 查询广州白云是否有数据，为什么？
GET /es_db/_search
{
  "query":{
    "term": {
      "address": {
        "value": "广州白云"
      }
    }
  }
}

# 采用term精确查询, 查询字段映射类型为keyword
GET /es_db/_search
{
  "query":{
    "term": {
      "address.keyword": {
        "value": "广州白云山公园"
      }
    }
  }
}
```

​      

在ES中，Term查询，对输入不做分词。会将输入作为一个整体，在倒排索引中查找准确的词项，并且使用相关度算分公式为每个包含该词项的文档进行相关度算分。

```
PUT /product/_bulk
{"index":{"_id":1}}
{"productId":"xxx123","productName":"iPhone"}
{"index":{"_id":2}}
{"productId":"xxx111","productName":"iPad"}

# 思考： 查询iPhone可以查到数据吗？  
GET /product/_search
{
  "query":{
    "term": {
      "productName": {
        "value": "iPhone"
      }
    }
  }
}

GET /product/_analyze
{
  "analyzer":"standard",
  "text":"iPhone"
}

# 对于英文，可以考虑建立索引时忽略大小写
PUT /product
{
  "settings": {
    "analysis": {
      "normalizer": {
        "es_normalizer": {
          "filter": [
            "lowercase",
            "asciifolding"
          ],
          "type": "custom"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "productId": {
        "type": "text"
      },
      "productName": {
        "type": "keyword",
        "normalizer": "es_normalizer",
        "index": "true"
      }
    }
  }
}
```

可以通过 Constant Score 将查询转换成一个 Filtering，避免算分，并利用缓存，提高性能。

- 将Query 转成 Filter，忽略TF-IDF计算，避免相关性算分的开销
- Filter可以有效利用缓存

```
GET /es_db/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "address.keyword": "广州白云山公园"
        }
      }
    }
  }
}
```

​      

**ES中的结构化搜索**

结构化搜索(Structured search)是指对结构化数据的搜索。

**结构化数据：**

- 日期，布尔类型和数字都是结构化的

- 文本也可以是结构化的。

- - 如彩色笔可以有离散的颜色集合：红(red) 、绿(green、蓝(blue)
  - 一个博客可能被标记了标签，例如，分布式(distributed)和搜索(search)
  - 电商网站上的商品都有UPC(通用产品码Universal Product Code)或其他的唯一

标识，它们都需要遵从严格规定的、结构化的格式。

应用场景：对bool，日期，数字，结构化的文本可以利用term做精确匹配

```
GET /es_db/_search
{
  "query": {
    "term": {
      "age": {
        "value": 28
      }
    }
  }
}
```

​            

term处理多值字段，term查询是包含，不是等于

```
POST /employee/_bulk
{"index":{"_id":1}}
{"name":"小明","interest":["跑步","篮球"]}
{"index":{"_id":2}}
{"name":"小红","interest":["跳舞","画画"]}
{"index":{"_id":3}}
{"name":"小丽","interest":["跳舞","唱歌","跑步"]}

POST /employee/_search
{
  "query": {
    "term": {
      "interest.keyword": {
        "value": "跑步"
      }
    }
  }
}
```

   

## **前缀查询prefix**

它会对分词后的term进行前缀搜索。

- 它不会分析要搜索字符串，传入的前缀就是想要查找的前缀
- 默认状态下，前缀查询不做相关度分数计算，它只是将所有匹配的文档返回，然后赋予所有相关分数值为1。它的行为更像是一个过滤器而不是查询。两者实际的区别就是过滤器是可以被缓存的，而前缀查询不行。

prefix的原理：需要遍历所有倒排索引，并比较每个term是否已所指定的前缀开头。

```
GET /es_db/_search
{
  "query": {
    "prefix": {
      "address": {
        "value": "广州"
      }
    }
  }
}
```

## **通配符查询wildcard**

通配符查询：工作原理和prefix相同，只不过它不是只比较开头，它能支持更为复杂的匹配模式。

```
GET /es_db/_search
{
  "query": {
    "wildcard": {
      "address": {
        "value": "*白*"
      }
    }
  }
}
```

​    

## **范围查询range**

- range：范围关键字
- gte 大于等于
- lte  小于等于
- gt 大于
- lt 小于
- now 当前时间

```
POST /es_db/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 25,
        "lte": 28
      }
    }
  }
}
```

### **日期range**

```
DELETE /product
POST /product/_bulk
{"index":{"_id":1}}
{"price":100,"date":"2021-01-01","productId":"XHDK-1293"}
{"index":{"_id":2}}
{"price":200,"date":"2022-01-01","productId":"KDKE-5421"}

GET /product/_mapping

GET /product/_search
{
  "query": {
    "range": {
      "date": {
        "gte": "now-2y"
      }
    }
  }
}
```

   

## **多id查询ids**

ids 关键字 : 值为数组类型,用来根据一组id获取多个对应的文档

```
GET /es_db/_search
{
  "query": {
    "ids": {
      "values": [1,2]
    }
  }
}
```

## **模糊查询fuzzy**

在实际的搜索中，我们有时候会打错字，从而导致搜索不到。在Elasticsearch中，我们可以使用fuzziness属性来进行模糊查询，从而达到搜索有错别字的情形。

fuzzy 查询会用到两个很重要的参数，fuzziness，prefix_length

- fuzziness：表示输入的关键字通过几次操作可以转变成为ES库里面的对应field的字段

- - 操作是指：新增一个字符，删除一个字符，修改一个字符，每次操作可以记做编辑距离为1，
  - 如中文集团到中威集团编辑距离就是1，只需要修改一个字符；
  - 该参数默认值为0，即不开启模糊查询。
  - 如果fuzziness值在这里设置成2，会把编辑距离为2的东东集团也查出来。

- prefix_length：表示限制输入关键字和ES对应查询field的内容开头的第n个字符必须完全匹配，不允许错别字匹配

- - 如这里等于1，则表示开头的字必须匹配，不匹配则不返回
  - 默认值也是0
  - 加大prefix_length的值可以提高效率和准确率。

```
GET /es_db/_search
{
  "query": {
    "fuzzy": {
      "address": {
        "value": "白运山",
        "fuzziness": 1    
      }
    }
  }
}

GET /es_db/_search
{
  "query": {
    "match": {
      "address": {
        "query": "广洲",
        "fuzziness": 1
      }
    }
  }
}

```

注意: fuzzy 模糊查询 最大模糊错误 必须在0-2之间

- 搜索关键词长度为 2，不允许存在模糊
- 搜索关键词长度为3-5，允许1次模糊
- 搜索关键词长度大于5，允许最大2次模糊

## **高亮highlight**

highlight 关键字: 可以让符合条件的文档中的关键词高亮。

highlight相关属性：

- pre_tags 前缀标签
- post_tags 后缀标签
- tags_schema 设置为styled可以使用内置高亮样式
- require_field_match 多字段高亮需要设置为false

示例数据

```
#指定ik分词器
PUT /products
{
  "settings" : {
      "index" : {
          "analysis.analyzer.default.type": "ik_max_word"
      }
  }
}

PUT /products/_doc/1
{
  "proId" : "2",
  "name" : "牛仔男外套",
  "desc" : "牛仔外套男装春季衣服男春装夹克修身休闲男生潮牌工装潮流头号青年春秋棒球服男 7705浅蓝常规 XL",
  "timestamp" : 1576313264451,
  "createTime" : "2019-12-13 12:56:56"
}

PUT /products/_doc/2
{
  "proId" : "6",
  "name" : "HLA海澜之家牛仔裤男",
  "desc" : "HLA海澜之家牛仔裤男2019时尚有型舒适HKNAD3E109A 牛仔蓝(A9)175/82A(32)",
  "timestamp" : 1576314265571,
  "createTime" : "2019-12-18 15:56:56"
}
```

​        

测试

```
GET /products/_search
{
  "query": {
    "term": {
      "name": {
        "value": "牛仔"
      }
    }
  },
  "highlight": {
    "fields": {
      "*":{}
    }
  }
}

```

 

### **自定义高亮html标签**

可以在highlight中使用pre_tags和post_tags

```
GET /products/_search
{
  "query": {
    "term": {
      "name": {
        "value": "牛仔"
      }
    }
  },
  "highlight": {
    "post_tags": ["</span>"], 
    "pre_tags": ["<span style='color:red'>"],
    "fields": {
      "*":{}
    }
  }
}
```

​        

### **多字段高亮**

```
GET /products/_search
{
  "query": {
    "term": {
      "name": {
        "value": "牛仔"
      }
    }
  },
  "highlight": {
    "pre_tags": ["<font color='red'>"],
    "post_tags": ["<font/>"],
    "require_field_match": "false",
    "fields": {
      "name": {},
      "desc": {}
    }
  }
}
```

​      
