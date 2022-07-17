# Mongo内嵌数组按条件查询指定字段中的记录


由于之前对mongo缺乏了解，在工作中，遇到了返回特定字段的内嵌查询场景，因此记录下此场景下的解决方案。

<!--more-->

### 背景

{{< admonition type="info" title="背景">}}
例如一篇文章有多个评论，要筛选出满足特定条件的评论，但只返回评论字段。
```json
{
  "id": "xxx",
  "name": "测试的文章名",
  "comments": [
    {
      "name": "用户名1",
      "type": "好评",
      "conteng": "评论的内容1"
    },
    {
      "name": "用户名2",
      "type": "差评",
      "conteng": "评论的内容2"
    },
    {
      "name": "用户名3",
      "type": "好评",
      "conteng": "评论的内容3"
    }
  ]
}
```
示例如上所示，但实际应用远比想象的复杂且量大。因此实际场景，只想返回指定的comments中满足条件的数据。
{{< /admonition >}}

### 解决方案

```java
//1. 指定查询主文档
MatchOperation match1 = Aggregation.match(Criteria.where("id").is("xxx"));
//2. 拆分内嵌文档(将数组中的每一个元素转为每一条文档)
UnwindOperation unwind = Aggregation.unwind("comments");
//3. 指定查询子文档
MatchOperation match2 = Aggregation.match(Criteria.where("comments.type").is("好评"));
//4. 限制查询条数
LimitOperation limit = Aggregation.limit(1);
//5. 指定投影，返回哪些字段
ProjectionOperation project = Aggregation.project("comments");
//创建管道查询对象
Aggregation aggregation = Aggregation.newAggregation(match1, unwind, match2, limit, project);
AggregationResults<BasicDBObject> results = mongoTemplate.aggregate(aggregation, "t_table_name", BasicDBObject.class);
```

### Aggregation

数据通过条件限制传递到管道中的下一阶段，见[Aggregation](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/)文档。以下是本解决方案中用到的限制条件：

* [**$unwind**](https://docs.mongodb.com/manual/reference/operator/aggregation/unwind/):将数组中的每一个元素转为每一条文档。
* [**$match**](https://docs.mongodb.com/manual/reference/operator/aggregation/match/):过滤文档，条件查询。
* [**$project**](https://docs.mongodb.com/manual/reference/operator/aggregation/project/):指定投影，返回指定字段。
* [**$limit**](https://docs.mongodb.com/manual/reference/operator/aggregation/limit/):限制返回查询条数。

