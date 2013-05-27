---
layout: post
title: "zookeepr 单机伪集群配置"
date: 2012-12-06 21:37
comments: true
categories: zookeeper
---

1. 要在单机下搭建群集，简单的就是分别解压复制N个目录。

2. 每个目录，分配配置对应的data 和 myid ，其中myid 用来表明当前启动是哪个zookeeper

3. 配置文件的群集属性

<pre>
server.1=127.0.0.1:2881:3881
server.2=127.0.0.1:2882:3883
server.3=127.0.0.1:2883:3883
server.4=127.0.0.1:2884:3884
server.5=127.0.0.1:2885:3885
</pre>

4. （我被坑的地方）每个zookeeper 服务有三个端口，

	clientPort=2181,这个用于指明给客户端应用连接的端（例如java 程序访问就是这个端口）

    server.1=127.0.0.1:2881:3881 ，其中2881 用于群集间的服务器进行数据通讯，
	3881 用于群集刚启动时 

或leader 当机时 进行选举。

完整的配置文件（单机环境，5个实例的端口都要不一样）
<pre>
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=D:/data/zookeeper/server5/data
# the port at which the clients will connect
clientPort=2185
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=127.0.0.1:2881:3881
server.2=127.0.0.1:2882:3883
server.3=127.0.0.1:2883:3883
server.4=127.0.0.1:2884:3884
server.5=127.0.0.1:2885:3885
</pre>

附上完整的配置文件 https://swtools.googlecode.com/git/linux/zookeeper/zookeeper.zip
