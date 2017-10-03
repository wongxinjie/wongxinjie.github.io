title: SQL基础教材笔记
date: 2015-05-22 21:40:33
categories: MySQL
tags:
- 笔记
---

### 聚合与排序

#### 1、聚合查询
SQL中有5个常用的聚合函数：COUNT、SUM、AVG、MAX、MIN。聚合，就是将多行数据汇总成一行，所以聚合函数输入的是多行，而输出的只是一行。
使用聚合函数时，NULL的列是不参与计算的。比如在goods表中7列中有3列的price为NULL，则
{% code %}
SELECT COUNT(*), COUNT(price) FROM goods;
{% endcode %}
的结果为7|4

#### 2、对表进行分组
要对表的数据进行分组处理，用GROUP BY子句。在GROUND BY子句中指定的列被称为聚合键或者分组列，是决定表切分方式的非常重要的列。
**书写顺序：SELECT….FROM….WHERE….GROUP BY…..**
**执行顺序：FROM—->WHERE—->GROUP BY——>SELECT**
使用聚合函数和GROUP BY子句时要避免犯以下错误：
（1）把聚合键之外的列面书写在SELECT子句中
（2）在GROUP BY子句中不能使用SELECT子句中定义的别名
（3）GROUP BY显示的结果是无序的
（4）在WHERE子句中不能使用聚合函数
<!--more-->

#### 3、为聚合结果指定条件
使用GROUP BY子句对表进行分组后，可以使用HAVING来选取特定的分组。
**书写顺序：SELECT…FROM…WHERE…GROUP BY….HAVING**
比如，要实现通过对商品种类进行分组后，选取销售单价的平均值大于等于2500元的种类：
{% code %}
SELECT type, AVG(price) FROM goods GROUP BY type HAVING AVG(price) >= 2500;
{% endcode %}
HAVING子句中能够使用的只能是常数、聚合函数以及聚合键。

#### 4、对查询结果进行排序
对查询结果结果进行排序，用ORDER BY。
**书写顺序：SELECT…FROM…WHERE…GROUP BY…HAVING…ORDER BY…**
**执行数序：FROM—->WHERE—->GROUP BY—->HAVING—->SELECT—->ORDER BY**
ORDER BY默认是升序排序的，所以假如要降序必须要明确指明，ORDER BY <列名> DESC。


### 数据更新

#### 1、数据插入
语法：INSERT INTO <表名>(列1，列2，列3，……) VALUES(值1，值2，值3，……);
多行插入， INSERT INTO <表名> VALUES (值A1, 值A2，值A3， ……), (值B1，值B2，值B3，……), ……
从其他表中复制数据：首先建立一个表，然后：
{% code %}
INSERT INTO copy_table (colums) SELECT (columns) FROM source_tables;
{% endcode %}

#### 2、数据删除
删除表及数据：DROP TABLE;
删除表数据：DELETE FROM;
注：在MySql等数据库中还有TRUNCATE语句，可以高效的删除表中所有的数据。

#### 3、数据更新
语法：UPDATESET column = value WHERE;

#### 4、事务
事务是需要在同一个处理单元中经行的一系列更新处理的集合。
语法:
{% code %}
START/BEGIN TRANSACTION
DML actionA;
DML actionB;
……
COMMIT/ROLLBACK;
{% endcode %}
事务开始语句在SQL Server、PostgreSQL中为BEGIN TRANSACTION，在MySQL中为START TRANSACTION。
COMMIT提交处理，一旦提交无法修改。ROLLBACK取消处理。
**事务四种标准约定，ACID特性：原子性、一致性、隔离性以及持久性。**


### 复杂查询

#### 1、视图
从SQL的角度上看，视图就是一张表。与表不同的是，视图不会将数据保存到存储设备之中。实际上视图保存的是SELECT语句，从视图中读取数据时，视图会在内部先执行SELECT语句并创建一张临时表。
视图的好处：视图无需保存数据，可以节省存储设备的容量；将频繁使用的SELECT语句保存成视图，可以减少重复工作。
语法：
{% code %}
CREATE VIEW(columnA, columnB, ……) AS;
{% endcode %}
比如，要建立一个商品种类数量合计的视图：
{% code %}
CREATE VIEW GoodsSum (type, type_sum) AS SELECT type, SUM(\*) FROM goods GROUP BY type;
{% endcode %}
使用:
{% code %}
SELECT type, type_sum FROM GoodsSum;
{% endcode %}
使用视图查询的实际执行过程为：先执行视图的SELECT语句，再执行FROM子句中的SELECT语句。因此，多重视图会降低SQL的性能。
视图的使用是有限制的，其一视图不能使用ORDER BY子句。其二对视图进行更新存在一些限制，不满足SELECT子句中未使用DISTINCT、FROM子句中只使用一张表、未使用GROUP BY以及未使用HAVING来定义SELECT语句的视图才能进行更新。
删除视图：DROP VIEW;

#### 2、子查询
子查询实际上就是一张一次性的视图，子查询就是将定义视图的SELECT语句直接用于FROM语句中。在实际使用过程中，子查询要有合适的名称。
标量子查询，查询的结果只能返回一行一列的结果。比如，“查询出售单价高于平均销售单价的商品”：
{% code %}
SELECT goods_id, name, price FROM goods WHERE prices >= (SELECT AVG(price) FROM goods);
{% endcode %}
标量子查询基本可以用于语句的所有地方。

#### 3、关联子查询
对于表中某一部分记录的集合进行比较时，就可以使用关联子查询。
比如要实现选取各商品分类中高于该分类平均销售单价的商品。
{% code %}
SELECT goods_id, name, price FROM goods AS S1 WHERE price > (SELECT AVG(price) FROM goods AS S2 WHERE S1.type = S2.type GROUP BY type);
{% endcode %}


### 谓词

#### 1、LIKE谓词
LIKE谓词主要对字符串经行部分一致的查询。
前方一致查询SELECT title FROM blog WHERE title LIKE ‘sql%’; 中间一致查询 SELECT title FROM blog WHERE title LIKE ‘%sql%’;后方一致查询SELECT title FROM blog WHERE ‘%sql’;
其中%代表0个字符以上的任意字符串。而_代表的是任意一个字符。

##### 2、BETWEET谓词
使用BETWEET可以进行范围查询。
语法：
{% code %}
SELECT * FROM table WHERE column BETWEET valueA AND valueB; 效果与SELECT * FROM table WHERR column >valueA AND column < valueB;
{% endcode %}

#### 3、IS NULL、IS NOT NULL进行空值、非空值判断。

#### 4、IN谓词
IN谓词就是OR的简洁版。
IN谓词的一个常用的用法便是与子查询组合编写可维护的查询语句。子查询作为IN谓词的参数。
语法：
{% code %}
SELECT * FROM tableA where tid in (SELECT tid FROM tableB where col=value);
{% endcode %}

#### 5、EXISTS谓词
EXISTS谓词的作用是判断符合条件的记录是否存在，如果记录存在则返回TRUE, 否则返回FALSE。
语法：
{% code %}
SELECT * FROM tableA AS a WHERE EXISTS (SELECT * FROM tableB AS b WHERE col=value AND a.id = b.id);
{% endcode %}


### 表的联接
表的联接就是将多张表的列合并在一起已选取合适的数据（列）。

#### 1、INNER JOIN：内联接
将表A中的列作为桥梁，将表B中满足同样条件的列汇集到一起。
例子:
{% code %}
SELECT TS.tenpo_id, TS.tenpo_mei, TS.shohib_id, S.shohin_mei, S.hanbai_tanka FROM TenpoShohin AS TS INNER JOIN Shohin AS S ON TS.shohin_id = S.shohin_id;
{% endcode %}
内联要点（1）FROM子句，在子句中要进行 tableA INNER JOIN tableB；（2）ON子句，ON子句就是联接的条件，也就是两张表所用的联结键。ON子句必须写在FROM和WHERE之间。（3）可以使用WHERR子句进行筛选。

2、OUTER JOIN：外联接
外联接的作用在于选出主表的全部信息。在实际使用中通常用于生成报表。
与内联接语法的差异在FROM子句：
{% code %}
tableA LEFT(RIGHT) OUTTER JOIN tableB
{% endcode %}
，其中假如是LEFT OUTTER JOIN则tableA为主表，否则tableB为主表。
