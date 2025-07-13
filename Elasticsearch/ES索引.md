![image-20230329150831050](resources\image-20230329150831050.png)



- 倒排索引`Inverted Index`

  - 词典term dictionary。
    - 同时，词典也有索引term index（词典目录，数据结构）

  - 索引posting list。

  - 疑问

    - b\b+树区别
      - 比b树多一层term index，更快锁定范围


> b+树用主键索引，叶节点存储地址 

- 联合索引`Skip List`和`bitset合并`

> 为什么叫倒排？因为不是由记录（主键）来确定属性值，而是由属性值来确定记录的位置
>



## 查询

### string search

字符简单查询

```
GET /conference/event/_search?q=+attendees:Dave&sort=reviews:desc
查询attendees必须（+：必须；-：不含）包含Dave，同时按照reviews降序排序
```

### DSL



排序、分页语法

```json
{
    "query": {
        ...
    },
    "sort": [
        {
            "reviews": {
                "order": "desc"
            }
        }
    ],
    "from": 2,
    "size": 10
}
```



### _search

- match_all

  全量查询，有分页

  ```json
  {
      "query": {
          "match_all": {
          }
      }
  }
  ```

  

- **match**

  full-text全文检索。会将输入的关键字拆解，去倒排索引里面一一匹配，只要匹配上任意一个拆解后的单词，就可以作为结果返回，放在数组 hits 中。之后文档按_score进行排序，_score字段表示文档的相似度（默认的相似度算法为BM25）。

  - `query`关键字，也可属性后直接跟关键字，这样其它参数均为默认
  - `operator`匹配逻辑，默认`or`
  
  - `boost`权重

```json
{
    "query": {
        "match": {
            "属性": {
                "query": "片 颠茄",
                "operator": "and"
            }
        }
    }
}
```

- bool

  分为若干个条件集合

  - `should`应有
  - `must`
  - `must_not`

​		集合对象为上文`match`

```json
{
    "query": {
        "bool": {
            "must": [
                {
                    "match": {
                        "name": {
                            "query": "苯 片",
                            "operator": "and"
                        }
                    }
                }
            ],
            "should": [
                {
                    "match": {
                        "name": {
                            "query": "苯溴马隆片",
                            "boost": "3"
                        }
                    }
                },
                {
                    "match": {
                        "name": {
                            "query": "复方利血平氨苯蝶啶片",
                            "boost": "10"
                        }
                    }
                }
            ],
            "must_not": [
                {
                    "match": {
                        "name": "盐酸"
                    }
                }
            ]
        }
    }
}
```

- multi_match

  一个关键词匹配多个属性

  涉及的匹配策略

  - best_fields
    doc的某个属性匹配尽可能多的关键词, 那么这个doc会优先返回.
  - most_fields
    某个关键词匹配doc尽可能多的属性, 那么这个doc会优先返回.
  - cross_fields
    跨越多个field搜索一个关键词.

```json
{
    "query": {
        "multi_match": {
            "query": "苯 片",
            "type": "best_fields",
            "fields": ["name", "generalName"],
            "operator": "and"
        }
    }
}
```

- match_phrase、match_phrase_prefix

  **短语搜索**，不像match会把搜索词拆分开。

  - `slop`允许间隔，默认不允许即0

```json
{
    "query": {
        "match_phrase": {
            "name": {
                "query": "唑片",
                "slop": 1
            }
        }
    }
}
```

​		但短语搜索性能很低（为什么），一般先全文再短语

- rescore

  使用rescore后置处理

  ````json
  {
      "query": {
          "match": {
              "name": "开塞露（青岛）★ 盐酸左西替利嗪片"
          }
      },
      "rescore": {
          "window_size": 20,
          "query": {
              "rescore_query": {
                  "match_phrase": {
                      "name": {
                          "query": "盐酸左西替利嗪片"
                      }
                  }
              }
          }
      }
  }
  ````

- fuzzy

  模糊搜索，允许纠正拼写错误。

  - fuzziness，允许纠正字符数

  中文句子好像不适配，没达到预期效果，即使完全匹配也搜索不到

```json
{
    "query": {
        "fuzzy": {
            "name": {
                "value":"槐耳颗粒",
                "fuzziness": 2
            }
        }
    }
}
```

- term

​		结构化查询，查询词不会被分词，类似等于

````json
{
    "query": {
        "term": {
            "generalName": {
                "value": "开塞露",
                "boost": 1
            }
        }
    }
}
````

- filter

  ```json
  {
      "query": {
          "bool": {
              "must": [
                  {
                      "match": {
                          "attendees": "Clint"
                      }
                  }
              ],
              "filter": {
                  "range": {
                      "reviews": {
                          "gte": 2
                      }
                  }
              }
          }
      }
  }
  ```

  

## 聚合

- aggs
  - size，展示hits的数量

```json
{
  "size": 0,
  "aggs": {
    "max_sales_color": {
      "terms": {
        "field": "color.keyword"
      }
    }
  }
}
```

聚合属性，求平均值

```json
{
    "size": 0,
    "aggs": {
        "group_color": {
            "terms": {
                "field": "color.keyword"
            },
            "aggs": {
                "avg_price": {
                    "avg": {
                        "field": "price"
                    }
                }
            }
        }
    }
}
```

## 分词

​	_analyze

- 分词器analyzer
  - standard，ES默认分词器，按单词分类并进行小写处理
  - simple，按照非字母切分，然后去除非字母并进行小写处理
  - stop，按照停用词过滤并进行小写处理，停用词包括the、a、is
  - whitespace，按照空格切分
  - language，据说提供了30多种常见语言的分词器
  - patter，按照正则表达式进行分词，默认是\W+ ,代表非字母
  - keyword，不进行分词，作为一个整体输出

```json
{
    "analyzer": "standard",
    "text": "15g（0.05%）/剂"
}
```

