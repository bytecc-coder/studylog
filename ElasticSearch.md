# ElasticSearch

## 启动集群

elasticsearch -E node.name=node0 -E cluster.name=geektime -E path.data=node0_data -d

elasticsearch -E node.name=node1 -E cluster.name=geektime -E path.data=node1_data -d

elasticsearch -E node.name=node2 -E cluster.name=geektime -E path.data=node2_data -d

elasticsearch -E node.name=node3 -E cluster.name=geektime -E path.data=node3_data -d

查看集群节点

/_cat/nodes

logstash导入数据：logstash -f logstash.conf

#### 插件安装：

1. 进入es的容器并启动bash。 命令 docker exec -it es7_01 bash  注：es7_01 即容器名称
2. 第一步成功你会发现你已经在容器内部，此时输入 pwd 命令会发现自己处于/usr/share/elasticsearch 路径。此时即可输入插件安装命令 bin/elasticsearch-plugin install analysis-icu等待插件下载并安装完毕
3. 输入exit退出容器bash。
4. 如法炮制es7_02并安装插件。
5. docker-compose restart 重启容器
6. 重启后，检查安装是否成功，输入 curl 127.0.0.1:9200/_cat/plugins，输出：
   es7_01 analysis-icu 7.2.0
   es7_02 analysis-icu 7.2.0

## 文档相关信息

##### _index:文档所属索引名称

##### _type:文档所属类型名（7.x已废弃）

##### _id:文档唯一ID

##### _source:文档的原始json数据

##### _version:文档的版本信息

##### _score:相关性打分

##### mapping:定义文档字段的类型

##### setting:定义不同的数据分布

## 节点

### 节点类型

#### Data Node

​				可以保存数据的节点，负责保存分片数据，在数据扩展上起到至关重要的作用。

#### Coordinating Node

​				负责接受Client请求，将请求分发给合适的节点，最终把结果汇集到一起。

​                每个节点默认都起到了Coordinating Node的职责

#### Hot&Warm Node(冷热节点)

​                不同配置的Data Node,用来实现Hot&Warm架构，降低集群部署的成果

#### Machine Learning Node

​                  负责跑机器学习的Job，用来做异常监测

### 节点配置

![image-20210414104817622](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210414104817622.png)

## 分片

### 分片类型

#### 主分片

解决数据水平扩展问题。通过主分片，可以将数据分布到集群所有节点。

- 一个分片是一个运行的Lucene的实例
- 主分片在索引创建时指定，后续不允许修改，除非Reindex

#### 副本

用以解决数据高可用问题，分片是主分片的拷贝

- 副本分片数可以动态调整
- 增加副本数，还可以在一定程度上提高服务的可用性。

## 基础配置

### 分词器

![image-20210423140747648](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210423140747648.png)

#### 系统自带分词器

```
$standard Analyzer - 按词切分，小写处理
#Simple Analyzer – 按照非字母切分（符号被过滤），小写处理
#Stop Analyzer – 小写处理，停用词过滤（the，a，is）
#Whitespace Analyzer – 按照空格切分，不转小写
#Keyword Analyzer – 不分词，直接将输入当作输出
#Patter Analyzer – 正则表达式，默认 \W+ (非字符分隔)
#Language – 提供了30多种常见语言的分词器
#ik_smart - 最少划分
#ik_max_word - 最小粒度划分
#icu-analyzer - 提供了Unicode的支持，更好的支持亚洲语言 
#thulac 清华大学分词器 http://github.com/microbun/elasticsearch-thulac-plugin
```

#### 自定义分词器

- 当自带的分词器无法满足时,可以通过自定义分词器。通过自组合不同的组件实现

  - Character Filters

    - 在Tokenizer之前对文本进行处理，例如增加、删除及替换字符。可以配置多个Character Filters。会影响Tokenizer的position和offset信息。

    - 一些自带的Character Filters

      - HTML strip ---  去除html标签

      - Mapping ---  字符串替换

      - Pattern replace ---  正则匹配替换

    ```elasticsearch
    POST _analyze
    {
    	"tokenizer":"keyword",
    	"char_filter":["html_strip"],
    	"text":"<b>hello world</b>"
    }
    result:
    {
      "tokens" : [
        {
          "token" : "hello world",
          "start_offset" : 3,
          "end_offset" : 18,
          "type" : "word",
          "position" : 0
        }
      ]
    }
    
    
    //使用char_filter进行替换
    POST _analyze
    {
    	"tokenizer":"standard",
    	"char_filter":[
    		{
    			"type":"mapping",
    			"mappings":["l=>t"]
    		}
    	],
    	"text":"hello world"
    }
    result:
    {
      "tokens" : [
        {
          "token" : "hetto",
          "start_offset" : 0,
          "end_offset" : 5,
          "type" : "<ALPHANUM>",
          "position" : 0
        },
        {
          "token" : "wortd",
          "start_offset" : 6,
          "end_offset" : 11,
          "type" : "<ALPHANUM>",
          "position" : 1
        }
      ]
    }
    
    //使用正则匹配替换
    POST _analyze
    {
    	"tokenizer":"standard",
    	"char_filter":[
    		{
    			"type":"pattern_replace",
    			"pattern":"http://(.*)",
    			"replacement":"$1"
    		}
    	],
    	"text":"http://baidu.com"
    }
    result:
    {
      "tokens" : [
        {
          "token" : "baidu.com",
          "start_offset" : 0,
          "end_offset" : 16,
          "type" : "<ALPHANUM>",
          "position" : 0
        }
      ]
    }
    ```

  - Tokenizer

    - 将原始的文本按照一定的规则切分为词（term or token）
    - Elasticsearch内置的Tokenizers
      - whitespace / standard / uax_url_email / pattern / keyword / path hierarchy
    - 可以用JAVA开发插件，实现自己的Tokenizer

    ```elasticsearch
    POST _analyze
    {
    	"tokenizer":"path_hierarchy",
    	"text":"usr/es/log"
    }
    result:
    {
      "tokens" : [
        {
          "token" : "usr",
          "start_offset" : 0,
          "end_offset" : 3,
          "type" : "word",
          "position" : 0
        },
        {
          "token" : "usr/es",
          "start_offset" : 0,
          "end_offset" : 6,
          "type" : "word",
          "position" : 0
        },
        {
          "token" : "usr/es/log",
          "start_offset" : 0,
          "end_offset" : 10,
          "type" : "word",
          "position" : 0
        }
      ]
    }
    ```

  - Token Filter

    - 将Tokenizer输出的单词(term)进行增加、修改、删除
    - 自带的Token Filter
      - Lowercase / stop / synonym(添加近义词)

    ```elasticsearch
    //whitespace与stop
    GET _analyze
    {
    	"tokenizer":"whitespace",
    	"filter":["stop"],
    	"text":["The rain Spain falls mainly on the plain."]
    }
    result:
    {
      "tokens" : [
        {
          "token" : "The",
          "start_offset" : 0,
          "end_offset" : 3,
          "type" : "word",
          "position" : 0
        },
        {
          "token" : "rain",
          "start_offset" : 4,
          "end_offset" : 8,
          "type" : "word",
          "position" : 1
        },
        {
          "token" : "Spain",
          "start_offset" : 9,
          "end_offset" : 14,
          "type" : "word",
          "position" : 2
        },
        {
          "token" : "falls",
          "start_offset" : 15,
          "end_offset" : 20,
          "type" : "word",
          "position" : 3
        },
        {
          "token" : "mainly",
          "start_offset" : 21,
          "end_offset" : 27,
          "type" : "word",
          "position" : 4
        },
        {
          "token" : "plain.",
          "start_offset" : 35,
          "end_offset" : 41,
          "type" : "word",
          "position" : 7
        }
      ]
    }
    
    ```

##### 自定义一个分词器

- 设置索引

```elasticsearch
PUT my_index
{
	"settings":{
		"analysis":{
			"analyzer":{
				"my_custom_analyzer":{
					"type":"custom",
					"char_filter":["emotions"],
					"tokenizer":"punctuation",
					"filter":["lowercase","english_stop"],
				}
			},
			"tokenizer":{
				"punctuation":{
					"type":"pattern",
					"pattern":"[.,!?]"
				}
			},
			"char_filter":{
				"emotions":{
					"type":"mapping",
					"mappings":[
						":)=>_happy_",
						":(=>_sad_"
					]
				}
			},
			"filter":{
				"english_stop":{
					"type":"stop",
					"stopwords":"_english_"
				}
			}
		}
	}
}
```



### 衡量相关性

#### 指标：

 Precision(查准率)-尽可能返回较少的无关文档

 Recall(查全率)-尽量返回较多的相关文档

 Ranking-是否能够按照相关度进行排序

##### 计算方法：

![image-20210419195753437](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210419195753437.png)

#### 相关性和相关性算分

![image-20210426151757165](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210426151757165.png)

##### 相关性---Relevance

- 搜索的相关性算分，描述了一个文档和查询语句匹配的程度。ES会对每个匹配查询条件的结果进行算分_score
- 打分的本质是排序，需要把最符合用户需求的文档排在前面。ES5之前，默认的相关性算分采用TF-IDF，现在采用BM 25。
- 

###### 词频  TF

- Term Frequency:检索词在一篇文档中出现的频率
  - 检索词出现的次数除以文档的总字数
- 度量一条查询和结果文档相关性的简单方法：简单将搜索中每一个词的TF进行相加
  - 区块链的应用：TF（区块链）+TF（的）+TF（应用）
- Stop Word
  - "的”在文档中出现了很多次，但是对贡献相关度几乎没有用处，不应该考虑他们的TF

###### 逆文档频率 IDF

- DF：检索词在所有文档中出现的频率

  - “区块链”在相对比较少的文档中出现
  - “应用”在相对比较多的文档中出现
  - “Stop World”在大量的文档中出现

- Inverse Document Frequency:简单的说=log(全部文档数/索词出现过的文档总数）

- TF-IDF本质上就是将TF求和变成了加权求和

  - TF(区块链) * IDF(区块链) +TF(的) * IDF(的) + TF(应用) * IDF(应用)

  ![image-20210426151411616](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210426151411616.png)

###### BM25 

![image-20210426151933697](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210426151933697.png)

###### 参数

- explain:true 查看计算过程

- Boosting  Relevance
  - Boosting是控制相关度的一种手段
    - 索引、字段或查询子条件
    
    ```json
    POST news/_search
    {
        "query":{
            "boosting":{
                "positive":{//正相关
                    "match":{
                        "content":"apple"
                    }
                },
                "negative":{//负相关
                    "match":{
                        "content":"pie"
                    }
                }
            },
            "negative_boost":0.5
        }
    }
    ```
    
    
    
  - 参数boost的含义
    - 当boost>1时，打分的相关度相对性提升
    - 当0<boost<1时，打分的权重相对性降低
    - 当boost<0时，贡献负分
    
    ```json
    POST blogs/_search
    {
        "query":{
            "bool":{
                "should":[
                    {
                        "match":{
                            "title":{
                                "query":"apple,ipad",
                                "boost":4
                            }
                        }
                    },
                    {
                        "match":{
                            "content":{
                                "query":"apple,ipad",
                                "boost":1
                            }
                        }
                    }
                ]
            }
        }
    }
    ```
    
    

### Mapping(/_mapping)

- Mapping类似数据中的schema的定义，作用如下
  - 定义索引中的字段名称C
  - 定义字段的数据类型
  - 字段、倒排索引的相关配置
- Mapping会把JSON文档映射成Lucene所需的扁平格式
- 一个Mapping属于一个索引的Type
- 控制dynamic mapping 

```
{
	"mappings":{
		"_doc":{
			"dynamic":"false"
		}
	}
}
```



#### 字段类型

##### 简单类型

- Text/Keyword
- Date
- Integer/Floating/long/byte/sort/double
- Boolean
- IPV4&IPV6

##### 复杂类型

- 对象类型/嵌套类型

##### 特殊类型

- geo_point&geo_shape/percolator

##### 多字段类型

- 多字段特性

  - 厂商名字实现精确匹配
    - 增加一个keyword字段

  - 使用不同的analyzer
    - 不同语言
    - pinyin字段的搜索
    - 还支持为搜索和索引指定不同的analyzer

##### Exact Values v.s Full Text

- Exact value 包括数字、日期、具体一个字符串(不做分词处理)
  - ElasticSearch中的keyword
- Full Text 非结构化的文本数据
  - ElasticSearch中的Text

#### 显式Mapping

```elasticsearch
{
	"mappings":{
	
	}
}
```

##### 设置mapping文件建议步骤

- 创建一个临时index,写入一些样本数据
- 通过访问Mapping API获得该临时文件的动态Mapping定义
- 修改后用，使用该配置创建你的索引
- 删除临时索引

##### mapping参数

- index:控制当前字段是否被索引。

  ```
  {
  	"mappings":{
  		"properties":{
  			"first_name":{
  				"type":"text"
  			},
  			"last_name":{
  				"type":"text",
  				"index":false
  			},
  			"mobile":{
  				"type":"text",
  				"index_options":"offsets"
  			}
  		}
  	}
  }
  ```

  

  - true:默认值，false:该字段不可被索引
  - index_options:控制倒排索引记录的内容，记录内容越多，占用存储空间越大
    - docs : （除text以外的默认值）记录doc id
    - freqs : 记录doc id 和 term frequencies
    - positions : (text默认值) 记录doc id / term frequencies / term position
    - offsets : doc id / term frequencies / term position / character offects

- null_value:需要对Null值实现搜索，只有Keyword类型支持设定Null_Value

```elasticsearch
{
	"mappings":{
		"properties":{
			"first_name":{
				"type":"text"
			},
			"last_name":{
				"type":"text",
				"index":false
			},
			"mobile":{
				"type":"keyword",
				"null_value":"NULL"
			}
		}
	}
}
```

- copy_to:满足一些特定的搜索需求，将字段的数值拷贝到目标字段，且目标字段不出现在_source中

```elasticsearch
{
	"mappings":{
		"properties":{
			"first_name":{
				"type":"text",
				"copy_to":"fullName"
			},
			"last_name":{
				"type":"text",
				"copy_to":"fullName"
			}
		}
	}
}

GET /users/_search?q=fullName:(zhang san)
```

- 数组类型：不提供专门的数组类型，但是任何字段都可以包含多个相同类类型的数值

### <u>Dynamic Mapping</u>

- 在写入文档的时候，如果索引不存在，会自动创建索引
- Dynamic Mapping的机制使得我们无需手动定义Mappng。ElasticSearch会自动根据文档信息推算出字段的类型
- 但是有时候会推算错误，例如地理位置信息
- 当类型设置不对时，会导致一些功能无法正常运行，例如Rang查询

#### 字段类型匹配

- 字符串
  - 匹配日期格式，设置成Date
  - 配置数字设置为float或者long,该选项默认关闭
  - 设置为Text,并且增加keyword子字段
- 布尔值 ：boolean
- 浮点数：float
- 整数：long
- 对象：Object
- 数组：由第一个非空值的类型所决定
- 空值：忽略

### 能否更改Mapping的字段类型

- 两种情况
  - 新增加字段
    - Dynamic设为true时，一旦有新增字段的文档写入，Mapping也同时被更新
    - Dynamic设为false时,Mapping不会被更新，新增字段的数据无法被索引，但是信息会在_source中
    - Dynamic设为Strict,新增字段的文档写入失败
  - 对已有字段，一旦已经有数据写入，就不再支持修改字段定义
    - Lucene实现的倒排索引，一旦生成后就不允许被修改
  - 如果希望改变字段类型，必须Reindex API,重建索引
- 原因
  - 如果改了字段的数据类型，会导致已经被索引的数据无法被搜索
  - 但是如果是增加新的字段，就不会有这样的影响

### Index

#### Template

```
//查看template
GET /_template/{template_name}
```

##### Index Template

- 工作方式
  - 当一个索引被创建时
    - 首先应用Elasticsearch默认的settings和mappings
    - 优先应用order数值低的index template设定
    - 应用order高的Index Template中的设定时，之前的设定会被覆盖
    - 应用创建索引时，用户如果指定的Setting和Mappings会覆盖之前模板中的设定

- 帮助设定Mappings和Settings,并按照一定的规则，自动匹配到新创建的索引之上

  - 模板仅在一个索引被创建时才会产生作用。修改模板不会影响已创建的模板
  - 可以设置多个索引模板，这些设置会被merge在一起
  - 可以指定order的数值，控制merging的过程

  ```
  PUT _templates/template_default //主分片和副本分片都是1
  {
  	"index_patterns":["*"],
  	"order":0,
  	"version":1,
  	"settings":{
  		"number_of_shards":1,
  		"number_of_replacas":1
  	}
  }
  
  PUT _templates/template_test //以test开头的索引，主分片1，副本分片2，关闭字符串自动转date,开启字符串自动映射数字
  {
  	"index_patterns":["test*"],
  	"order":1,
  	"settings":{
  		"number_of_shards":1,
  		"number_of_replacas":2
  	}
  	"mappings":{
  		"date_detection":false,
  		"numeric_detection":true
  	}
  }
  ```

##### Dynamic Template

- 根据Elasticsearch识别的数据类型，结合字段名称，来动态设定字段类型

  - 所有的字符串类型都设定成Keyword,或者关闭keyword字段
  - is开头的字段都设置成boolean
  - long_开头的都设置成long类型

- 使用方法

  - 定义在某个索引的Mapping中
  - 有一个名称
  - 匹配规则是一个数组
  - 为匹配字段设置Mapping

  ```json
  PUT my_test_index
  {
  	"mappings":{
  		"dynamic_templates":[
              {
                  "full_name":{
                      "path_match":"name.*",
                      "path_unmatch":".middle",
                      "mapping":{
                          "type":"text",
                          "copy_to":"full_name"
                      }
                  }
              }
  		]
  	}
  }
  
  PUT /my_test_index/_doc/1
  {
  	"name":{
          "firstName":"ming",
          "middle":"Miston",
          "last":"zb"
      }
  }
  
  GET my_test_index/_search?q=full_name:ming
  {
    "took" : 59,
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
      "max_score" : 0.2876821,
      "hits" : [
        {
          "_index" : "my_test_index",
          "_type" : "_doc",
          "_id" : "1",
          "_score" : 0.2876821,
          "_source" : {
            "name" : {
              "firstName" : "ming",
              "middle" : "Miston",
              "last" : "zb"
            }
          }
        }
      ]
    }
  }
  
  
  //demo
  PUT my_index
  {
  	"mappings":{
  		"dynamic_templates":[
  			{
  				"string_as_boolean":{
  					"match_mapping_type":"string",
  					"match":"is*",
  					"mapping":{
  						"type":"boolean"
  					}
  				}
  			},
  			{
  				"string_as_keywords":{
  					"match_mapping_type":"string",
  					"mapping":{
  						"type":"keyword"
  					}
  				}
  			}
  		]
  	}
  }
  
  PUT /my_index/_doc/1
  {
  	"firstName":"ming",
  	"isVIP":true
  }
  
  GET /my_index/_mapping
  {
    "my_index" : {
      "mappings" : {
        "dynamic_templates" : [
          {
            "string_as_boolean" : {
              "match" : "is*",
              "match_mapping_type" : "string",
              "mapping" : {
                "type" : "boolean"
              }
            }
          },
          {
            "string_as_keywords" : {
              "match_mapping_type" : "string",
              "mapping" : {
                "type" : "keyword"
              }
            }
          }
        ],
        "properties" : {
          "firstName" : {
            "type" : "keyword"
          },
          "isVIP" : {
            "type" : "boolean"
          }
        }
      }
    }
  }
  
  ```

  

## Search API

```
/_search 集群上所有的索引
/index1/_search index1
/index1,index2/_search index1和index2
/index*/_search 以index开头的索引
```

### URI Search(在URL中使用查询参数)

```curl
curl -XGET "localhost:9200/{索引名称}/_search?q={查询的字段:查询的内容}"
```

##### 布尔操作：

```
AND,OR,NOT,&&,||,!
(
	必须大写，
	title:(matrix NOT reloaded)
)
```

##### 分组：

```
+：表示must
-: 表示must_not
title:(+matrix -reloaded)
```

##### 范围：

```
[] 闭区间
{} 开区间
year:{2019 TO 2018]
year:{* TO 2018]
```

##### 算数符号:

```
year:>2000
year:(>2010&&<=2021)
year:(+>2010&&+<=2021)
```

##### 通配符查询

```
?代表1个字符
*代表0或多个字符
```

##### 正则表达

```
title:[bt]oy
```

##### 模糊匹配与近似查询

```
title:befutifl~1
title:befutifl~2
```

### Request Body Search(DSL，基于JSON格式的更加完备的查询)

#### 参数配置

```
from:第n条开始
size:每页条数
sort:排序 eg:  sort:[{"order_date":"desc"}]
_source:返回字段过滤  eg:_source:["title","order_date"]
```

##### 脚本字段

```elasticsearch
"script_fields":{
    "new_field":{
    	"script":{
    		"lang":"painless",
    		"source":"doc['order_date'].value+'_hello'"
    	}
    }
}
```

##### match(terms之间是or的关系)

```elasticsearch
默认：last OR christmas
{
	"query":{
		"match":{
			"comment":"last christmas"
		}
	}
}
```

```elasticsearch
ADN
{
	"query":{
		"match":{
			"comment":{
				"query":"last christmas",
				"operator":"AND"
			}
		}
	}
}
```

##### Match Phrase(短语搜索，terms之间是and关系，并且term之间位置关系也影响搜索的结果)

```elasticsearch
{
	"query":{
		"match_phrase":{
			"comment":{
				"query":"last christmas",
				"slop":1
			}
		}
	}
}
*slop:代表短语中间可以有几个term介入
```

### Query String Query

```elasticsearch
{
	"query":{
		"query_string":{
			"default_field":"name",
			"query":"Ruan AND Yiming"
		}
	}
}

{
	"query":{
		"query_string":{
			"fields:["name","about"],
			"query":"(Ruan AND Yiming) OR (JAVA AND Elasticsearch)"
		}
	}
}
```

### Simple Query String Query

- 类似Query String,但是会忽略错误的语法，同时只支持部分查询语法
- 不支持AND,OR,NOT，会当作字符串处理，Term之间默认关系是OR，可以指定Operator
- 支持部分逻辑：+代替AND，|代替OR，-代替NOT

```elasticsearch
{
	"query":{
		"simple_query_string":{
			"fields":["name"],
			"query":"Ruan AND Yiming",
			"default_operator":"AND"
		}
	}
}
```

### 深入搜索

#### 基于Term的查询

- Term的重要性

  - Term是表达语意的最小单位。搜索和利用统计语言模型进行自然语言处理都需要处理Term。

- 特点

  - Term Level Query:Term Query/Range Query/Exists Query/Prefix Query/Wildcard Query

    - Term Query:精确查询
    - Range Query：范围查询

    ```json
    GET /_search
    {
      "query": {
        "range": {
          "age": {
            "gte": 10,
            "lte": 20,
            "boost": 2.0
          }
        }
      }
    }
    ```

    ![image-20210426143635641](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210426143635641.png)

    - Exists Query：某个字段是否存在（值为null无法查出）
    
    ```json
    {
      "query": {
        "exists": {
          "field": "user"
        }
      }
    }
    ```
    
    - Fuzzy Query：返回包含与搜索项类似的项的文档
      - 分类
        - Changing a character (**b**ox → **f**ox)
        - Removing a character (**b**lack → lack)
        - Inserting a character (sic → sic**k**)
        - Transposing two adjacent characters (**ac**t → **ca**t)
      - 参数
        - fuzziness：允许匹配的最大编辑距离
    - max_expansions：允许的最大变体数量，Avoid using a high value in the max_expansions parameter
        - transpositions:是否包含两个字符换位，默认为true
    
    ```json
    GET /_search
    {
      "query": {
        "fuzzy": {
          "user.id": {
            "value": "ki",
            "fuzziness": "AUTO",
            "max_expansions": 50,
            "prefix_length": 0,
            "transpositions": true,
            "rewrite": "constant_score"
          }
        }
      }
    }
    ```

    - IDs Query:Returns documents based on their IDs. This query uses document IDs stored in the _id field
    
    ```json
    GET /_search
    {
      "query": {
        "ids" : {
          "values" : ["1", "4", "100"]
        }
      }
    }
    ```

    - prefix Query:返回在提供的字段中包含特定前缀的文档
    
    ```json
    GET /_search
    {
      "query": {
        "prefix": {
          "user.id": {
            "value": "ki"
          }
        }
      }
    }
    ```
  
- Regexp Query:返回包含匹配正则表达式的术语的文档
      - 默认情况下，正则表达式限制为1,000个字符。您可以使用[`index.max_regex_length`](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/index-modules.html#index-max-regex-length) 设置更改此限制。
  
    ```json
    GET /_search
    {
      "query": {
        "regexp": {
          "user.id": {
            "value": "k.*y",
            "flags": "ALL",
            "case_insensitive": true,//允许不区分大小写
            "max_determinized_states": 10000,
            "rewrite": "constant_score"
          }
        }
      }
  }
    ```

    - Terms Query
  
    ```json
    GET /_search
    {
      "query": {
        "terms": {
          "user.id": [ "kimchy", "elkbee" ],
          "boost": 1.0
        }
      }
  }
    ```

    - Terms set Query 
  
    ```json
    PUT /job-candidates/_doc/1?refresh
    {
      "name": "Jane Smith",
      "programming_languages": [ "c++", "java" ],
      "required_matches": 2
  }
    ```

    - Wildcard Query
    
      - 支持？*两种通配符
      - Avoid beginning patterns with `*` or `?`
  - boost:用于增加或减少相关性分数
      - case_insensitive [7.10.0]:true->允许进行不区分大小写的匹配，默认值为false
    ```
      {
        "query": {
          "wildcard": {
            "user.id": {
              "value": "ki*y",
              "boost": 1.0,
              "rewrite": "constant_score"
            }
          }
        }
      }
      ```

##### Constant Score 转为Filter

- 将Query转成Filter,忽略TF-IDF计算，避免相关性算分的开销
- Filter可以有效利用缓存
- 即便是对keyword进行term查询，同样会进行算分，可以将查询转成Filtering,取消相关性算分的环节，以提升性能

```json
POST /products/_search
{
    "explain":true,
    "query":{
        "constant_score":{
            "filter":{
                "term":{
                    "productID.keyword":"XHDK-A-1293-#FJ3"
                }
            }
        }
    }
}
```

#### 基于全文的查询

- 查询方式：Match Query / Match Phrase Query / Query String Query

- 特点

  - 索引和搜索时都会进行分词，查询字符串先传递到一个合适的分词器，然后生成一个供查询的词项列表
  - 查询时，会先对输入的查询进行分词，然后每个词项逐个进行底层的查询，最终将结果进行合并。并为每个文档生成一个算分。--例如：查询“Matrix reloaded",会查到包括Matrix或者reload的所有结果

  ![image-20210426141929899](C:\Users\wuym7\AppData\Roaming\Typora\typora-user-images\image-20210426141929899.png)

#### 结构化搜索

##### 结构化数据

- 结构化搜索是指对结构化数据的搜索
  - 日期、布尔类型、数字都是结构化的
- 文本也可以是结构化的
  - 如彩色笔可以有离散的颜色集合：红、绿、蓝
  - 一个博客可能被标记了标签，例如：分布式 和 搜索
  - 电商网站上的商品都有UPCs（通用产品码 Universal Produce Codes)或其他的唯一标识，它们都需要遵从严格规定的、结构化的格式

#### Query Context & Filter Context

- 高级搜索功能：支持多项文本输入，支持多个字段进行搜索
- 在ElasticSearch中，有Query和Filter两种不同的Context
  - Query Context:相关性算分
  - Filter Context:不需要算分（Yes or No),可以利用Cache,获得更好的性能

##### 复合查询（bool Query)

- 一个bool查询是一个或者多个查询子句的组合，支持bool嵌套
- 子句
  - must:必须匹配，贡献算分
  - should:选择性匹配，贡献算分
  - must_not:Filter Context 必须不能匹配，不贡献算分
  - filter:必须匹配，不贡献算分
- 同一层级下的竞争字段，具有相同权重
- 通过嵌套bool查询，可以改变对算分的影响
- should算分过程
  - 查询should语句中的评分
  - 加和查询的评分
  - 乘以匹配语句的总数
  - 除以所有语句的总数

###### Disjunction Max Query

- 将任何与任一匹配的文档作为结果返回，采用字段上最匹配的评分为最终评分返回
- 通过Tie Breaker 参数调整，Tier Breaker是一个介于0~1之间的浮点数。0代表使用最佳匹配；1代表所有语句同等重要
  1. 获得最佳匹配语句的评分_score.
  2. 将其他匹配语句的评分与tie_breaker相乘
  3. 对以上评分求和并规范化

```json
POST blogs/_search
{
    "query":{
        "dis_max":{
            "queries":[
                {
                    "match":{
                        "title":"Quick fox"
                    }
                },
                {
                    "match":{
                        "body":"Quick fox"
                    }
                }
            ],
            "tie_breaker":0.2
        }
    }
}
```

