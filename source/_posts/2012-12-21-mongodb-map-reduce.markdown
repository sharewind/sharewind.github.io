---
layout: post
title: "MongoDB MapReduce"
date: 2012-12-21 17:09
comments: true
categories: MongoDB
---

**MapReduce**  

db.collection.mapReduce()的用法：

	db.collection.mapReduce(
                         <mapfunction>,
                         <reducefunction>,
                         {
                           out: <collection>,
                           query: <document>,
                           sort: <document>,
                           limit: <number>,
                           finalize: <function>,
                           scope: <document>,
                           jsMode: <boolean>,
                           verbose: <boolean>
                         }
                       )


**坑**：  
1. MapReduce 在map funtion 中 emit 相应的key 只有一个文档命中的时候，不会执行reduce方法，需要在finailize 方法中进行处理。  
2. reduce 方法针对相同的key 有可能多次执行，reduce 计算的结果只和当前传入的values 有关系。

现有有个表topic_photos结构如下：

	{
	  "_id" : ObjectId("50b223832115fb21eca4a485"),
	  "created_at" : ISODate("2012-11-25T13:56:19.548Z"),
	  "last_liked" : ISODate("2012-12-10T08:44:49.276Z"),
	  "likes" : 6,
	  "photo_id" : NumberLong("39967821345988608"),
	  "topic_id" : NumberLong("38362556125028352"),
	  "user_id" : NumberLong(12449)
	}

现在我们要选出被likes最多的用户，以及该用户likes最多那张照片及该照片的likes 数。

	public void groupPhotosByUser(long topicId, long startTime, long endTime){

		String map = " function(){ "
				+ "		var key = this.user_id; " // 以用户的user_id 为 key
				+ "		value={photo_id:this.photo_id,likes:this.likes,photo_count:1,created_at:ISODate()};" // value 包含photo_id,likes,photo_count 初始为1。
				+ "		emit(key,value); "
				+ "	};";

		String reduce = "  function (user_id, values){"
								// map reduce 是并行执行的，所以每次执行的时候都要初始化result
						+ "		var result = {user_id:user_id,photo_id:0,photo_count:0,likes:0,created_at:Date()}; "
						+ " 	for(var i=0; i<values.length; i++){ "
						+ " 		value = values[i];  			"
						+ "     	if(value > result.likes){ " 
							// 选中values中likes最多的对象，并记录likes数与 photo_id
					    + "             result.likes = value.likes; "
					    + "             result.photo_id = value.photo_id;  "
					    + "         } "
					    + "         result.photo_count += value.photo_count; "
						// 在循环中累计photo_count, 这里不是取values.length!!! 因为values个数多的时候，是分多次map_reduce执行的。
						// 例如有20个values,可能分别reduce 10个文档，再将两次reduce的结果再次reduce,这时如果直接获取 values.length 结果就为2了。
						+ " 	}; "
						+ "  	return result; "
						+ "}";

		String finalize = " function (user_id, result) { "
				+ "		if(typeof(result.created_at) == 'undefined' ){ " 
						// 用户只有一张topic_photo的情况，在这里进行初始化。
				+ "			result = {user_id:user_id,photo_count:1,photo_id:result.photo_id,likes:result.likes,created_at:ISODate()}; "
				+ "		} "
				+ "  	return result; "
				+ "}";

		DBObject query = QueryBuilder.start("topic_id").is(topicId)
						.put("created_at").greaterThanEquals(new Date(startTime))
						.lessThanEquals(new Date(endTime))
						.get();
		String outputCollection = String.format("topic_%s_users", topicId);

		DBCollection inputCollection = getMongoDB().getCollection("topic_photos");
		MapReduceCommand mapReduceCommand = new MapReduceCommand(inputCollection, map, reduce, outputCollection, OutputType.REPLACE, query);
		mapReduceCommand.setFinalize(finalize);
		inputCollection.mapReduce(mapReduceCommand);

		//TODO creat mongodb index
	}

#### MongoDB的调试

Java程序中

	static{

		// Enable MongoDB logging in general
		System.setProperty("DEBUG.MONGO", "true");

		// Enable DB operation tracing
		System.setProperty("DB.TRACE", "true");
	}	


利用printjson()函数，在MapReduce 的执行过程打印出执行日志

	var map = function(){
		var key = this.user_id;
		var value = {photo_id:this.photo_id, likes:this.likes, photo_count:1, created_at:ISODate()};
		emit(key,value);
	}
	
	var reduce = function (user_id, values){
		if (user_id == 350278) {
			printjson(values);
		}
	
		var result = {user_id:user_id,photo_id:0,photo_count:0,likes:0,created_at:Date()};
		for(var i=0; i<values.length; i++){
			var value = values[i];
			if(value > result.likes){
				result.likes = value.likes;
				result.photo_id = value.photo_id;
			}
			result.photo_count += value.photo_count;
		};
		return result;
	}
	
	var finalize = function (user_id, result) {
		if(typeof(result.created_at) == 'undefined' ){
			result = {user_id:user_id,photo_count:1,photo_id:result.photo_id,likes:result.likes,created_at:ISODate()};
		}
		return result;
	}
	
	var query = {"topic_id" : 38362556125028352}
	
	db.runCommand({"mapreduce" : "topic_photos" ,
			  "map" : map, 
			  "reduce" : reduce,
			  "finalize": finalize,
			  "query" : query,		   
			  "out" : { "replace" : "topic_38362556125028352_users"}, 
			  "verbose" : true
			})
	

MongoDB控制台输出

![](/pics/mongodb-map-reduce.jpg)

从上图可以看到，最后一次reduce 时，是将前两次reduce 的结果再执行 reduce 操作，所以reduce 中的 photo_count 计数，不能依赖于 values.length, 而应该从传入的参数中获取。

#### 利用Underscore.js 框架调试MapReduce方法
参见[Debugging MapReduce in MongoDB](http://gregorowicz.blogspot.com/2010/12/debugging-mapreduce-in-mongodb.html)


**Tips**

- MongoDB 最分将不同的业务进行垂直切分，存储到不到的db中，这样分别在找出慢查询及定位问题的时候更清晰。
- Pretty print in MongoDB shell:  <code>db.collection.find().pretty()</code>


#### 参考资料

- [Map-Reduce](http://docs.mongodb.org/manual/applications/map-reduce/)
- [db.collection.mapReduce()](http://docs.mongodb.org/manual/reference/method/db.collection.mapReduce/#db.collection.mapReduce)
- [Getting Started with the mongo Shell](http://docs.mongodb.org/manual/tutorial/getting-started-with-the-mongo-shell/)
- [Mongodb Mapreduce讨论及分享（平均数和唯一数统计误区）](http://www.oschina.net/question/105422_49646)

