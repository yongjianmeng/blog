# 记一次使用 MongoDB Cursor Snapshot 遇到的坑 #

## 场景 ##

最近在做一个业务，其中有一个功能是：根据查询语句找出所有符合的交易数据，然后把这些交易数据更新。由于系统要求保存交易数据的所有历史记录，所以更新操作有所不同：系统都会先拷贝每一条要更新的交易数据，然后在之前那条交易数据上加一个终结的时间（endDate），代表这条数据已经在endDate的时候失效了，然后再在拷贝的那条交易数据上做更新操作，并更新它的startDate为当前时间，代表这条数据已经在startDate的时候开始生效了，最后再把更新后的数据插入数据库。例如：
```
// 更新之前的数据
{
    _id: 1,
    name: 'Jack',
    value: 'x',
    startDate: ISODate('2018-09-01T06:01:17.171Z')
}

------ 在 updateDate = 2018-09-15T04:23:15.753Z 时更新_id为1的交易数据，把它的value更新为'y' ------

// 更新操作之后的数据

// 旧数据被加上了一个endDate，endDate为更新的时间
{
    _id: 1,
    name: 'Jack',
    value: 'x',
    startDate: ISODate('2018-09-01T06:01:17.171Z'),
	endDate: ISODate('2018-09-15T04:23:15.753Z')
}

// 插入一条新数据，value被更新为'y'，startDate为更新的时间
{
    _id: 2,
    name: 'Jack',
    value: 'y',
    startDate: ISODate('2018-09-15T04:23:15.753Z')
}
```

嗯。。。第一次看这个功能时，感觉实现起来很简单，后来发现有的时候要更新的数据量太大了，那好办，引入MongoDB Cursor，做批量的操作：
```
var MAX_BATCH_COUNT = 1000;

// query对应的交易数据有几万条
var stream = db.collection('trades').find(query).stream();

var sumUpdateCount = 0; // 被更新的交易数据总条数
var batchTrades = []; // 一个批次的交易数据
var batchCount = 0;  // 该批次交易数据的条数

stream.on('data', function(trade) {
	batchTrades.push(trade);
	if (batchCount === MAX_BATCH_COUNT) {
		sumUpdateCount += MAX_BATCH_COUNT;
		stream.pause(); // 先暂停stream
		expireAndInsertTrades(batchTrades) // expireAndInsertTrades就是上面描述的更新操作
			.then(function (err) {
				// ...
				batchTrades = [];
				batchCount = 0;
				stream.resume(); // 继续stream
			});
	}
});

stream.on('end', function() {
    if (batchTrades.length) {
		sumUpdateCount += batchTrades.length;
		// 把剩余的交易数据（未满MAX_BATCH_COUNT）也执行expireAndInsertTrades
	}
});

```

看上去一切都很完美，然后就跑一个试试，1分钟。。2分钟。。5分钟。。20分钟过去了，还是没结束，不对啊？更新几万条数据应该不不至于这么慢吧？然后开启调试模式，发现更新操作不会终止，而是无限循环下去，也就是说stream的data事件一直被触发，而end事件没有被触发（说明一直有数据来），sumUpdateCount 都已经更新到好几十万了，但是query对应的交易数据只有几万条呀？为什么会更新到好几十万而且还停不下来呢？

查了一下官方文档，发现MongoDB的Cursor会因为文档大小变大而重新去读取一遍（intervening write operations result in a move of the document due to the growth in document size，我理解为：一些内部的写操作把文档大小变大，导致文档的移动），因为我每次的操作都会添加一个endDate字段，所以每次更新操作都伴随着文档的移动，都被会Cursor再次读到，所以会无限地执行下去。

那么有什么解决方法呢？可以使用Cursor提供的[snapshot](http://www.mongoing.com/docs/reference/method/cursor.snapshot.html)，

> Append the snapshot() method to a cursor to toggle the “snapshot” mode. T**his ensures that the query will not return a document multiple times**, even if intervening write operations result in a move of the document due to the growth in document size.

但是使用snapshot还是有限制的。Snapshot必须在获取文档之前设定，不能在开始获取文档后再设定，并且**snapshot不能用于分片（sharded）的集合**。

行，它的限制我都满足，那我就加个snapshot试试：
```
// ... 省略

var stream = db.collection('trades').find(query).stream().snapshot();

// ... 省略
```

调试了一下，发现还是会无限地执行下去。那是什么原因呢？仔细再看了一下官方文档，发现snapshot还有两条限制！第一个是snapshot**不能保证隔离插入操作和删除操作**，而要做的功能里却有插入操作，所以每次插入数据都没有被Cursor隔离起来，又会被重新读取，导致无限循环下去。这是snapshot的限制，那只能通过更改query来**自己做隔离**了。

在系统的交易数据里都有startDate这个字段，代表数据开始生效时间，那么后面插入的新数据的startDate肯定是晚于更新事件updateDate的，因此可以在query中通过增加 { startDate: { $lt: updateDate } } 来做隔离。那么有的小伙伴可能会问了，如果数据没有可用的时间来做隔离呢？如果没有时间可以用来做隔离，那么还能使用MongoDB自带的_id来做隔离，为什么呢？因为_id中是带有时间戳的，而且时间戳还是在最前面几位！

![](https://github.com/yongjianmeng/blog/blob/master/images/%E8%AE%B0%E4%B8%80%E6%AC%A1%E4%BD%BF%E7%94%A8%20MongoDB%20Cursor%20Snapshot%20%E9%81%87%E5%88%B0%E7%9A%84%E5%9D%91-0.png)

如上图所示，ObjectId 是一个12字节 BSON 类型数据，**前4个字节表示时间戳**，接下来的3个字节是机器标识码，紧接的两个字节由进程id组成（PID），最后三个字节是随机数。MongoDB中存储的文档必须有一个"_id"键。在一个集合里面，每个文档都有唯一的"_id"值，来确保集合里面每个文档都能被唯一标识。
```
var updateTimestamp = updateDate.getTime();
var updateFlagObjectId = ObjectID.createFromTime(updateTimestamp / 1000);
```
之后只要在query中添加 { objectId: { $lt: updateFlagObjectId } } 即可。

对于隔离删除操作，我是觉得如果你都将那条数据删除了，那么它也就没有存在的意义了，Cursor没有把它隔离，后面也就读取不到已经被删除的数据了，因为我这的系统没有涉及删除操作，所以也就没去深入研究如何隔离删除操作，有任何建议去解决隔离删除操作，欢迎提pull request。

> 欢迎转载，注明出处即可。