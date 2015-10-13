layout: post
title: Sql树状结构
tags: sql
date: 2013-12-22

---
最近看到了一本好书《SQL Antipatterns》，作者（Bill Karwin）非常厉害，他在关系型数据库上的开发经验非常丰富，总结出了sql设计时经常陷入的误区（反模式）。这本书的在线版本：<a href="http://edu.ercess.co.in/ebooks/SQL/SQL-Antipatterns-Avoiding-the-Pitfalls-of-Database-Programming.pdf">SQL Antipatterns</a>，非常值得一读！！！
<!--more-->
作为一个专业码农，当我进大学的第一学期，就被它给坑了（好烦的数据结构），但好歹这么多年的积（che）累（dan），勉强是能应付了。但是在数据库设计方面，我经验更（hao）加（xiang）少（mei）了（you），就直接拿书中的案例来研究吧。比如，常见的employee表：

```sql
	create table employee (
		id int auto_increment primary key,
		naemployeeme varchar(50),
		manager_id int,
		foreign key (manager_id) references employee(id)
	);
```

表中的每条记录，都保存这一个manager_id，用来关联自己的上司。如果将这种组织关系画出来，就是一个像金字塔般的结构——树。而在设计employe表时，我采用了外键约束，用来串联这种内在关系。这种设计这被称为“邻接表”。

此时，查询某个经理的直接下属、某个下属的直接领导、增加一个职员都是比较轻松的工作，这也是树的特性（所以经常会用在搜索中）；但是这里存在一种比较困难的查询：某个经理的所有下属（包括每层的员工）？其中第一层的下属很常见，  

```sql
	select e2.id, e2.name
	from employee e1 left join employee e2 
	on e2.manager_id = e1.id and e1.id = '张三'; --一层，直接领导
```

但是，如果想保证搜索结果的完整，可能就要用到union，可能还需要上层应用程序的支持，即便如此，还有出错的风险。这种代价是很高的。  

这里存在的反模式类型叫做依赖父节点。虽然很多人都喜欢使用邻接表，但它的使用范围是有限的，需要谨慎设计。（**说到底，很多设计都应该做到因地制宜，需要权衡各方面的优劣势，万能钥匙很少见的。**）  

为了解决实际问题，作者提到了一些替代方案。这些方案都适当增加了设计的复杂度，但却降低了查询的困难。    
* 1.递归查询，部分数据库平台从SQL的角度加以支持。其实就是在SQL语言层，解决了递归查询的问题。  
* 2.路径枚举，该方案的最大特点是，它不再存储一个父节点的id（外键），而是将root到其自身的路径以某种格式（通常就是可解析的字符串），存储在path中，path是varchar类型。比如1/2/3/4../n，n是父节点的id。缺点？首先是字符串的解析，需要支持。其次是字符串的长度限制了树的结构不是可无限增长的。其实最重要的一点是，缺少了**外键约束**，意味着path中可能存在无效的节点，而我们却不知道。  
* 3.嵌套集，这是一个聪明的设计。依然，我们不存储父节点的id。这回使用2个属性，left\_id 小于｛id|id∈后代节点｝小于 right\_id。呵呵，在这数值区间中的节点即当前节点的子节点。不过在查询直接父节点，稍显复杂，需要保证2个节点之间不存在其他节点。缺陷？部分查询复杂、增删节点复杂。  
* 4.闭包集，一个通用设计。不过它需要2张表了，因为它不再纠结于用一张表同时解决信息存储和结构的存储。将所有父子关系移到新表中。

```sql
	create table employeeRelation (
	manager_id int ,
	employee_id int ,
	primary key (manager_id, employee_id),
	foreign key (manager_id) references employee (id),
	foreign key (employee_id) references employee (id)
	);
```

外键的应用保证了数据库的引用完整性，查询也优化了，也便于理解了。还有一个优点是在调整组织结构关系时，不会影响到employee表中的基本属性，灵活性大大提升。不过这种设计的前提是需要额外的表。说起来，我更喜欢这种直接的方式。  

当然这些设计模型在某些特定场合都有可能成为合理的设计，或者说可行的设计，需要使用者自己去甄别。如果需要更多地了解数据库中层级结构的设计，可以Google更多的文献。
