---
layout: post
title: use liquibase
category: java
excerpt: liquibase是一个开源的数据库迁移工具
tags: [liquibase]
---
{% include JB/setup %}  

###### <a href="http://www.liquibase.org/">liquibase</a>是一个开源的数据库迁移工具，它和rails世界中的migration比较接近，这对大型的j2ee项目开发来说，可谓是优质的辅助。在项目开发过程中选择liquibase是有原因的。对于一个健壮的j2ee平台而言，稳定的数据库系统是不可或缺的。在此之前，我们采用的是DDM+Review的机制来管理数据库脚本的提交（DDM是一个从开发数据库导出安全脚本的工具），之后再通过人工的方式去执行。这就带来了许多隐患：对于一个数据库，我们无法知道哪些脚本执行过，我们也无法搞清楚当前数据库所处的版本（对应于代码流）。liquibase正好提供了一个解决思路。为了应对搭建数据库的不便，我们目前采用了InitialDB+liquibase的机制，这将极大地方便我们在不同的开发产品线之间切换。   

----------

使用liquibase，首先看看简单的例子。

	<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
	    <changeSet author="xxxx (generated)" id="1399880011956-1">
	        <createTable remarks="fee_name" tableName="T_BCP_FEE_TYPE_NAME">
	            <column name="ID" remarks="pk" type="NUMBER(10, 0)">
	                <constraints nullable="false"/>
	            </column>
	            <column name="FEE_TYPE" remarks="fee type" type="NUMBER(20, 0)"/>
	            <column name="FEE_NAME" remarks="fee name" type="VARCHAR2(60)"/>
	        </createTable>
	    </changeSet>
	</databaseChangeLog>

我们在这个例子里新建了一张表T\_BCP\_FEE\_TYPE\_NAME，有3个列分别是ID、FEE\_TYPE、FEE\_NAME。它的基本元素是changeSet，代表着数据库的一次修改。liquibase通常会将每个changeSet放在一个transaction中执行，所以比较好的做法是每个changeSet尽量只含有一个数据库结构的改动。  
liquibase还支持rollback，只需要在changeSet中加入rollback标签（某些change，liquibase可以自动rollback）。前面的例子可以改成
	
	<changeSet author="hao.xie (generated)" id="1399880011956-1">
        <createTable remarks="fee_name" tableName="T_BCP_FEE_TYPE_NAME">
            <column name="ID" remarks="pk" type="NUMBER(10, 0)">
                <constraints nullable="false"/>
            </column>
            <column name="FEE_TYPE" remarks="fee_type" type="NUMBER(20, 0)"/>
            <column name="FEE_NAME" remarks="fee name" type="VARCHAR2(60)"/>
        </createTable>
		<rollback>
		drop table T_BCP_FEE_TYPE_NAME
		</rollback>
		<!--<rollback><dropTable tableName="T_BCP_FEE_TYPE_NAME"/></rollback> -->
		<!--<rollback changeSetId="xx" changeSetAuthor="xx"/> -->
    </changeSet>  
这里面用了3种方式来写，都是可行的。加入了rollback的机制，从某种意义上讲，使得DB的迁移也有了版本的概念（不一定正确，但应该便于理解）。我们可以选择rollback到某一点，也可以再重新update到最新的数据库版本。以下是liquibase支持的几种“Roll Back To” Modes.

<table border="1">
	<tbody>
		<tr><th>Modes</th><th>Description</th></tr>
		<tr><td>Tag</td><td>Specifying a tag to rollback to will roll back all change-sets that were executed against the target database after the given tag was applied</td></tr>
		<tr><td>Number of Change Sets</td><td>You can specify the number of change-sets to rollback</td></tr>
		<tr><td>Date</td><td>You can specify the date to roll back to</td></tr>		
	</tbody>
</table>


下面要提的这个，我觉得也有版本的意味——Diff。在写代码的时候，我们经常会比较不同版本之间的代码。考虑到数据库的开发，liquibase也支持了这种compare。如果使用command，就是这样
	
	liquibase.sh --driver=oracle.jdbc.OracleDriver \
        --url=jdbc:oracle:thin:@testdb:1521:test \
        --username=bob \
        --password=bob \
    diff \
        --referenceUrl=jdbc:oracle:thin:@localhost/XE \
        --referenceUsername=bob \
        --referencePassword=bob
liquibase通过diff的产出分为报表和changeLog这2种形式，前者便于阅读，后者便于liquibase直接执行（使得2个数据库merge到同一个版本）。我更倾向于后者。在这里它使用了默认的diffTypes来比较2个数据库，现在liquibase支持的比较类型是：

- tables [DEFAULT]
- columns [DEFAULT]
- views [DEFAULT]
- primaryKeys [DEFAULT]
- indexes [DEFAULT]
- foreignKeys [DEFAULT]
- sequences [DEFAULT]
- data    

*tip：我发现这个操作也是很消耗内存的，建议在执行前，手动设置jvm heap的大小。*   
总而言之，liquibase使数据库的开发变得像代码的版本管理。它最大的好处是方便了数据库的迁移和管理，并且还支持了command、ant、maven等多种格式，非常适合在java世界使用。正如我开头所说，它就是java世界的migration（rails）工具。