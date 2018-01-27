title: MySQL笔记
date: 2017-02-24 22:00:32
categories: 数据库
tags:
    - MySQL
    - 读书笔记
---
在开发过程中，虽然大多数时候都用ORM来实现数据库操作，但是有时候需要裸写SQL时候发现有些概念不是很清晰，所以重新学习一下MySQL并做记录。
#### 检索数据
##### 简单检索
获取检索的前N行：
```SQL
SELECT col FROM which_table LIMIT N;
```

获取从指定N行开始后的M行：

```SQL
SELECT col FROM which_table LIMIT N, M;
SELECT col FROM which_table LIMIT M OFFSET N;
```

##### 检索排序

数据库不对检索出来的结果做假定有序的，要排序需要明确排序顺序，如

```SQL
SELECT prod_id, prod_price, prod_name
FROM products
ORDER BY prod_price, prod_name;
```

数据排序默认是升序排序（从A到Z），要降序排序需明确指明：

```SQL
SELECT prod_price
FROM products
ORDER BY prod_price DESC;
```
<!--more-->

##### 过滤数据

实际生产环境中检索通常需要过滤，即有SELECT通常会有WHERE，WHERE 子句的操作符和一般编程语言一致。

检查某个值（通常是数字类型）使用BETWEEN检查范围：

```SQL
SELECT proc_name, prod_price
FROM products
WHERE prod_price BETWEEN 5 AND 10;
```

对于某列存在空值NULL的，使用空值检查而不能使用 = NULL或 != NULL等语法：

```SQL
SELECT prod_id
FROM products
WHERE prod_desc IS NULL;
```

还可以用AND和OR来连接多个过滤条件，需要注意的是AND 的优先级高于OR：

```SQL
SELECT prod_name, prod_price
FROM products
WHERE (vend_id=1002 OR vend_id=1003) AND prod_price > 20;
```

对于同一个列多个过滤条件，使用OR可以使用更灵活的IN来代替：

```SQL
SELECT prod_name, prod_price
FROM products
WHERE vend_id IN (1002, 1003) AND prod_price > 20;
```

##### 通配符与正则表达式

在检索一些文本字段时，会遇到一些需要检索包含的，这是就需要用到LIKE谓词了。与LIKE搭配使用的通配符有(%)和(\_)，%表示任何字符出现任何次数，\_只匹配单个任意字符。

```SQL
SELECT prod_name, prod_price
FROM products
WHERE prod_name LIKE "Python%";

SELECT Prod_name, Prod_price
FROM products
WHERE prod_name LIKE "%MySQL%";
```

使用%可以检索一某个关键字开始/结束或者包含的字段，如上面的例子。

有些类似查找电话号码之类的工作通配符是无法实现的，这时就需要使用MySQL的正则表达式了。如要匹配四位数字相连的字段，通配符就没办法实现了：

```SQL
SELECT prod_name
FROM products
WHERE REGEXP '[[:digit:]]{4}'
ORDER BY prod_name;
```

MySQL的正则表达式还是值得看一看的，但日常工作中较少碰到，所以先不深究。

##### 创建计算字段

有时候表中没有我们需要的字段，但可以通过表中的字段进行运算得出，这时候就要用计算字段了。

常用的拼接字段，如将供应商的名字和国家去掉空格拼接打印出来：

```SQL
SELECT
Concat(RTrim(vend_name), '(', RTrim(vend_conutry), ')') AS vend_title
FROM vendor
ORDER BY vend_name;
```

或则根据字段执行计算：

```SQL
SELECT
    prod_id,
    quantity,
    item_price,
    quantity*item_price AS expanded_price
FROM orderitems;
```

#### 数据聚合与分组

常用的聚合函数:

| 函数      | 说明       |
| ------- | -------- |
| COUNT() | 返回某列的行数  |
| AVG()   | 返回某列的平均值 |
| SUM()   | 返回某列值之和  |
| MAX()   | 返回某列的最大值 |
| MIN()   | 返回某列的最小值 |

使用中有一些注意事项：

* 使用COUNT(*)对表行数目进行统计，不管表列中是否包含NULL值还是非空值
* 使用COUNT(column)会忽略NULL值的行


* AVG() 、SUM()、MAX()和MIN() 函数忽略列为NULL值的行

使用GROUP BY 可以对数据进行分组：

```SQL
SELECT vend_id, COUNT(*)
FROM products
GROUP BY vend_id;
```

同时可以使用HAVING对分组数据进行过滤:

```SQL
SELECT vend_id, COUNT(*) AS product_count
FROM products
GROUP BY vend_id HAVING product_count > 5;
```

通常GROUB BY会对输出的数据要求排序，这时候用ORDER BY：

```SQL
SELECT order_num, SUM(quantity * item_price) as ordertotal
FROM orderitems
WHERE order_status != 'CANCEL'
GROUP BY order_num HAVING ordertotal >= 50
ORDER BY ordertotal;
```

#### 子查询

子查询是嵌套在其他查询中的查询，SQL语句对嵌套的子查询深度没有限制，但一般为了维护和性能，都不会嵌套太深，一个例子:

```SQL
SELECT cust_name, cust_contact
FROM customers
WHERE cust_id IN (SELECT cust_id
                  FROM orders
                  WHERE order_num IN (SELECT order_num
                                      FROM orderitems
                                      WHERE prod_id='INT2'));
```

实际上子查询可以作为计算字段：

```SQL
SELECT
    cust_name,
    cust_state,
    (SELECT COUNT(*)
     FROM orders
     WHERE orders.cust_id = customers.cust_id) AS orders
FROM customers
ORDER BY cust_name;
```

#### 表联接

SQL最大的功能之一就是在检索查询时执行联接(join)。在数据库设计时通常要根据数据库范式将数据拆分到多个表存储，单检索这些数据时就需要使用表联接。

##### 等值联接

等值联接又叫内联接，是最常用的联接如:

```SQL
SELECT vend_name, prod_name, prod_price
FROM vendors, products
WHERE vendord.vend_id = products.vend_id
ORDER BY vend_name, prod_name;
```

在联接两个表时，实际上MySQL数据库只是将第一个表的每一个行和第二个表的每一行进行搭配（笛卡儿积），WHERE子句作为过滤条件，只输出匹配给定条件的行。实际上有另外一种等同的写法：

```SQL
SELECT vend_name, prod_name, prod_price
FROM venders INNER JOIN products
ON vendors.vend_id = products.vend_id;
```

SQL对一条SQL语句中可以联接的表数量没有限制，但从性能来讲，联接的表不要超过三个。

有时候对一些复杂的SELECT查询，可以使用联接替代子查询，如子查询一节的一个列子：

```SQL
SELECT cust_name, cust_contact
FROM customers, orders, orderitems
WHERE customers.cust_id = orders.cust_id AND\
      orderitems.order_num = orders.order_num AND\
      prod_id = 'INT2';
```

##### 自联接

自联接是指在查询时联接的是同一表，通常自联接可以用子查询来替换，但是子查询的性能有时候比子查询好，一个例子:

子查询解决方案:

```SQL
SELECT prod_id, prod_name FROM products
WHERE vend_id=(SELECT vend_id
               FROM products
               WHERE prod_id='DINTER');
```

内联接解决方案：

```SQL
SELECT p1.prod_id, p1.prod_name
FROM products AS p1, products AS p2
WHERE p1.vend_id = p2.vned_id AND p2.prod_id = 'DINTER';
```

##### 外部联接

有时候需要联接包含那些相关表没有相关联的行，我们需要外部联接。一些需要用到外部联接的场景:

* 对每个客户下了多少订单进行计数，包括那些从未下订单的客户
* 列出所用产品的订购数量，包括没有人订购的产品
* 计算平均销售规模，包括那些至今尚未下订单的用户

为了检索所有客户，包括那些没有下过订单的客户：

```SQL
SELECT customers.cust_id, orders.order_num
FROM customers LEFT OUTER JOIN orders
ON customers.cust_id = orders.cust_id;
```

外部联接又分左右联接，实际上左右联接唯一的区别便是所关联的表的顺序不一样，LEFT表示结果包含左表所有的行，RIGHT 表示包含右表所有的行。上面的例子同样可以使用又联接来实现：

```SQL
SELECT customers.cust_id, orders.order_num
FROM orders RIGHT OUTER JOIN customs
ON customs.cust_id = orders.cust_id;
```

#### 组合查询

组合查询是将多个查询结构合并作为单个查询结果集返回。一般采用组合查询有两种情况:

* 在单个查询中从不同的表返回类似的结构的数据
* 对单个表执行多个查询，按单个查询返回结果

使用UNION是组合SQL查询，一个例子是在分表中要从过个表获取数据然后返回:

```SQL
SELECT user_id, traffic, bandwidth
FROM logs1
WHERE timestamps='2017-01-02 00:00:00'
UNION
SELECT user_id, traffic, bandwidth
FROM logs2
WHERE timestamps='2017-01-02 00:00:00';
```

使用UNION要注意几点：

* UNION必须由两条或以上的SELECT 语句组成
* UNION中的查询必须包含相同的列、表达式或者聚合函数（不要求顺序）
* 列数据类型必须不要求一致，但必须兼容

在使用UNION时，重复行会被自动取消，加入要返回全部的行包括重复行，用UNION ALL。

#### 表的创建的操纵

##### 主键

主键可是是单个列，也可以是组合列，但无论是单个列还是组合列，都必须是唯一的，而且是不能为NULL值的。实际使用MySQL的AUTO_INCREMENT列做主键是比较常见的情况。

##### 引擎类型

在创建表时会指定引擎（ENGINE=InnoDB语句）。MySQL和其他DBMS一样有一个具体管理和处理数据的内部引擎。在CREATE TABLE语句时，引擎复制具体建表，而使用SELECT语句时，引擎负责处理请求。数据库的默认引擎可以在配置文件中设置，较新版本的MySQL的默认引擎是InnoDB。下表是MySQL 5.7支持的引擎常见的引擎（[文档](https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html))：

| 引擎     | 特性                            |
| ------ | ----------------------------- |
| InnoDB | 支持事务，事务安全，支持行锁，支持全文搜索（5.6.4+） |
| MyISAM | 只支持表级所，多用于读或只读表               |
| Memory | 数据存储在内存中，处理速度快，适合临时表          |

##### 修改表
用ALTER TABLE 来修改表，如定义外键：
```SQL
ALTER TABLE orderitems
ADD CONSTRAINT fk_orderitems_orders FOREIGN KEY (order_num)
REFERENCES orders (order_num);
```

#### 其他话题
* [全文搜索](https://dev.mysql.com/doc/refman/5.7/en/fulltext-search.html)
* [视图](https://dev.mysql.com/doc/refman/5.7/en/view-syntax.html)
* 存储过程
* 触发器
* **管理事务**



**参考资料**

* 《MySQL必知必会》
* [MySQL 5.7文档](https://dev.mysql.com/doc/refman/5.7/en/)
