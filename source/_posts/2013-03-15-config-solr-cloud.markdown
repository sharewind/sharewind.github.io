---
layout: post
title: "config solr cloud"
date: 2013-03-15 16:37
comments: true
categories: Solr
---

1.配置solr.xml 文件

solr.xml

	<?xml version="1.0" encoding="UTF-8" ?>
	<solr persistent="true" sharedLib="lib">
	  <cores defaultCoreName="photos" adminPath="/admin/cores" zkClientTimeout="${zkClientTimeout:15000}" hostPort="${jetty.port:}" hostContext="solr">
	    <core name="photos" loadOnStartup="true" instanceDir="photos\" dataDir="${dataDir}\photos" transient="false"  />
	    <core name="users"  loadOnStartup="true" instanceDir="users\"  dataDir="${dataDir}\users" transient="false" />
	  </cores>
	</solr>


参数：  
${jetty.port:} jetty 启动的端口  
${dataDir} 在启动时，指定数据存放的目录。

2.切分为两个分片的solr 启动脚本 

cd ./solr-4.1.0/example
java -Dsolr.solr.home=../../cloudatlas/solr/ -DdataDir=o:/opt/data/solr8983 -Djetty.port=8983 -Dbootstrap_conf=true -DzkRun -DnumShards=2 -jar start.jar

参数说明：  
-Dsolr.solr.home 指定solr 的home目录，即solr配置文件所在的目录。  
-DdataDir 指定数据文件的目录  
-Dbootstrap_conf=true 指定此参数才能分别上传 photos，users 的schema 配置到Zookeeper  
-DzkRun 启动Zookeeper
-DnumShards shard 的个数


其它的启动脚本 

cd ./solr-4.1.0/example
java -Dsolr.solr.home=../../cloudatlas/solr/ -DdataDir=o:/opt/data/solr8900 -Djetty.port=8900 -DzkHost=localhost:9983 -jar start.jar

cd ./solr-4.1.0/example
java -Dsolr.solr.home=../../cloudatlas/solr/ -DdataDir=o:/opt/data/solr7574 -Djetty.port=7574 -DzkHost=localhost:9983 -jar start.jar

cd ./solr-4.1.0/example
java -Dsolr.solr.home=../../cloudatlas/solr/ -DdataDir=o:/opt/data/solr7500 -Djetty.port=7500 -DzkHost=localhost:9983 -jar start.jar

以上，要分别指定不同的dataDir 和 jetty.port。

#### 参考资料

- [SolrCloud](http://wiki.apache.org/solr/SolrCloud)- Solr Wiki





