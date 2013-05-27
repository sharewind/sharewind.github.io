---
layout: post
title: "MongoDB的复合索引"
date: 2012-12-06 21:52
comments: true
categories: MongoDB
---

MongoDB 支持创建包含一个文档内的多个字段的复合索引。

例如某个主题活动的documents：

db.topic_photos

	{
	  "_id" : ObjectId("50b664372115fb21eca4b2b1"),
	  "created_at" : ISODate("2012-11-28T19:21:27.528Z"),
	  "featured" : true,
	  "featured_at" : ISODate("2012-12-06T10:17:15.565Z"),
	  "featured_by" : NumberLong(234234),
	  "last_liked" : ISODate("2012-12-06T13:51:29.742Z"),
	  "likes" : 386,
	  "photo_id" : NumberLong("41136819156557824"),
	  "topic_id" : NumberLong("38362556125028352"),
	  "user_id" : NumberLong(3532432)
	}

为了查询出该主题活动最受欢迎的图片，可以建立包含topic_id ,likes 和 created_at 字段的复合索引。

	db.topic_photos.ensureIndex({"topic_id":1,"likes":-1,"created_at":-1})

任何使用索引中的前缀字段的查询都能命中复合索引。（Compound indexes support queries on any prefix of the fields in the index. ）举个例子，使用topic_id 或者是 topic_id 和  likes 的查询都能命中索引。**然而，以下查询的情况无法命中索引：**
	
- 只用 likes 查询
- 只用 created_at 查询
- 只用 likes 和 created_at 查询
- 只用 topic_id 和 created_at 查询

当创建一个复合索引时，紧随索引字段的数字会指定索引的排序方式。1为升序，-1 为倒序。按何种方式排序并不影响随机访问，但却对利用复合索引进行带排序的查询很重要。

索引字段的顺序也非常关键。在上面的例子中，这个索引会包括首先按照第一个字段topic_id的值 排序，然后再按照likes 的值排序，最后才是按时间created_at 排序。

索引前缀(Index prefixes) 必须是索引字段的子集。例如，创建了索引{a:1, b:1, c:1},使用{a:1} 和 {a:1, b:1} 都是该索引的前缀。

**提示**：不用担心查询时字段的顺序。如果写查询语句 find({b:1,a:1}) 还是能够命中索引{a:1, b:1, c:1}。

总结

MongoDB的复合索引采用索引字段的前缀匹配，因此创建复合索引时，索引字段的顺序或 字段本身的排序非常重要。可以使用explain 查看 查询执行时是否命中索引。

**对比**  
例如在MySQL 建立 topic_id,likes,created_at 三个字段的复合索引。  
能够命中索引的查询：

- 单独查询topic_id
- 查询topic_id and likes
- 查询topic_id and created_at
- 查询所有列

不能命中索引的查询：

- 单独查询likes
- 单独查询 created_at
- 查询 likes 和 created_at


参考资料

- [Indexing Overview - MongoDB Manual](http://docs.mongodb.org/manual/core/indexes/)
- [关于MySQL中复合索引优化](http://leyteris.iteye.com/blog/825799)

