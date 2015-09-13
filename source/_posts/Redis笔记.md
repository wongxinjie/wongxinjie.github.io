title: Redis笔记
date: 2015-07-27 23:45:39
categories: NoSQL
tags:
- redis
---
redis的Python模块redis-py。安装此模块之后，就可以利用Python与Redis交互了。
redis中一共有5中数据结构，分别是String、Hash、List、Set、Sorted Set。这5中数据结构各有各自己适用的场景。
- String（字符串）：简单的key-value类型，相当于Memcached，但其value可以是String，也可以是数字。但存储的形式是字符串。
- Hash（字典）：redis提供的Hash是一种key-dict形式的存储，可以想关系型数据库一样update某个字段。
- List（列表）：redis的list其实是一个双端链表，利用List可以实现最新消息排行等功能。List另外一个应用就是消息队列（生产者消费者模型）。
- Set（集合）：redis的集合和python的集合一样，去重、求合并差集相当方便。
- SortedSet （有序集合）：在Set中增加一个权重参数score，可实现带权重的队列。
<!--more-->

redis-py基本实现了redis的操作命令。
{% code %}
import redis
rdb = redis.Redis(host='127.0.0.1', port=6379, db=0) #db可选
srdb = redis.StrictRedis(host='127.0.0.1', port=6739)
{% endcode %}

以上代码便是利用redis-py链接redis，利用rdb或srdb即可进行数据的存储。Redis和StrictRedis的区别在于，Redis是StrictRedis的子类，为了向后兼容。
推荐使用的StrictRedis。
各种数据结构的操作
{% code %}
#String
>>> srdb.set('string:key', 'python')
>>> value = srdb.get('string:key')
>>> value
python
#带过期时间戳，单位s
>>> srdb.set('string:key:with:expire', 'redis-python', 10)
# 10s later
>>> srdb.get('string:key:with:expire')
# get None

#Hash
>>> srdb.hset('hash:key', 'dict_key', 'dict_value')
1L
>>> srdb.hget('hash:key', 'dict_key')
dict_value
>>> srdb.hgetall('hash:key')
{'dict_key': 'dict_value'}
#List
>>> rdb.rpush('list', 'python-redis')
1L
>>> rdb.lpop('list')
python-redis
{% endcode %}
以及
{% code %}
#获取所有键值
srdb.keys()
#获取匹配建筑
keys = [ key  for key in srdb.scan_ier(match='PREFIX*']
{% endcode %}
更多基本命令参照redis命令参考
