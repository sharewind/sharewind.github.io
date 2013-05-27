---
layout: post
title: "solr 4.1 install guide"
date: 2013-03-08 09:29
comments: true
categories: Solr Lucene
---


1.从[Solr的官方网站](http://lucene.apache.org/solr/)下载Solr 4.1的安装包.

2.解压solr-4.1.0.zip 到安装目录。

3.将 solr-4.1.0目录下的example复制成目标索引项目的名称 cloudatlas.

4.进行 solr-4.1.0\cloudatlas\solr目录，复制 collection1 为索引文档的名称。
   collection1 --> photos, users.

5.在 solr-4.1.0\cloudatlas\solr\solr.xml文件中，配置两个core 实例。

	<?xml version="1.0" encoding="UTF-8" ?>
	<solr persistent="true">
	  <cores defaultCoreName="photos" adminPath="/admin/cores" zkClientTimeout="${zkClientTimeout:15000}" hostPort="8983" hostContext="solr">
	    <core name="photos" loadOnStartup="true" instanceDir="photos\" transient="false" />
	    <core name="users"  loadOnStartup="true" instanceDir="users\" transient="false" />
	  </cores>
	</solr>

6.配置IK中文分词。

	
	在 [googlecode](https://code.google.com/p/ik-analyzer/) 下载IK分词器的 jar 包。	在 solr-4.1.0\cloudatlas\solr 下创建 lib 目录，并将下载的IK分词器jar放到该目录中。

	在 solr-4.1.0\cloudatlas\solr\photos\conf\solrconfig.xml 配置lib 目录  
	
	<lib dir="../lib" />

	在 solr-4.1.0\cloudatlas\solr\photos\conf\schema.xml 配置中文分词的field_type.

 	<fieldType name="text_cn" class="solr.TextField" positionIncrementGap="100">
      <analyzer type="index">
        <tokenizer class="org.wltea.analyzer.solr.IKTokenizerFactory" useSmart ="false"/>
      </analyzer>
      <analyzer type="query">
        <tokenizer class="org.wltea.analyzer.solr.IKTokenizerFactory" useSmart ="false"/>
      </analyzer>
    </fieldType>
 

	采用中文分词的field配置：

		<field name="photo_desc" type="text_cn" indexed="true" stored="true"/>

7.在solr.xml 中注释 elevator 的配置（因为我们的schema 字段配置与 elevate中的字段配置有冲突）
	
	  <!-- Query Elevation Component
	
	       http://wiki.apache.org/solr/QueryElevationComponent
	
	       a search component that enables you to configure the top
	       results for a given query regardless of the normal lucene
	       scoring.
	    -->
	  <searchComponent name="elevator" class="solr.QueryElevationComponent" >
	    <!-- pick a fieldType to analyze queries -->
	    <str name="queryFieldType">string</str>
	    <str name="config-file">elevate.xml</str>
	  </searchComponent>
	
	  <!-- A request handler for demonstrating the elevator component -->
	  <requestHandler name="/elevate" class="solr.SearchHandler" startup="lazy">
	    <lst name="defaults">
	      <str name="echoParams">explicit</str>
	      <str name="df">text</str>
	    </lst>
	    <arr name="last-components">
	      <str>elevator</str>
	    </arr>
	  </requestHandler>


8.配置索引的文档schema.

photos的schema.xml 配置

	<?xml version="1.0" encoding="UTF-8" ?>	
	<schema name="photos" version="1.5">
	
	 <fields>        
	   <field name="id" type="long" indexed="true" stored="true" required="true" multiValued="false" /> 
	   <field name="user_id" type="long" indexed="true" stored="true" omitNorms="true" />   
	   <field name="folder_id" type="long" indexed="true" stored="true" omitNorms="true" />   
	   <field name="folder_name" type="text_cn" indexed="true" stored="true"/>
	   <field name="folder_desc" type="text_cn" indexed="true" stored="true"/>
	   <field name="photo_desc" type="text_cn" indexed="true" stored="true"/>
	   <field name="farm" type="string" indexed="false" stored="true" omitNorms="true"/>
	   <field name="bucket" type="string" indexed="false" stored="true" omitNorms="true"/>
	   <field name="storage_keys" type="string" indexed="false" stored="true" omitNorms="true"/>
	   <!-- catchall field, containing all other searchable text fields (implemented
	        via copyField further on in this schema  -->
	   <field name="text" type="text_cn" indexed="true" stored="false" multiValued="true"/>
	   <field name="_version_" type="long" indexed="true" stored="true" multiValued="false" />   
	 </fields>
	
	
	 <!-- Field to use to determine and enforce document uniqueness. 
	      Unless this field is marked with required="false", it will be a required field
	   -->
	 <uniqueKey>id</uniqueKey>
	
	   <copyField source="folder_name" dest="text"/>
	   <copyField source="folder_desc" dest="text"/>
	   <copyField source="photo_desc" dest="text"/>  
	 
	  <types>
	
	    <!-- The StrField type is not analyzed, but indexed/stored verbatim. -->
	    <fieldType name="string" class="solr.StrField" sortMissingLast="true" />
	
	    <!-- boolean type: "true" or "false" -->
	    <fieldType name="boolean" class="solr.BoolField" sortMissingLast="true"/>
	
	    <fieldType name="int" class="solr.TrieIntField" precisionStep="0" positionIncrementGap="0"/>
	    <fieldType name="float" class="solr.TrieFloatField" precisionStep="0" positionIncrementGap="0"/>
	    <fieldType name="long" class="solr.TrieLongField" precisionStep="0" positionIncrementGap="0"/>
	    <fieldType name="double" class="solr.TrieDoubleField" precisionStep="0" positionIncrementGap="0"/>
		
		<fieldType name="text_cn" class="solr.TextField" positionIncrementGap="100">
	      <analyzer type="index">
	        <tokenizer class="org.wltea.analyzer.solr.IKTokenizerFactory" useSmart ="false"/>
	      </analyzer>
	      <analyzer type="query">
	        <tokenizer class="org.wltea.analyzer.solr.IKTokenizerFactory" useSmart ="false"/>
	      </analyzer>
	    </fieldType>
	 </types>
	 
		<!-- field for the QueryParser to use when an explicit fieldname is absent -->
		<defaultSearchField>text</defaultSearchField>
	
		<!-- SolrQueryParser configuration: defaultOperator="AND|OR" -->
		<solrQueryParser defaultOperator="OR"/>
		
	</schema>
	
users的schema 配置

	<?xml version="1.0" encoding="UTF-8" ?>

	<schema name="users" version="1.5">
	
	 <fields>        
	   <field name="id" type="long" indexed="true" stored="true" required="true" multiValued="false" /> 
	   <field name="nickname" type="text_cn" indexed="true" stored="true"/>
	   <field name="text" type="string" indexed="true" stored="false" multiValued="true"/>
	   <field name="_version_" type="long" indexed="true" stored="true" multiValued="false" />
	 </fields>
	
	
	 <!-- Field to use to determine and enforce document uniqueness. 
	      Unless this field is marked with required="false", it will be a required field
	   -->
	 <uniqueKey>id</uniqueKey>
	
	<!--
	   <copyField source="folder_name" dest="text"/>
	   <copyField source="folder_desc" dest="text"/>
	   <copyField source="photo_desc" dest="text"/>  
	 -->
	 
	  <types>
	    <!-- The StrField type is not analyzed, but indexed/stored verbatim. -->
	    <fieldType name="string" class="solr.StrField" sortMissingLast="true" />
	
	    <!-- boolean type: "true" or "false" -->
	    <fieldType name="boolean" class="solr.BoolField" sortMissingLast="true"/>
	
	    <fieldType name="int" class="solr.TrieIntField" precisionStep="0" positionIncrementGap="0"/>
	    <fieldType name="float" class="solr.TrieFloatField" precisionStep="0" positionIncrementGap="0"/>
	    <fieldType name="long" class="solr.TrieLongField" precisionStep="0" positionIncrementGap="0"/>
	    <fieldType name="double" class="solr.TrieDoubleField" precisionStep="0" positionIncrementGap="0"/>
		
		<fieldType name="text_cn" class="solr.TextField" positionIncrementGap="100">
	      <analyzer type="index">
	        <tokenizer class="org.wltea.analyzer.solr.IKTokenizerFactory" useSmart ="false"/>
	      </analyzer>
	      <analyzer type="query">
	        <tokenizer class="org.wltea.analyzer.solr.IKTokenizerFactory" useSmart ="false"/>
	      </analyzer>
	    </fieldType>
	 </types>
	 
	 	<!-- field for the QueryParser to use when an explicit fieldname is absent -->
		<defaultSearchField>nickname</defaultSearchField>
	
		<!-- SolrQueryParser configuration: defaultOperator="AND|OR" -->
		<solrQueryParser defaultOperator="OR"/>
		
	</schema>

10.启动solr 服务 

	java -jar start.jar 


#### 参考资料
- [Solr Tutorial](http://lucene.apache.org/solr/4_2_0/tutorial.html)











