layout: post
title: 数据库设计中表的继承
date: 2014-2-22
tags: [sql]

---
### 单表继承
将所有相关的类型都存在一张表中，为所有类型的所有属性都保留一列。所以，当系统中新增添了一个子类型时，我们都需要将它的属性逐一添加进数据库表中，也就是改变表的结构，添加新的列。同时要保证2点：必须有一列能够区分出每一行的子类型；列允许空值。

```sql	
	CREATE TABLE Issues (
		issue_id SERIAL PRIMARY KEY,
		reported_by BIGINT UNSIGNED NOT NULL,
		product_id BIGINT UNSIGNED,
		priority VARCHAR(20),
		version_resolved VARCHAR(20),
		status VARCHAR(20),
		issue_type VARCHAR(10), -- BUG or FEATURE
		severity VARCHAR(20), -- only for bugs
		version_affected VARCHAR(20), -- only for bugs
		sponsor VARCHAR(50), -- only for feature requests
		FOREIGN KEY (reported_by) REFERENCES Accounts(account_id)
		FOREIGN KEY (product_id) REFERENCES Products(product_id)
	);
```

### 实体表继承 
为每个子类型建一张独立的表。每个表都包含属于父类的属性，同时也有子表所特有的属性。这种设计也比较简单明了，无非就是一种类型一张表。所以我们缺少元数据，搞清楚子表之间的关系（我们会发现数据库中某些表有重复的属性，却不知其缘由）。
### 类表继承 
首先创建一张父表，包含了公共属性。而其他的子表通过外键去和父表关联。

```sql
	CREATE TABLE Issues (
		issue_id SERIAL PRIMARY KEY,
		reported_by BIGINT UNSIGNED NOT NULL,
		product_id BIGINT UNSIGNED,
		priority VARCHAR(20),
		version_resolved VARCHAR(20),
		status VARCHAR(20),
		FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
		FOREIGN KEY (product_id) REFERENCES Products(product_id)
	);
	CREATE TABLE Bugs (
		issue_id BIGINT UNSIGNED PRIMARY KEY,
		severity VARCHAR(20),
		version_affected VARCHAR(20),
		FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
	);
	CREATE TABLE FeatureRequests (
		issue_id BIGINT UNSIGNED PRIMARY KEY,
		sponsor VARCHAR(50),
		FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
	);
```

在这种设计中，数据库的表之间的父子关系由元数据来确保，是一个不错的设计，唯一没有解决的是扩展问题。当需要经常增加新属性时，还是不得不改变表结构，增加新的列。