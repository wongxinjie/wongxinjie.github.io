title: Redis笔记
date: 2015-07-27 23:45:39
categories: Redis
tags:
- 笔记
---

### redis基础数据类型及其应用场合
##### String 字符串
string 是redis中最基本的数据类型，能存储任何形式/编码的字符串（当然包括二进制），单个value的最大容量是512M。
简单的key-value类型，相当于Memcached，但其value可以是String，也可以是数字。但存储的形式是字符串。
string可以取代memcached作为缓存，存用户的验证码/session数据等，自带的expire功能。
#### hash 字典
hash是redis中一种非常有用的数据类型，相当于python中的dict，能存储key-value，但是无论是但是key/value只支持string而不支持其他的数据类型（即你不能嵌套一个list什么类型的）。
redis提供的Hash是一种key-dict形式的存储，可以想关系型数据库一样update某个字段。
hash的应用场合很多，譬如作为集群任务处理的注册中心，简单的登记哪部机器在处理什么任务，避免重复处理。在网站后台中也可以实现一些类似于帖子的热度信息等。
### list 列表
list 是redis中的一个有序的字符串列表，列表内的元素是非唯一的，相当于双端队列。
redis的list其实是一个双端链表，利用List可以实现最新消息排行等功能。List另外一个应用就是消息队列（生产者消费者模型）。
list 可以作为单机的任务队列，在不同的进程中通信等。通过list可以实现简单的分布式任务处理机制。也可以实现分页、新鲜事等。
#### set 集合
set是和list类似的一种数据类型，但是set是自动排重的，当要存储一个没有重复元素的列表数据，可以使用set。而且set还提供了判断某个成员是否存在列表的一个接口。
set还提供了求交集、并集、差集的操作，这能很方便的实现类似微博等应用中的好友共同关注、共同喜好、二度好友等功能。
#### sorted set 有序集合
sorted set，是set中每个value都关联一个权值，不同的value可以有相同的权值，sorted set会根据权值进行排序。
sorted set可以实现一些排序功能，比如按点击量/时间排序/时间轴等功能。作为任务队列可是实现优先队列，优先处理一些权值高的任务等。
<!--more-->

### 持久化
通常 Redis 将数据存储在内存中或虚拟内存中,它是通过以下两种方式实现对数据的持久化。
#### 快照方式:(默认持久化方式)
客户端也可以使用 save 或者 bgsave 命令通知 redis 做一次快照持久化。save 操作是在主线程中保存快照的,由于 redis 是用一个主线程来处理所有客户端的请求,这种方式会阻塞所有客户端请求。所以不推荐使用。另一点需要注意的是,每次快照持久化都是将内存数据完整写入到磁盘一次,并不是增量的只同步增量数据。如果数据量大的话,写操作会比较多,必然会引起大量的磁盘 IO 操作,可能会严重影响性能。
#### 日志追加方式
这种方式redis会将每一个收到的写命令都通过write函数追加到文件中。当 redis 重启时会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。当然由于操作系统会在内核中缓存 write 做的修改,所以可能不是立即写到磁盘上。这样的持久化还是有可能会丢失部分修改。不过我们可以通过配置文件告 诉redis 我们想要通过 fsync 函数强制操作系统写入到磁盘的时机。有三种方式如下(默认是:每秒 fsync 一次)
```
appendonly yes  //启用日志追加持久方式
appendfsync always  //每次收到写命令立即强制写入磁盘，最慢，但完全保证持久化，不推荐
appendfsync everysec // 每秒强制写入磁盘一次，在性能和持久化方面坐了很好的折衷
appendfsync no //依赖系统设置，性能最好，持久化没保证
```

### 主从同步
Redis 支持将数据同步到多台从库上,这种特性对提高读取性能非常有益。
* master 可以有多个 slave。
* 除了多个 slave 连到相同的 master 外,slave 也可以连接其它 slave 形成图状结构。
* 主从复制不会阻塞 master。也就是说当一个或多个 slave 与 master 进行初次同步数据时,master 可以继续处理客户端发来的请求。相反 slave 在初次同步数据时则会阻塞不能处理客户端的请求。
* 主从复制可以用来提高系统的可伸缩性 ,我们可以用多个slave专门用于客户端的读请求,比如sort操作可以使用slave来处理。也可以用来做简单的数据冗余。
* 可以在 master 禁用数据持久化,只需要注释掉 master 配置文件中的所有 save 配置,然后只在 slave 上配置数据持久化。

##### 过程
当设置好 slave 服务器后,slave 会建立和 master 的连接,然后发送 sync 命令。无论是第一次同步建立的连接还是连接断开后的重新连接,master 都会启动一个后台进程,将数据库快照保存到文件中,同时 master 主进程会开始收集新的写命令并缓存起来。后台进程完成写文件后,master 就发送文件给 slave,slave 将文件保存到磁盘上,然后加载到内存恢复数据库快照到 slave 上。接着 master 就会把缓存的命令转发给 slave。而且后续 master 收到的写命令都会通过开始建立的连接发送给 slave。从 master 到 slave 的同步数据的命令和从 客户端发送的命令使用相同的协议格式。当 master 和 slave 的连接断开时 slave 可以自动重新
建立连接。如果 master 同时收到多个 slave 发来的同步连接命令,只会启动一个进程来写数据库镜像,然后发送给所有 slave。
配置 slave 服务器很简单,只需要在配置文件中加入如下配置
```
slaveof 192.168.1.1 6379 #指定master的ip和port
```



### python的redis模块
redis的Python模块redis-py。安装此模块之后，就可以利用Python与Redis交互了。
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
#获取匹配键值
keys = [ key  for key in srdb.scan_ier(match='PREFIX*']
{% endcode %}
更多基本命令参照redis命令参考
