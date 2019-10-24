---
title: mysql转换成es-rest
date: 2019-10-24 08:17:37
tags:
- es rest查询
categories:
- es
- es rest查询
---

# es查询与mysql查询转换
## es的简介
es作为一款搜索引擎,用作快速检索.本文不做es的过多分析,在实际使用中,可能会遇到mysql语句好实现,但是写成es的rest查询就比较困难的情况.因此列举一些常用转换,持续更新中.
## 所使用框架:
   **bboss-elasticsearch:**
  一款类似于mybaits,将mysql封装成以xml的方式来进行sql查询.此开源项目是将es的rest语句封装成以xml的方式进行es查询.具有很大的便利性.
[github地址](https://github.com/bbossgroups/bboss-elasticsearch)
[使用文档](https://esdoc.bbossgroups.com/#/README)
## 进入正题,案例如下(持续更新中):
注:自定义的属性,后面都会带着 `_name`,可按照实际情况替换.
### =查询
mysql:
```
select * from table_name where name = ''名字 limit 1;
```
es:
```
GET index_name /_search
{
	"size": 1,
	"query":{
		"match_phrase":{
			"pn": #[pn]
		}
	}
}
```

### between,order by,limit查询
mysql:
```
select * from table_name where time_name between start_name and end_name order by id_name desc limit 100;
```
es:
```
GET index_name /_search
{
  "size": 100,
  "sort": [
    {
      "time_name": {
        "order": "desc"
      }
    }
  ],
  "query": {
    "bool": {
      "filter": [
        {
          "bool": {
            "must": [
              {
                "range": {
                  "time_name": {
                    "from": "start_name",
                    "to": "end_name"
                  }
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```
###  min(),max()查询
mysql:
```
select min(time_name),max(time_name) from table_name;
```
es:
```
GET index_name/_search
{
  "size": 0,
  "aggs": {
    "grades_stats": {
      "stats": {
        "field": "time_name"
      }
    }
  }
}
```
es返回值如下,分析结果,提取出值:
```
  "aggregations": {
    "grades_stats": {
      "count": 10581611,
      "min": 1483228800000,
      "max": 1571184000000,
      "avg": 1527674316115.0793,
      "sum": 16165255347820800000,
      "min_as_string": "2017-01-01T00:00:00.000Z",
      "max_as_string": "2019-10-16T00:00:00.000Z",
      "avg_as_string": "2018-05-30T09:58:36.115Z",
      "sum_as_string": "292278994-08-17T07:12:55.807Z"
    }
  }
```
### in,order by,between 查询
mysql:
```
select * from table_name where name in ('名字1','名字2','名字3') and time_name between start and end limit 100;
```
es:
```
GET index_name/_search
{
	"size": #[size],
	"sort": [{
		"so_date": {
			"order": "desc"
		}
	}],
	"query": {
		"bool": {
			"filter":[
				#if(1>2)
				<!--时间范围查询,大于等于操作-->
				#end
				{
					"bool": {
						"must": [
							{
								"range": {
									"so_date": {
										"from": #[start_name],
										"to": #[end_name]
									}
								}
							}
						]
					}
				}
				#if($name && $name.size() > 0),
				{
					"bool":{
						"must":[
							{
								"bool":{
									"should": [
										#foreach($nameTmp in $name)
											#if($velocityCount > 0),#end
											{
												"match_phrase":{
													"name":"$nameTmp"
												}
											}
										#end
									],
									"minimum_should_match": 1
								}
							}
						]
					}
				}
				#end
			]
		}
	}	
}
```
### and ( (a=1 and b=2) or (c=3 and d=4) )操作
### 
对象数组查询:
```
class user {
    private string name;
    private string desc;
    private string age;
}
```
mysql:
```
select * from table_name where time_name between start_name and end_name and ( (name=1 and age=2) or (name=3 andage=4) ) ;
```
es:
```
GET index_name/_search
{
	"size": #[size],
	"sort": [{
		"so_date": {
			"order": "desc"
		}
	}],
	"query": {
		"bool": {
			"filter":[
				#if(1>2)
				<!--时间范围查询,大于等于操作-->
				#end
				{
					"bool": {
						"must": [
							{
								"range": {
									"so_date": {
										"from": #[start_name],
										"to": #[end_name]
									}
								}
							}
						]
					}
				}
				#if(1>2)
				<!--对象数组查询,and ( (a=1 and b=2) or (c=3 and d=4) )操作-->
				#end
				#if( $user && $user.size() > 0 )
					,{
						"bool":{
							"should":[
								 #if($user && $user.size() > 0)
									 #foreach($userTemp in $user)
										 #if($velocityCount > 0),#end
										 {
											 "bool":{
												"must":[
													{
														"match_phrase":{
															"name":"$userTemp.name"
														}
													}
													#if($userTemp.productDetail),
													{
														"match_phrase":{
															 "age":"$userTemp.age"
														}
													}
													#end
												]
											 }
										 }
									 #end
								 #end
							]
						}
					}
				#end
			]
		}
	}	
}
```
### like查询,并对查询结果去重
mysql:
```
select distinct name from table_name where name like '名字%' 
```
es:
```
GET index_name/_search
{
	"size": 0,
	"query":{
		"wildcard":{
			"name":{
				"value": #[名字*]
			}
		}
	 },
	"aggs": {
		"distinct": {
			"terms": {
				"field": "name.keyword"
			}
		}
	}
}
```
对聚合结果进行分析,取出所要的值.
### 去重查询
mysql:
```
select distinct name from table_name;
```
es:
```
GET index_name/_search
{
	"size": 0,
	"aggs": {
		"distinct": {
			"terms": {
				"field": "name.keyword"
			}
		}
	}
}
```
### 去重模糊查询,返回多个字段
mysql:
```
select id_name,name form table_name where name like "名字%" group by name desc ,id_name desc limit 10;
```
es:
```java
GET index_name/_search
{
           "size": 10,
            "sort": [{
                "name.keyword": {
                    "order": "desc"
                }
            }],
            "query":{
                "wildcard":{
                    "name.keyword":{
                        "value": #[var]
                    }
                }
             },
          "aggs": {
            "type":{
              "terms": {
                "field": "id_name.keyword",
                "size": 10
              },
              "aggs": {
                "redistinct": {
                  "top_hits": {
                    "sort": [{
                      "id_name.keyword": {"order": "desc"}
                    }],
                    "size": 1
                  }
                }
              }
            }
          }
 }
```