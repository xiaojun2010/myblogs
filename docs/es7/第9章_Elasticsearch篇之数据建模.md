# 第9章 Elasticsearch 篇之数据建模

[TOC]



# 1 什么是数据建模

- 英文为Data Modeling，为创建数据模型的过程

- 数据模型（Data Model）

  > 对现实世界进行抽象描述的一种工具和方法
  >
  > 通过抽象的实体及实体之间联系的形式去描述业务规则，从而实现对现实世界的映射



# 2 数据建模的过程

- 概念模型

  > 确定系统的核心需求和范围边界，设计实体和实体间的关系

- 逻辑模型

  > 进一步梳理业务需求，确定每个实体的属性、关系和约束等

- 物理模型

  > 结合具体的数据库产品，在满足业务读写性能等需求的前提下确定最终的定义
  >
  > Mysql、MongoDB、elasticsearch等
  >
  > 第三范式



# 3 数据建模的意义

<img src="./img09/01.png" alt="image-20230819223106731" style="zoom:30%;"  />


一个好的数据模型允许系统更好的做集成，或者更简化的做一些对外的接口




# 4 ES中的数据建模

- ES是基于Lucene以**倒排索引**为基础实现的存储体系，不遵循关系型数据库中的范式约定

  <img src="img09/02.png" alt="image-20230819223728303" style="zoom:20%;"  />



# 5 mapping 字段的相关设置

- enabled

  >  true | false
  >
  > 仅存储，不做搜索或聚合分析

- index

  >  true | false
  >
  > 是否构建倒排索引

- index_options

  > docs | freqs | positions | offsets
  >
  > 存储倒排索引的哪些信息

- norms

  >  true | false
  >
  > 是否存储归一化相关参数，如果字段仅用于过滤和聚合分析，可关闭

- doc_values

  >  true | false
  >
  > 是否启用doc_values，用于排序和聚合分析

- field_data

  >  true | false
  >
  > 是否为text类型启用field_data，实现排序和聚合分析；可以动态实时开关

- store

  >  true | false
  >
  > 是否存储该字段

- coerce

  >  true | false
  >
  > 是否开启**自动数据类型转换**功能，比如字符串转为数字、浮点转为整型等

- multifields多字段

  > 灵活使用多字段特性来解决多样的业务需求

- dynamic

  >  true | false | strict
  >
  > 控制mapping自动更新

- data_detection

  >  true | false
  >
  > 是否自动识别日期类型，建议设置为false



# 6 mapping字段属性的设定流程

<img src="img09/03.png" alt="image-20230819223728303" style="zoom:100%;"  />





- **是何种类型：**
- **是否需要检索：**如果要检索，则index 设置为true，并设置index_options到何种粒度
- **是否需要排序和聚合分析：**如果不需要聚合分析，doc_values设置为false，节省存储空间
- **是否需要另行存储：**



## 6.1 是何种类型？

- 字符串类型

  > 需要分词则设定为text类型，否则设置为keyword类型

- 枚举类型

  > 基于性能考虑将其设定为keyword类型，即使该类型为整型

- 数值类型

  > 尽量选择贴近的类型，比如byte即可表示所有数值时，即选用byte，不要用long

- 其他类型

  > 比如bool类型、日期、地理位置数据等

## 6.2 是否需要检索

- 完全不需要检索、排序、聚合分析的字段

  - enabled 设置为false ，则不会构建索引，也不会使用doc_values，仅在查询数据时返回

- 不需要检索的字段

  - index 设置为false ，则不会构建索引

- 需要检索的字段，可以通过如下配置设定需要的存储粒度

  - index_options 结合需要设定

  - norms 不需要归一化数据时关闭即可

- 不需要排序或者聚合分析功能

  - doc_values 设定为false
  - fielddata 设定为false（默认就是false）

- 是否需要专门存储当前字段的数据？

  - store 设定为true，即可存储该字段的原始内容（与 _source 中的不相关），`如果store 为true，__source 为true 则数据存储了2份`
  - 一般结合 _source 的enabled 设定为false时使用

-

## 6.3 实例

- 博客文章blog_index

  - 标题 title
  - 发布日期 publish_date
  - 作者 author
  - 摘要 abstract
  - 网络地址 url

- blog_index 的mapping 设置如下：

  ![image-20230820004512798](img09/04.png)

  ```json
  PUT /blog_index
  {
    "mappings": {
      "properties": {
        "title":{
          "type": "text",
          "fields": {
            "keyword":{
              "type":"keyword",
              "ignore_above":100
            }
          }
        },
        "publist_date":{
          "type": "date"
        },
        "author":{
          "type": "keyword",
          "ignore_above":100
        },
        "abstract":{
          "type": "text"
        },
        "url":{
          "enabled":false
        }
      }
    }
  }
  
  PUT blog_index/_doc/1
  {
    "title":"blog title",
    "content":"blog content"
  }
  
  GET blog_index/_search
  
  GET blog_index/_search?_source=title
  ```




- 如果索引添加字段：博客文章blog_index

  - 标题 title

  - 发布日期 publish_date

  - 作者 author

  - 摘要 abstract

  - **<font color = 'red'>内容content</font>**

    content 字段内容比较大，如果 content 还是 text类型，则每次去取原始数据时，通过_source 获取原始内容时，需要取非常多的内容，则性能非常慢。

    ![image-20230820001328862](img09/05.png)

    ![image-20230820005143897](img09/07.png)

    ```json
    DELETE /blog_index

    PUT /blog_index
    {
      "mappings": {
        "_source": {"enabled": false},
        "properties": {
          "title":{
            "type": "text",
            "fields": {
              "keyword":{
                "type":"keyword",
                "ignore_above":100
              }
            },
            "store": true
          },
          "publist_date":{
            "type": "date",
            "store": true
          },
          "author":{
            "type": "keyword",
            "ignore_above":100,
            "store": true
          },
          "abstract":{
            "type": "text",
            "store": true
          },
          "content":{
            "type": "text",
            "store": true
          },
          "url":{
            "type": "keyword",
            "doc_values": false,
            "norms": false,
            "ignore_above": 100,
            "store": true
          }
        }
      }
    }

    PUT blog_index/_doc/1
    {
      "title":"blog title",
      "content":"blog content"
    }

    PUT blog_index/_doc/1
    {
      "title":"blog title",
      "content":"blog content"
    }

    GET blog_index/_search

    GET blog_index/_search
    {
      "stored_fields": ["title","publist_date","author","abstract","url"],
      "query": {
        "match": {
          "content": "content"
        }
      },
      "highlight": {
        "fields": {
          "content": {}
        }
      }
    }

    ```

    ![image-20230820003041837](img09/06.png)

  - 网络地址 url

# 7 关联关系处理

ES不擅长处理关系型数据库中的关联关系，比如文章表blog与评论表comment之间通过blog_id关联，在ES中可以通过如下2种手段变相解决:

- Nested object
- Parent / child

例子：

评论Comment

- 文章ID blog_id
- 评论人 username
- 评论日期 date
- 评论内容 content

| <img src="img09/08.png" alt="image-20230820010459363" style="zoom:20%;" /> | <img src="img09/09.png" alt="image-20230820010459363" style="zoom:60%;" /> |
| ------------------------------------------------------------ | ------------------------------------------------------------ |

## 7.1 关联关系处理-nested object

<img src="img09/10.png" alt="image-20230820010930623" style="zoom:30%;"  />

把comments 都整合到blog中，好处是获取blog时就可以获取评论

<img src="img09/11.png" alt="image-20230820011853430" style="zoom:30%;"    />

**但是查询不符合要求：**
要求：查询 username = lee and content 包含thanks的用户评论



Comments默认是Object Array，存储结构类似下面的形式：

<img src="img09/12.png" alt="image-20230820012052746" style="zoom:30%;"    />

这种结构查询就会有问题，Nested Object 可以解决这个问题

<img src="img09/13.png" alt="image-20230820012237672" style="zoom:30%;"   />

<img src="img09/14.png" alt="image-20230820012414967" style="zoom:30%;"    />

为什么nest object可以解决这个问题呢？

### 7.1.1 Nest Object Array的存储结构类似下面的形式

<img src="img09/15.png" alt="image-20230820012622752" style="zoom:30%;"   />

例子：查询`comments.username包含lee 并且comments.content包含thanks`

<img src="img09/16.png" alt="image-20230820013250703" style="zoom:100%;"   />

结果不对

```json
DELETE blog_index
PUT blog_index/_doc/2
{
  "title":"Blog Number One",
  "author":"alfred",
  "comments":[
      {
        "username":"lee",
        "date":"2017-01-02",
        "content":"awesome article!"
      },
      {
        "username":"fax",
        "date":"2017-04-02",
        "content":"thanks!"
      }
    ]
}

GET blog_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "comments.username": "lee"
          }
        },
        {
          "match": {
            "comments.content": "thanks"
          }
        }
      ]
    }
  }
}
```

用nested 解决该问题：

创建索引：

<img src="img09/17.png" alt="image-20230820015548028" style="zoom:80%;"    />

<img src="img09/18.png" alt="image-20230820015654992" style="zoom:80%;"   />

代码

```json
DELETE blog_index_nested

PUT /blog_index_nested
{
  "mappings": {
    "properties": {
     "title":{
        "type": "text",
        "fields": {
          "keyword":{
            "type":"keyword",
            "ignore_above":100
          }
        },
        "store": true
      },
      "publist_date":{
        "type": "date",
        "store": true
      },
      "author":{
        "type": "keyword",
        "ignore_above":100,
        "store": true
      },
      "abstract":{
        "type": "text",
        "store": true
      },
      "content":{
        "type": "text",
        "store": true
      },
      "url":{
        "type": "keyword",
        "doc_values": false,
        "norms": false,
        "ignore_above": 100,
        "store": true
      },
      "comments":{
        "type": "nested",
        "properties": {
          "username":{
            "type":"keyword",
            "ignore_above": 100
          },
          "date":{
            "type":"date"
          },
          "content":{
            "type": "text"
          }
        }
      }
    }
  }
}

PUT blog_index_nested/_doc/2
{
  "title":"Blog Number One",
  "author":"alfred",
  "comments":[
      {
        "username":"lee",
        "date":"2017-01-02",
        "content":"awesome article!"
      },
      {
        "username":"fax",
        "date":"2017-04-02",
        "content":"thanks"
      }
    ]
}

GET blog_index_nested/_search

GET blog_index_nested/_search
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "comments.username": "lee"
              }
            },
            {
              "match": {
                "comments.content": "awesome"
              }
            }
          ]
        }
      }
    }
  }
}
#无结果
GET blog_index_nested/_search
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "comments.username": "lee"
              }
            },
            {
              "match": {
                "comments.content": "thanks"
              }
            }
          ]
        }
      }
    }
  }
}
```

## 7.2 关联关系处理 - Parent/Child

### 7.2.1 ES提供了类似关系数据库中join的实现方式，使用`join`数据类型实现

<img src="img09/19.png" alt="image-20230820020927091" style="zoom:40%;" />

Join 6.x之后才有，之前使用的是type，多type方式

<img src="img09/20.png" alt="image-20230820021308652" style="zoom:40%;" />

**注意：子文档必须好父文档在一个shard里，否则查询时会有问题**

### 7.2.2 常见query语法包括如下几种

- parent_id 返回某父文档的子文档
- has_child 返回包含某子文档的父文档
- has_parent 返回包含某父文档的子文档

#### 7.2.2.1 parent_id查询：返回某父文档的子文档

<img src="img09/21.png" alt="image-20230820022018241" style="zoom:40%;" />

注释：返回 父文档id=2 的所有子文档类型为 comment 的子文档

#### 7.2.2.2 has_child查询：返回包含某子文档的父文档

<img src="img09/22.png" alt="image-20230820022434276" style="zoom:50%;" />

返回子文档类型comment的字段comment 包含world 的所有父文档

#### 7.2.2.3 has_parent查询：返回包含某父文档的子文档

<img src="img09/23.png" alt="image-20230820022647890" style="zoom:40%;" />

#### 7.2.2.4 例子

<img src="img09/24.png" alt="image-20230820024116740" style="zoom:100%;" />



![image-20230820024806099](img09/25.png)

![image-20230820024951206](img09/26.png)

![image-20230820025039835](img09/27.png)

### 7.3 Nested Object vs Parent/Child

<img src="img09/28.png" alt="image-20230820031008390" style="zoom:50%;" />



# 8 reindex

- 重建所有数据的过程，一般发生在如下情况：
  - mapping设置变更，比如字段类型变化、分词器字典更新等
  - index设置变更，比如分片数更改等
  - 迁移数据
- ES提供了现成的API用于完成该工作
  - _update_by_query 在现有索引上重建
  - _reindex 在其他索引上重建
  -

## 8.1 Reindex - _update_by_query

在原索引上重建

<img src="img09/29.png" alt="image-20230820095424995" style="zoom:50%;" />

**注意：**底层是基于scroll 实现，快照；一般是索引不再发生变动的时候操作

例子：

![image-20230820100034950](img09/30.png)

![image-20230820100216806](img09/31.png)

```json
GET blog_index/_search

GET blog_index/_doc/2

#reindex
POST blog_index/_update_by_query?conflicts=proceed

GET blog_index/_doc/2
```



## 8.2 Reindex - _reindex

将原索引数据重建到目的索引里面

<img src="img09/32.png" alt="image-20230820100505661" style="zoom:50%;" />

还支持远程的es集群

<img src="img09/33.png" alt="image-20230820100734621" style="zoom:50%;" />

例子：

<img src="img09/34.png" alt="image-20230820100921941" style="zoom:100%;" />

```json
POST _reindex
{
  "source": {
    "index": "blog_index"
  },
  "dest": {
    "index": "blog_new_index"
  }
}



GET blog_new_index/_search

```

## 8.3 Redindex - Task

- 数据重建的时间受源索引文档规模的影响，当规模越大时，所需时间越多，此时需要通过设定url参数 `wait_for_completion 为false` 来异步执行，ES以task来描述此类执行任务
- ES提供了Task API 来查看任务的执行进度和相关数据

![image-20230820101436353](/Users/zhangxiaojun/work/study/es/workspace/elasticsearch/docs/es7/09/img09/35.png)

例子：

![image-20230820101619325](/Users/zhangxiaojun/work/study/es/workspace/elasticsearch/docs/es7/09/img09/36.png)

![image-20230820101751297](img09/37.png)

词库发生变化，需要reindex

```json

POST blog_index/_update_by_query?conflicts=proceed&wait_for_completion=false


GET _tasks/Dd2NWBj0T0CihpRoRYM0gg:1260096

```

任务限流：

<img src="img09/38.png" alt="image-20230820101957160" style="zoom:50%;" />

<img src="img09/39.png" alt="image-20230820102046157" style="zoom:50%;" />

利用scroll slice加速

详细见API文档：https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docs-update-by-query.html



# 9 其他建议

## 9.1 数据模型版本管理

- 对mapping进行版本管理

  - 包含在代码或者以专门的文件进行管理，添加好注释，并加入git 等版本管理仓库中，方便回顾

  - 为每个增加一个metadata字段，在其中维护一些文档相关的元数据，方便对数据进行管理

    <img src="img09/40.png" alt="image-20230820102651234" style="zoom:50%;" />

-

## 9.2 防止字段过多

- 字段过多主要有如下的坏处：
  - 难于维护，当字段成百上千时，基本很难有人能明确知道每个字段的含义
  - mapping 的信息存储在cluster state里面，和所有节点进行同步，过多的字段会导致mapping 过大，最终导致更新变慢
- 通过设置 `index.mapping.total_fields.limit `可以限定索引中最大字段数，默认是1000
- 可以通过 key/value 的方式解决字段过多的问题，但并不完美



<img src="img09/42.png" alt="image-20230820103450726" style="zoom:50%;"  />

<img src="img09/43.png" alt="image-20230820103837171" style="zoom:40%;"  />

10个字段把变化多样的cookie都包含在里面了，查询复杂度会提高

<img src="img09/44.png" alt="image-20230820103837171" style="zoom:40%;"  />



Key / value 解决字段过多问题：

```json
DELETE demo_common

# key value 方式
PUT demo_common
{
  "mappings": {
    "properties": {
      "url":{
        "type": "keyword"
      },
      "@timestamp":{
        "type": "date"
      },
      "cookies":{
        "properties": {
          "username":{
            "type":"keyword"
          },
          "startTime":{
            "type":"date"
          },
          "age":{
            "type":"integer"
          }
        }
      }
    }
  }
}

PUT demo_common/_doc/1
{
  "url":"/home",
  "@timestamp":"2017-09-10",
  "cookies":{
    "username":"time",
    "statTime":"2017-09-10T12:09:02",
    "age":12
  }
}

PUT demo_common/_doc/2
{
  "url":"/home",
  "@timestamp":"2017-10-10",
  "cookies":{
    "username":"tom",
    "statTime":"2017-08-10T12:09:02",
    "age":20
  }
}


GET demo_common/_search
{
  "query": {
    "range": {
      "cookies.age": {
        "gte": 10,
        "lte": 20
      }
    }
  }
}


# key value 方式
DELETE demo_key_value

PUT demo_key_value
{
  "mappings": {
    "properties": {
      "url":{
        "type": "keyword"
      },
      "@timestamp":{
        "type": "date"
      },
      "cookies":{
        "type": "nested",
        "properties": {
          "cookieName":{
            "type":"keyword"
          },
          "cookieValueKeyword":{
            "type":"keyword"
          },
          "cookieValueInteger":{
            "type":"integer"
          },
          "cookieValueDate":{
            "type":"date"
          }
        }
      }
    }
  }
}


PUT demo_key_value/_doc/1
{
  "url":"/home",
  "@timestamp":"2017-09-10",
  "cookies":[
      {
        "cookieName":"username",
        "cookieValueKeyword":"time"
      },
      {
        "cookieName":"statTime",
        "cookieValueDate":"2017-09-10T12:09:02"
      },
      {
        "cookieName":"age",
        "cookieValueInteger":12
      }
    ]
}

PUT demo_key_value/_doc/2
{
  "url":"/index",
  "@timestamp":"2017-10-10",
  "cookies":[
      {
        "cookieName":"tom",
        "cookieValueKeyword":"time"
      },
      {
        "cookieName":"statTime",
        "cookieValueDate":"2017-08-10T12:09:02"
      },
      {
        "cookieName":"age",
        "cookieValueInteger":20
      }
    ]
}

GET demo_key_value/_search
{
  "query": {
    "nested": {
      "path": "cookies",
      "query": {
        "bool": {
          "filter": [
            {
              "term": {
                "cookies.cookieName": "age"
              }
            },
            {
              "range": {
                "cookies.cookieValueInteger": {
                  "gte": 15,
                  "lte": 20
                }
              }
            }
          ]
        }
      }
    }
  }
}

```

对 cookieName 做聚合分析，则key/value 无法实现，因为分成2个字段

### 9.2.1 key/value 方式详解

- 虽然通过这种方式可以极大地减少field数量，但也有一些明显的坏处
  - query 语句复杂度飙升，且有一些可能无法实现，比如聚合分析相关的
  - 不利于在Kibana中做可视化分析
  -

### 9.2.2 防止字段过多

- 一般字段过多的原因是由于没有高质量的数据建模导致的，比如 dynamic 设置为 true
- 考虑拆分多个索引来解决
