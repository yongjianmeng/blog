# 理解MongoDB执行计划 #
最近在钻研MongoDB查询的优化，在几千万的表中查询数据，查询时间从原先的100+秒甚至超时到现在的10秒以内，有近10倍的提升，当然，个人感觉还有很大的优化空间。在这一过程，最需要感谢的必然是MongoDB提供的执行计划解析功能：  
```
cursor.explain(verbose)
```
其中，verbose是可选的参数，它影响了explain方法的**解析行为**以及**输出信息**。目前，verbose有三种模式：
1. queryPlanner （默认）
2. executionStats
3. allPlansExecution  

queryPlanner是explain方法的默认模式。

为了方便后面的实践，我们现在插入一些测试数据：
```
// 给User表插入4条数据
db.User.insertMany([
	{ userName: 'Jonas', age: 25, title: 'A' },
	{ userName: 'Mike', age: 16, title: 'A' },
	{ userName: 'Jay', age: 69, title: 'B' },
	{ userName: 'Rich', age: 31, title: 'C' }
]);

// 给User表在title字段上建立索引
db.User.createIndex({
  title: 1
})
```
> 在MongoDB 3.0之后，执行计划的输出格式和之前的版本有很大的变化，本文只讨论3.0版本之后的执行计划，不讨论与旧版本执行计划的兼容信息。

## queryPlanner 模式 ##
在queryPlanner模式下，查询语句并不会被真正地执行，而是会使用[查询优化器](https://docs.mongodb.com/manual/core/query-plans/ "查询优化器")去评估并选择出最优执行计划（winning plan），所以如果我们并不需要查询语句真正地执行，只想要知道MongoDB将如何执行这个查询语句时可以使用该模式。
```
db.User.find({ age: { $gt: 30 } }).explain('queryPlanner')
db.User.find({ age: { $gt: 30 } }).explain() 
```
上述两个语句都是在queryPlanner模式下执行explain方法进行解析（'queryPlanner'为默认参数），通过运行该语句，我们可以得到下面的解析结果：
*注意：虚线及后面的备注为后期添加上去的注释*
```
{ 
    "queryPlanner" : {  --------------------> 执行queryPlanner模式返回的结果
        "plannerVersion" : 1.0,
        "namespace" : "test.User", ---------> 查询语句所查询的表 <database>.<collection>
        "indexFilterSet" : false,  ---------> 查询语句是否在查询模型（query shape）上使用indexFilter去选用特定的索引
        "parsedQuery" : {  -----------------> 被解析后的查询语句
            "$and" : [
                {
                    "title" : {
                        "$eq" : "B"
                    }
                }, 
                {
                    "age" : {
                        "$gt" : 30.0
                    }
                }
            ]
        }, 
        "winningPlan" : {  -----------------> 最优执行计划，MongoDB用树形结构来表示执行计划
            "stage" : "FETCH",  ------------> 执行步骤名，这里是FETCH，也是最后一个步骤。不同类型的步骤包含不同的信息
            "filter" : {  ------------------> FETCH步骤特有的字段，代表过滤器
                "age" : {
                    "$gt" : 30.0
                }
            }, 
            "inputStage" : {  --------------> FETCH步骤的子步骤，也称为输入步骤，为父步骤提供数据或索引
                "stage" : "IXSCAN",  -------> 执行步骤名，这里是IXSCAN，代表扫描索引
                "keyPattern" : {  ----------> 扫描的索引，这里表示用了之前在title上建立的索引
                    "title" : 1.0
                }, 
                "indexName" : "title_1",  --> IXSCAN所使用的索引名
                "isMultiKey" : false,  -----> 是否是Multikey，如果索引建立在array上，此处将是true
                "multiKeyPaths" : {
                    "title" : [

                    ]
                }, 
                "isUnique" : false,  --------> 是否是唯一索引
                "isSparse" : false,  --------> 是否是稀疏索引
                "isPartial" : false, --------> 是否是部分索引
                "indexVersion" : 2.0, 
                "direction" : "forward",  ---> 查询顺序，forward是顺序，另外一个值是"backward"，代表逆序
                "indexBounds" : {  ----------> IXSCAN所扫描的索引范围
                    "title" : [
                        "[\"B\", \"B\"]"
                    ]
                }
            }
        }, 
        "rejectedPlans" : [  -----------------> 非最优而被查询优化器拒绝的执行计划，具体信息与winningPlan相同

        ]
    }, 
    "serverInfo" : {  ------------------------> 服务器信息，对于分片的表，返回对每一个分片所在服务器的信息
        "host" : "LAPTOP-UN65HSDR", 
        "port" : 27017.0, 
        "version" : "3.6.2", 
        "gitVersion" : "489d177dbd0f0420a8ca04d39fd78d0a2c539420"
    }, 
    "ok" : 1.0
}

```

## executionStats 模式 ##
在executionStats模式下，**winningPlan**会被真正地执行，并返回执行过程的详细信息。
```
db.User.find({
  title: 'B',
  age: { $gt: 30 }
}).explain('executionStats')
```
注意到，上述查询语句传入explain的参数变为executionStats，通过运行该语句，我们可以得到下面的解析结果：
```
{ 
    "queryPlanner" : {
        //...上文已经解析过，故省略
    }, 
    "executionStats" : {  ------------------------> 执行executionStats模式返回的结果
        "executionSuccess" : true,  --------------> 是否执行成功
        "nReturned" : 1.0,  ----------------------> 查询返回的条数
        "executionTimeMillis" : 0.0,  ------------> 整体执行时间
        "totalKeysExamined" : 1.0,  --------------> 索引扫描次数
        "totalDocsExamined" : 1.0,  --------------> 文档扫描次数
        "executionStages" : {  -------------------> 执行计划的具体步骤
            "stage" : "FETCH",  ------------------> 执行步骤名，这里是FETCH，也是最后一个步骤。
            "filter" : {
                "age" : {
                    "$gt" : 30.0
                }
            }, 
            "nReturned" : 1.0,  ------------------> 该执行步骤返回的文档/索引条数
            "executionTimeMillisEstimate" : 0.0, -> 该执行步骤估计的执行时间
            "works" : 2.0,  ----------------------> 该执行步骤指定的工作单元（work unit）数，
            "advanced" : 1.0, 
            "needTime" : 0.0, 
            "needYield" : 0.0, 
            "saveState" : 0.0, 
            "restoreState" : 0.0, 
            "isEOF" : 1.0, 
            "invalidates" : 0.0, 
            "docsExamined" : 1.0,  ---------------> 该执行步骤文档扫描次数
            "alreadyHasObj" : 0.0, 
            "inputStage" : {
                "stage" : "IXSCAN", 
                "nReturned" : 1.0,  --------------> 该执行步骤返回的文档/索引条数
                "executionTimeMillisEstimate" : 0.0, 
                "works" : 2.0, 
                "advanced" : 1.0, 
                "needTime" : 0.0, 
                "needYield" : 0.0, 
                "saveState" : 0.0, 
                "restoreState" : 0.0, 
                "isEOF" : 1.0, 
                "invalidates" : 0.0, 
                "keyPattern" : {
                    "title" : 1.0
                }, 
                "indexName" : "title_1", 
                "isMultiKey" : false, 
                "multiKeyPaths" : {
                    "title" : [

                    ]
                }, 
                "isUnique" : false, 
                "isSparse" : false, 
                "isPartial" : false, 
                "indexVersion" : 2.0, 
                "direction" : "forward", 
                "indexBounds" : {
                    "title" : [
                        "[\"B\", \"B\"]"
                    ]
                }, 
                "keysExamined" : 1.0,  ------------> 该执行步骤索引扫描次数
                "seeks" : 1.0, 
                "dupsTested" : 0.0, 
                "dupsDropped" : 0.0, 
                "seenInvalidated" : 0.0
            }
        }
    }, 
    "serverInfo" : {
        //...上文已经解析过，故省略
    }, 
    "ok" : 1.0
}
```
> 针对工作单元（work unit），文档给出的解释是：每个查询执行步骤把它的工作分成了较小的工作单元，每个工作单元可能包含检查单个的索引、从集合中获取单个的文档、把映射应用在单个文档上、做一部分内部的记录等等。

> 其他字段，如advanced，needTime，needYield，saveState，restoreState，isEOF等在实践过程中用的比较少，解释起来也相对较为晦涩，故不作解释，具体请查看[官方文档](https://docs.mongodb.com/manual/reference/explain-results/)的解释。

## allPlansExecution 模式 ##
在executionStats模式下，所有的执行计划都会被真正地执行，包括winningPlan和rejectedPlans，并返回执行过程的详细信息。该模式的信息和executionStats模式基本一样，所以不再做重复的解析。

## Stage的类型 ##
在上述解析中，执行计划中有一些不同的执行步骤，这里总结一下常见的stage及其所代表的意义：  

| 类型             | 解释           | 备注  |
| --------------- | -------------- | ----- | 
| COLLSCAN        | 全表扫描        | 避免使用，数据量大时性能很低 | 
| IXSCAN          | 索引扫描 |  |
| FETCH           | 根据索引去检索指定文档 |  |
| SHARD_MERGE     | 将各个分片返回数据进行合并 |  | 
| SORT            | 在内存中进行了排序 | 避免使用，不使用索引时会在内存中排序，数据量大时性能很低 | 
| LIMIT           | 限制返回数 |
| SKIP            | 跳过指定个数的文档 | 使用不恰当时容易导致性能问题 |
| IDHACK          | 针对_id进行查询 |
| SHARDING_FILTER | 通过mongos对分片数据进行查询 |
| COUNT           | 进行count运算 |
| COUNTSCAN       | count不使用索引进行count时的stage返回 | 数据量大时性能很低 |
| COUNT_SCAN      | count使用了索引进行count时的stage返回 |
| SUBPLAN         | 未使用到索引的$or查询的stage返回 | 未用到索引的$or查询容易导致性能问题 |
| TEXT            | 使用全文索引进行查询时候的stage返回 |
| PROJECTION      | 限定返回字段时候stage的返回 |

## 总结 ##
当遇到性能问题时，查看执行计划是一个非常好的方式去做性能调优，学会查看执行计划也是深入学习MongoDB的必备技能，有问题欢迎提交 PR 讨论。