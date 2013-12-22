---
layout: post
title: 数据库反模式：树
category: sql antipattern
excerpt: 在阅读SQL Antipatterns时，几乎是重新认识了数据库设计。原作者（Bill Karwin）非常厉害，他的数据库经验对于普通的开发人员来说，是很宝贵的。尤其是像我，在设计数据库时，往往会陷入一种纠结中......
tags: [sql]
---
{% include JB/setup %}

在学习的过程中，我发现不少章节提出的问题，之前或者没有遇到过，或者没有细想过，这回一起受教了。在此，我挑选了一些，并缩小一下篇幅，方便回忆。<a href="http://edu.ercess.co.in/ebooks/SQL/SQL-Antipatterns-Avoiding-the-Pitfalls-of-Database-Programming.pdf">SQL Antipatterns在线版本</a>
### 一 树（设计） ###
作为一个开发者，是非常熟悉“树”这种结构的，而文章中也提到了几种可能的使用场景：1.组织架构；2.话题型讨论。  
有一种简单的设计是使用外键约束，这被称为“邻接表”。比如说，常见的employee表，公司的每个员工都有自己的领导（大Boss例外）。

	create table employee (
		id int auto_increment primary key,
		naemployeeme varchar(50),
		manager_id int,
		foreign key (manager_id) references employee(id)
	);
此时，查询某个经理的直接下属、某个下属的直接领导、增加一个职员都是比较轻松的工作，这也是树的特性（所以经常会用在搜索中）；但是这里比较困难的查询是：张三管理的所有员工？  

	select e2.id, e2.name
	from employee e1 left join employee e2 
	on e2.manager_id = e1.id and e1.id = '张三'; --一层，直接领导
显然，如果想保证搜索结果的完整，可能就要用到union，可能还需要上层应用程序的支持，即便如此，还有出错的风险。这种代价是很高的。  
这里存在的反模式类型叫做依赖父节点。虽然很多人都喜欢使用邻接表，但它真实的使用范围是有限的，需要谨慎设计。（说到底，很多设计都应该做到因地制宜，需要权衡各方面的优劣势，万能钥匙很少见的。）  
为了解决实际问题，作者提到了一些替代方案。这些方案都适当增加了设计的复杂度，但却降低了前面所举例的困难。    
<img src = "{{ ASSET_PATH }}/images/sql_naive_tree_1.jpg"/> 
1.递归查询，部分数据库平台从SQL的角度加以支持。  
2.路径枚举，该方案的最大特点是，它不在存储一个父节点的id（外键），而是将root到其自身的路径山的节点以某种格式，存储在path中，path是varchar类型。比如1/2/3/4../n，n是父节点的id。缺点？字符串的解析、字符串的长度限制了树的结构不是无限增长、path中可能存在无效的节点。  
3.嵌套集，这是一个聪明的设计。依然，我们不存储父节点的id。这回使用2个属性，left\_id 小于｛id|id∈后代节点｝小于 right\_id。呵呵，在这数值区间中的节点即当前节点的子节点。不过在查询直接父节点，稍显复杂，需要保证2个节点之间不存在其他节点。缺陷？部分查询复杂、增删节点复杂。  
4.闭包集，一个通用设计。不过它需要2张表了，因为它不再纠结于用一张表同时解决信息存储和结构的存储。将所有父子关系移到新表中。

	create table employeeRelation (
	manager_id int ,
	employee_id int ,
	primary key (manager_id, employee_id),
	foreign key (manager_id) references employee (id),
	foreign key (employee_id) references employee (id)
	);
外键的应用保证了数据库的引用完整性，查询也优化了，也便于理解了。还有一个优点是在调整组织结构关系时，不会影响到employee表中的基本属性，灵活性大大提升。不过这种设计的前提是需要额外的表。说起来，我更喜欢这种直接的方式。  
当然这些设计模型在某些特定场合都有可能成为合理的设计，或者说可行的设计，需要使用者自己去甄别。如果需要更多地了解数据库中层级结构的设计，可以Google更多的文献。如果想在数据库设计中，能顺利地解决类似问题，还需要多多实践。