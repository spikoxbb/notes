[TOC]



# Mapping

## Dynamic Mapping

- 在写入文档时，如果索引不存在会自动创建索引
- ES会自动根据文档信息推算字段类型

![WechatIMG12](../img/WechatIMG12.jpeg)

```java
//get mapping_test/_mapping 
"mappings" : {
      "properties" : {
        "firstName" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "lastName" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "loginDate" : {
          "type" : "date"
        }
      }
    }
  }
```

### 更改Mapping类型

- 新增字段

  1. Dynamic为true时，Mapping同时被更新（文档依旧被写入ES，新字段也被索引进去）

  2. Dynamic为false时，Mapping不会被更新，新增字段的数据无法被索引，但是信息会出现在_source中（新文档被写入ES，新字段未被索引进去）

     > PUT dynamic_mapping_test/_mapping
     > {
     >   "dynamic":false
     > }
     >
     > PUT dynamic_mapping_test/_doc/11
     > {
     >   "anotherField1":"someValue"
     > }
     >
     > //搜索不到
     >
     > get dynamic_mapping_test/_search?q=anotherField1:(someValue)

  3. Dynamic为strict时，文档写入失败（新文档未被写入ES，因此新字段更不能被索引进去）

     > PUT dynamic_mapping_test/_mapping
     > {
     >   "dynamic": "strict"
     > }
     >
     > //添加报错
     >
     > PUT dynamic_mapping_test/_doc/12
     > {
     >   "lastField":"value"
     > }

- 更新字段

  - 不可修改字段定义（倒排索引生成后不可修改，除非reindex）

## 显式Mapping设置

### 设置 index 为 false
```java
PUT users
{
    "mappings" : {
      "properties" : {
        "firstName" : {
          "type" : "text"
        },
        "lastName" : {
          "type" : "text"
        },
        "mobile" : {
          "type" : "text",
          "index": false
        }
      }
    }
}
//报错，未索引
POST /users/_search
{
  "query": {
    "match": {
      "mobile":"12345678"
    }
  }
}
```

### 设定Null_value

一切文本类型的字符串可以定义成 “text”或“keyword”两种类型。区别在于，text类型会使用默认分词器分词，当然你也可以为他指定特定的分词器。如果定义成keyword类型，那么默认就不会对其进行分词。

```java
//只有keyword类型才能设定Null_value
PUT users
{
    "mappings" : {
      "properties" : {
        "firstName" : {
          "type" : "text"
        },
        "lastName" : {
          "type" : "text"
        },
        "mobile" : {
          "type" : "keyword",
          "null_value": "NULL"
        }

      }
    }
}

PUT users/_doc/1
{
  "firstName":"Ruan",
  "lastName": "Yiming",
  "mobile": null
}
GET users/_search
{
  "query": {
    "match": {
      "mobile":"NULL"
    }
  }
}
```

### 设置 Copy to

```java
PUT users
{
  "mappings": {
    "properties": {
      "firstName":{
        "type": "text",
        "copy_to": "fullName"
      },
      "lastName":{
        "type": "text",
        "copy_to": "fullName"
      }
    }
  }
}

PUT users/_doc/1
{
  "firstName":"Ruan",
  "lastName": "Yiming"
}

GET users/_search?q=fullName:(Ruan Yiming)
//虽然_source中不显示，Mapping中有，应该是“数组”
 "fullName" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
```

### 数组类型

``` java
PUT users/_doc/1
{
  "name":"twobirds",
  "interests":["reading","music"]
}
GET users/_mapping
//类型为text
 "interests" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
```

# 多字段

```java
  "firstName" : {
          "type" : "text",
    //多字段定义，text类型默认有KeyWord
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
```

## Exact Value && Full Text

### Exact Value

精确值，如数字，日期，具体的字符串（如“Apple Store”）

- ES 的 keyword

###  Full Text

全文本

- ES 的 text

**Exact Value不需要被分词**