[TOC]

# search API

- URI模式

  ```java
  //q指定查询字符串
  GET movies/_search?q=title:Bulworth 
  
  GET mov*/_search?q=title:Bulworth
    
  //对集群上所有索引进行查询
  GET _all/_search?q=title:Bulworth  GET /_search?q=title:Bulworth 
  ```

- Request Body模式 （DSL语句）

  ```java
  POST movies/_search
  {
    "query": {"match_all": {}},
    "size": 1
  }
  ```

  ## 衡量相关性

  - Precision（查准率）---尽可能返回较少的无关文档
  - Recall（查全率）---尽可能返回较多的相关文档
  - Ranking ---是否能根据相关度排序

  ![WechatIMG9](/Users/spiko/Documents/notes/img/WechatIMG9.jpeg)

  Precision = True Positive /  (True Positive +  False Positive)

  Recall = True Positive /  (True Positive +  True Negatives)

## URI Search

- GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s

  > q指定查询语句，使用query String syntax
  >
  > df查询字段 不指定时会对所有字段进行查询
  >
  > sort排序
  >
  >  from size 进行分页
  >
  > profile可以查看查询是如何执行的

- GET /movies/_search?q=2012&df=title

  > "type" : "TermQuery"
  >
  >  "description" : "title:2012"

- GET /movies/_search?q=2012

  >  "type" : "DisjunctionMaxQuery",
  > "description" : "(title.keyword:2012 | id.keyword:2012 | year:[2012 TO 2012] | genre:2012 | @version:2012 | @version.keyword:2012 | id:2012 | genre.keyword:2012 | title:2012)"
  >
  > **每一个children query就是相应的TermQuery**

- GET /movies/_search?q=title:Beautiful Mind

  >  "type" : "BooleanQuery",
  >  "description" : """title:beautiful (title.keyword:Mind | id.keyword:Mind | MatchNoDocsQuery("failed [year] query, caused by number_format_exception:[For input string: "Mind"]") | genre:mind | @version:mind | @version.keyword:Mind | id:mind | genre.keyword:Mind | title:mind)""",

- GET /movies/_search?q=title:"Beautiful Mind"

  >  "type" : "PhraseQuery",
  > "description" : """title:"beautiful mind"""",

- GET /movies/_search?q=title:(Beautiful Mind)

  同 GET /movies/_search?q=title:(+Beautiful +Mind)

  >   "type" : "BooleanQuery"
  >
  >   "description" : "title:beautiful title:mind",

- GET /movies/_search?q=title:(Beautiful AND Mind)

  **AND等一定要大写**

  >  "type" : "BooleanQuery",
  > "description" : "+title:beautiful +title:mind",

- GET /movies/_search?q=title:(Beautiful NOT Mind)

  同 GET /movies/_search?q=title:(Beautiful -Mind)

  **分组：+must   -must not** 

  > "type" : "BooleanQuery",
  > "description" : "title:beautiful -title:mind",

- GET /movies/_search?q=title:(Beautiful %2BMind)

  > "type" : "BooleanQuery",
  >  "description" : "title:beautiful +title:mind",

- GET /movies/_search?q=title:beautiful AND year:[2002 TO 2018%7D

  > "type" : "BooleanQuery",
  >  "description" : "+title:beautiful +year:[2002 TO 2017]",

- GET /movies/_search?q=title:b*

  > "type" : "MultiTermQueryConstantScoreWrapper",
  > "description" : "title:b*",

- GET /movies/_search?q=title:beautifua~1

  >"type" : "BoostQuery",
  >"description" : "(title:beautiful)^0.8888889",

- GET /movies/_search?q=title:"Lord the Rings"~1

  >  "type" : "PhraseQuery",
  >   "description" : """title:"lord the rings"~1""",

## Request Body Search

### Match

- #ignore_unavailable=true，可以忽略尝试访问不存在的索引“404_idx”导致的报错

  #查询movies分页
  POST /movies,404_idx/_search?ignore_unavailable=true
  {
    "profile": true,
  	"query": {
  		"match_all": {}
  	}
  }

  默认从0开始，返回10条结果

  > "type" : "MatchAllDocsQuery",
  >   "description" : "*:*",

- POST /kibana_sample_data_ecommerce/_search
  {
    "from":10,
    "size":20,
    "query":{
      "match_all": {}
    }
  }

- #对日期排序

  POST movies/_search
  {
     "profile": true,
    "sort":[{"year":"desc"}],
    "query":{
      "match_all": {}
    }

  }

- #source filtering
  POST movies/__search
  {
    "_source":["year"],
    "query":{
      "match_all": {}
    }
  }

  可使用通配符"_source":["year","titl*"],

- #脚本字段 拼接
  GET movies/_search
  {
    "script_fields": {
      "lalala": {
        "script": {
          "lang": "painless",
          "source": "doc['year'].value+'hello'"
        }
      }
    },
    "query": {
      "match_all": {}
    }
  }

- POST movies/_search
  {
    "profile": true,
    "query": {
      "match": {
        "title": "last christmas"
      }
    }
  }

  >   "type" : "BooleanQuery",
  > "description" : "title:last title:christmas",

- POST movies/_search
  {
    "profile": true,
    "query": {
      "match": {
        "title": {
          "query": "last christmas",
          "operator": "and"
        }
      }
    }
  }

  >  "type" : "BooleanQuery",
  >  "description" : "+title:last +title:christmas",

### Match Phrase

- POST movies/_search
  {
    "query": {
      "match_phrase": {
        "title": {
          "query": "Nightmare Street",
          "slop": 2
        }
      }
    }
  }

  slop代表中间可以有其他的字符进入

### Query String 

和URI Search类似

- POST movies/_search
  {
    "query": {
      "query_string": {
        "default_field": "title",
        "query": "Six Days Seven NOT Nights"
      }
    }
  }

- POST movies/_search
  {
    "query": {
      "query_string": {
        "fields": ["title","genre"], 
        "query": "Six Days Seven NOT Nights"
      }
    }
  }

  指定多个field

### Simple Query String 

1. 类似Query String ，但是会忽略错误语法
2. 不支持NOT AND OR，会被当作字符串处理，默认是OR，需指定operator
3. 支持+ - |

- POST movies/_search
  {
    "profile": "true", 
    "query": {
      "simple_query_string": {
        "query": "Six AND Days AND  Nights ",
        "fields": ["title"]
      }
    }
  }

  >  "type" : "BooleanQuery",
  >  "description" : "title:nights title:days title:six (title:and)^2
  >
  > 由于Simple Query 默认的operator是 Or，且不支持NOT AND OR，需要转换成如下：

  POST movies/_search
  {
    "query": {
      "simple_query_string": {
        "query": "Six Days Nights ",
        "fields": ["title"],
        "default_operator": "AND"
      }
    }
  }

  >  "type" : "BooleanQuery",
  >  "description" : "+title:six +title:days +title:nights",
  
- GET /movies/_search
  {
  	"profile":true,
  	"query":{
  		"simple_query_string":{
  			"query":"Beautiful +mind",
  			"fields":["title"]
  		}
  	}
  }

  > "type" : "BooleanQuery",
  >     "description" : "title:beautiful title:mind",
