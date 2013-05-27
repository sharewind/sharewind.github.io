---
layout: post
title: "图片排行算法"
date: 2012-12-03 11:13
comments: true
categories: Algorithm
---

## 图片排行算法，如何生成所谓的热门图片 ##

### ● 相关的因子

1. 图片的上传时间 (created_at)
2. 图片的收藏数(likes)
3. 最后收藏时间(last_liked)
4. 图片上传者的权威性(认证用户？用户等级？ user_rank) （传图越多的用户，用户等级比例越高）
5. 图片的评论数(comment_count)
6. 最后评论时间(last_commented)
7. 图片的浏览数(page_view)
8. 图片被分享的次数(share_count)
9. 图片最后被分享的时间(last_shared)
10. 图片被转存的次数(copy_count)
11. 图片的质量(文件大小、尺寸大小)
12. 图片的分类（图片用户所在的标签：摄影用户优先在摄影标签中排名）


### ● 计算公式

参考：stackoverflow 的算法 
[http://www.ruanyifeng.com/blog/2012/03/ranking_algorithm_stack_overflow.html](http://www.ruanyifeng.com/blog/2012/03/ranking_algorithm_stack_overflow.html)

为了简化因子，我们把所有最后有相关更新时间的因子都视为一次投票行为 last_voted

时间因素
	age = Math.round((now  - created_at) /3600) 实现按小时排行
	last_voted = Math.round((now  - last_voted) /3600)

用户等级的因素：上传的图片数，原创数，认证用户，注册时间，活跃度。
在暂时没有数据的情况下，全部初始化为1.

图片的质量根据尺寸相对标准尺寸计算，其中800 * 600 是定义的标准尺寸。
	photo_quality = photo_width * photo_height /800 * 600


图片的上升因子： 
	dividend = log10(page_view * 4)  +  (likes + copy_count + comment_count  + share_count  * 5) 
				+ user_rank + photo_quality 
因为分享是个非常主动行为，被分享过去说明希望自己社交圈子里的朋友也看到。

时间衰减：
	divisor = pow(  ((age + 1) - (age - last_voted)/2) ,  1.5    )

最后得分：
	score = dividend / divisor


### ● 更新机制

1. 实时更新：有触发上述因子 就马上更新计数？并发修改问题？
2. 定时更新：把有触发过上述因子的图片，放入一个队列，过一个小时取出队列里的图片进行计算，然后更新。
3. **问题是那些在一小时内，或者说一天内没有进行操作过的图片，不会随着时间衰减？**

### ● 存储机制

由于原有的数据没有这些信息，如果需要记录可能需要修改表结构，为了省去这个步骤。
可以采用mongodb 来存储分数与排名信息，所有图片的分数初始化为0.
