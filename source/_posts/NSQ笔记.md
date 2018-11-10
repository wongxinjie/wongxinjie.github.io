title: NSQ简介
date: 2018-09-07 20:00:10
categories: 消息队列
tags:
- NSQ
- Golang
---


NSQ是分布式实时消息队列。NSQ是分布式的、拓扑结构，具有无单点故障、故障容错、高可用性和保证消息的可靠传递等特点，容易配置和部署。

### 核心概念
#### message
数据流形式，生产者需要消费者处理的一个数据流（包）。
#### topic
消息的逻辑关键词。
* message属于某个特定的topic
* 在由生产者在发布消息时生成
* 通常描述消息的内容

#### channel
生产者与消费者之间的消息通道，相当于消息队列。
* 一个topic可以有多个channel
* 一个channel只能对应一个topic
* channel由消费者首次订阅产生，一个channel可以有多个消费者，起负载均衡的作用。
* 一个topic的消息会复制到与topic关联的各个channel
* channel的名称通常描述的是消费者的业务

### 工具套件
#### nsqd
一个负责接收、排队、转发消息到客户端的守护进程。负责message的收发和队列的维护。
nsqd默认监听一个TCP端口（4150）和一个HTTP端口 （4151）。  nsqd功能特性：
* topic发布到channel中的消息会被消费者消费
* 只要channel存在，即使没有消费者，生产者的message也会缓存在队列中
* topic发布的message至少会被消费一次，nsq正常退出，消息会被写入磁盘
* 限制内存中用，channel中的message是存放在内存中的，可以限制其大小，超出时可以将 message缓存到磁盘
* topic和channel一旦创建就会一直存在，除非主动删除

#### nsqlookupd
中心管理服务进程。nsqlookupd提供管理nsqd服务，客户端查询生产者、topic和channel服务。
使用TCP(默认端口4160)管理nsqd服务，使用http(默认端口4161)管理nsqadmin服务。
* 一个 nsqd服务只能注册到一个nsqlookupd
* 去中心化，nsqlookupd崩溃也不会影响到nsqd
* 可以充当nsqd和nsqdadmin之间的交互中间件
* 消费者可以通过nsqlookupd获取生产者信息

<!--more-->
#### nsqadmin
一套Web用户界面，可实时查看集群的统计数据和执行各种各样的管理任务。默认访问端口是4171。
* 查看和管理topic和channel
* 查看message数量和状态
* 当前版本还是有点buggy，不建议在生成中使用

### NSQ使用
写本地，读远程。消息生产者与nsqd在同一个实例中，生产者不负责发现其他nsqd服务，只往本地nsqd投放消息。消费者可以通过nsqlookupd发现nsqd，连接nsqd消费消息。
启动nsqlookupd
```
nsqlookupd
```
启动nsqd
```
nsqd --lookupd-tcp-address=127.0.0.1:4160
```
运行nsqadmin（可选)
```
nsqadmin --lookupd-http-address=127.0.0.1:4161
```
发布消息：

```
curl -d 'user1;url1' 'http://127.0.0.1:4151/pub?topic=clicks'
curl -d 'user2;url2' 'http://127.0.0.1:4151/pub?topic=clicks'
curl -d 'user3;url3' 'http://127.0.0.1:4151/pub?topic=clicks'
```
topic管理

```
curl -X POST 'http://127.0.0.1:4151/topic/create?topic=nsqdTopic'
curl -X POST 'http://127.0.0.1:4151/topic/delete?topic=nsqdTopic'
curl -X POST 'http://127.0.0.1:4151/topic/empty?topic=nsqdTopic'
```
channel管理(要创建channel先要有对应的Topic)

```
curl -X POST 'http://127.0.0.1:4151/channel/create?topic=nsqdTopic&channel=nsqdChannel'
curl -X POST 'http://127.0.0.1:4151/channel/delete?topic=nsqdTopic&channel=nsqdChannel'
curl -X POST 'http://127.0.0.1:4151/channel/empty?topic=nsqdTopic&channel=nsqdChannel'
```
获取nsqd统计

```
curl 'http://127.0.0.1:4151/stats'
```
查看topic有哪些生产者

```
curl 'http://127.0.0.1:4161/lookup?topic=test'
```

nsqlookupd管理接口

```
curl 'http://127.0.0.1:4161/topics' 
curl 'http://127.0.0.1:4161/channels?topic=nsqdTopic'  
curl -X POST 'http://127.0.0.1:4161/topic/create?topic=lookupdTopic'  
curl -X POST 'http://127.0.0.1:4161/topic/delete?topic=lookupdTopic' 
curl -X POST 'http://127.0.0.1:4161/channel/create?topic=lookupdTopic&channel=lookupdChannel'
curl -X POST 'http://127.0.0.1:4161/channel/delete?topic=lookupdTopic&channel=lookupdChannel'
```
### 工作流程
1. 启动nsqlookupd服务
2. 启动nsqd服务，nsqd服务会向nsqlookup广播自己的节点信息，将自己的信息注册到nsqlookupd上
3. 生产者选一个topic向nsqd发布消息，这时nsqd会将这些信息同步到nsqlookupd服务，在消费者没有准备好时会将数据存放在本地
4. 消费者通过查询nsqlookupd或者直接绑定的方式连接nsqd，并创建channel和选择订阅自己感兴趣的topic
5. 有topic发布消息时，这些消息会被复制到各个channel，将消息推送(PUSH)给消费者消费

![nsq工作流程图](/images/nsq-lookups.png)
![message流通图](/images/nsq-message-flow.gif)

### stats参数解读
查看某个nsqd的状态实例，可以通过stats接口来看
```
http://10.13.82.72:8227/stats?topic=cts_core_events
```
数据的数据如下：
```
nsqd v0.3.8 (built w/go1.6.2)
start_time 2018-07-30T02:21:22Z
uptime 103h40m20.907107159s

Health: OK

   [cts_core_events] depth: 0     be-depth: 0     msgs: 165923   e2e%: 
      [group_member_updated     ] depth: 21393 be-depth: 17083 inflt: 1    def: 0    re-q: 8     timeout: 8     msgs: 165871   e2e%: 
        [V2 24b148db65b7:49574   ] state: 3 inflt: 1    rdy: 1    fin: 10462    re-q: 0        msgs: 10463    connected: 5h50m31s
      [namesearch               ] depth: 0     be-depth: 0     inflt: 0    def: 0    re-q: 0     timeout: 0     msgs: 165873   e2e%: 
        [V2 24b148db65b7:38860   ] state: 3 inflt: 0    rdy: 150  fin: 20       re-q: 0        msgs: 20       connected: 5h42m48s
      [space                    ] depth: 0     be-depth: 0     inflt: 0    def: 0    re-q: 0     timeout: 0     msgs: 165880   e2e%: 
        [V2 24b148db65b7:38882   ] state: 3 inflt: 0    rdy: 150  fin: 20       re-q: 0        msgs: 20       connected: 5h42m48s
      [wpsaccount               ] depth: 0     be-depth: 0     inflt: 0    def: 0    re-q: 0     timeout: 0     msgs: 165923   e2e%: 
        [V2 24b148db65b7:38874   ] state: 3 inflt: 0    rdy: 1500 fin: 20       re-q: 0        msgs: 20       connected: 5h42m48s
```

其中一个缩进代表一个层次：
cat_core_events 表示的是一个topic层次，各个参数的意义是：
- depth: 表示这个topic未被消费的消息总数，堆积太多的话有问题
- be-depth (backend-depth)：超出men-queue-size的未被消费的消息总数, 这些消息也就是写入磁盘的消息
- msgs: nsqd运行之后这个topic产生的消息总数（包括已经没消费和没被消费的）
- e2e%: 指定e2e-processing-latency-percentile时才会有，表示端到端在指定百分比的消息处理时间

第一层缩进 group_member_updated  表示的是topic下面的channel，各个参数的意义是：
* depth: 表示在这个channel中未被消费的消息总数
* be-depth: 超出men-queue-size的未被消费的消息总数, 这些消息也就是写入磁盘的消息
* inflt (in-flight-count)：发送过程中或者客户端处理过程中的数量，客户端没有发送FIN、REQ(重新入队列)和超时的消息数量。
* def(deferred_count)：重新入队并且还没有准备好重新发送的消息数量
* re-q(requeue_count): 超时或者客户端发送REQ的消息数量
* timeout: 超时未处理的消息数量
* msgs: channel处理的消息总数量

第二层缩进 V2 24b148db65b7:49574 表示的是这个channel下面连接的客户端：

* state: 客户端连接状态
* inflt(in-flight-count): 客户端正在处理的消息数量
* rdy(ready-count): 表示客户端能够处理的消息数量
* fin(finished): 客户端处理完成的消息数量
* re-q(requeue): 客户端发送重新排队的消息数量
* msgs: 客户端处理的消息数量
* connected: 客户端连接时长

实际上可以指明格式是json，这样子结合[文档](https://nsq.io/components/nsqadmin.html) 各个参数都是很明白的
```
http://10.13.82.72:8227/stats?format=json&topic=dsearch_events
```

以下是2018-08-03 的输出数据，显然这个数据要比上面的要好读很多：

```json
{
    "data": {
        "health": "OK",
        "start_time": 1532917282,
        "topics": [
            {
                "backend_depth": 0,
                "channels": [
                    {
                        "backend_depth": 0,
                        "channel_name": "ds",
                        "clients": [
                            {
                                "client_id": "a40251c8fb4e",
                                "connect_ts": 1533269935,
                                "deflate": false,
                                "finish_count": 24,
                                "hostname": "a40251c8fb4e",
                                "in_flight_count": 0,
                                "message_count": 24,
                                "name": "a40251c8fb4e",
                                "ready_count": 1,
                                "remote_address": "172.17.0.1:38848",
                                "requeue_count": 0,
                                "sample_rate": 0,
                                "snappy": false,
                                "state": 3,
                                "tls": false,
                                "tls_cipher_suite": "",
                                "tls_negotiated_protocol": "",
                                "tls_negotiated_protocol_is_mutual": false,
                                "tls_version": "",
                                "user_agent": "go-nsq/1.0.6",
                                "version": "V2"
                            }
                        ],
                        "deferred_count": 0,
                        "depth": 0,
                        "e2e_processing_latency": {
                            "count": 0,
                            "percentiles": null
                        },
                        "in_flight_count": 0,
                        "message_count": 5284,
                        "paused": false,
                        "requeue_count": 67,
                        "timeout_count": 67
                    },
                    {
                        "backend_depth": 0,
                        "channel_name": "plego",
                        "clients": [
                            {
                                "client_id": "WIN-366G26BO6HQ",
                                "connect_ts": 1533269981,
                                "deflate": false,
                                "finish_count": 9,
                                "hostname": "WIN-366G26BO6HQ",
                                "in_flight_count": 0,
                                "message_count": 9,
                                "name": "WIN-366G26BO6HQ",
                                "ready_count": 150,
                                "remote_address": "10.13.82.106:64640",
                                "requeue_count": 0,
                                "sample_rate": 0,
                                "snappy": false,
                                "state": 3,
                                "tls": false,
                                "tls_cipher_suite": "",
                                "tls_negotiated_protocol": "",
                                "tls_negotiated_protocol_is_mutual": false,
                                "tls_version": "",
                                "user_agent": "go-nsq/1.0.6",
                                "version": "V2"
                            },
                            {
                                "client_id": "WIN-366G26BO6HQ",
                                "connect_ts": 1533108206,
                                "deflate": false,
                                "finish_count": 1147,
                                "hostname": "WIN-366G26BO6HQ",
                                "in_flight_count": 0,
                                "message_count": 1147,
                                "name": "WIN-366G26BO6HQ",
                                "ready_count": 150,
                                "remote_address": "10.13.82.78:55391",
                                "requeue_count": 0,
                                "sample_rate": 0,
                                "snappy": false,
                                "state": 3,
                                "tls": false,
                                "tls_cipher_suite": "",
                                "tls_negotiated_protocol": "",
                                "tls_negotiated_protocol_is_mutual": false,
                                "tls_version": "",
                                "user_agent": "go-nsq/1.0.6",
                                "version": "V2"
                            },
                            {
                                "client_id": "WIN-366G26BO6HQ",
                                "connect_ts": 1533269984,
                                "deflate": false,
                                "finish_count": 3,
                                "hostname": "WIN-366G26BO6HQ",
                                "in_flight_count": 0,
                                "message_count": 3,
                                "name": "WIN-366G26BO6HQ",
                                "ready_count": 150,
                                "remote_address": "10.13.82.129:65515",
                                "requeue_count": 0,
                                "sample_rate": 0,
                                "snappy": false,
                                "state": 3,
                                "tls": false,
                                "tls_cipher_suite": "",
                                "tls_negotiated_protocol": "",
                                "tls_negotiated_protocol_is_mutual": false,
                                "tls_version": "",
                                "user_agent": "go-nsq/1.0.6",
                                "version": "V2"
                            },
                            {
                                "client_id": "WIN-366G26BO6HQ",
                                "connect_ts": 1533269952,
                                "deflate": false,
                                "finish_count": 5,
                                "hostname": "WIN-366G26BO6HQ",
                                "in_flight_count": 0,
                                "message_count": 5,
                                "name": "WIN-366G26BO6HQ",
                                "ready_count": 150,
                                "remote_address": "10.13.82.108:52359",
                                "requeue_count": 0,
                                "sample_rate": 0,
                                "snappy": false,
                                "state": 3,
                                "tls": false,
                                "tls_cipher_suite": "",
                                "tls_negotiated_protocol": "",
                                "tls_negotiated_protocol_is_mutual": false,
                                "tls_version": "",
                                "user_agent": "go-nsq/1.0.6",
                                "version": "V2"
                            }
                        ],
                        "deferred_count": 0,
                        "depth": 0,
                        "e2e_processing_latency": {
                            "count": 0,
                            "percentiles": null
                        },
                        "in_flight_count": 0,
                        "message_count": 5317,
                        "paused": false,
                        "requeue_count": 0,
                        "timeout_count": 0
                    }
                ],
                "depth": 0,
                "e2e_processing_latency": {
                    "count": 0,
                    "percentiles": null
                },
                "message_count": 5317,
                "paused": false,
                "topic_name": "dsearch_events"
            }
        ],
        "version": "0.3.8"
    },
    "status_code": 200,
    "status_txt": "OK"
}
```
json格式返回的参数基本上都是很清晰的，topics 下面是topics列表，为了方便示例，只选了一个topic。
其中topic这一层的 的参数意思：

- depth: 表示这个topic未被消费的消息总数，堆积太多的话有问题
- backend-depth：超出men-queue-size的未被消费的消息总数
- message_count: nsqd运行之后这个topic产生的消息总数（包括已经没消费和没被消费的）
- e2e_processing_latency: 指定e2e-processing-latency-percentile时才会有，表示端到端在指定百分比的消息处理时间
- pause: topic是否被暂停

channels这一层中比较重要的参数及其意思：

- depth: 表示在这个channel中未被消费的消息总数
- backend_depth: 超出men-queue-size的未被消费的消息总数
- in_flight_count：发送过程中或者客户端处理过程中的数量，客户端没有发送FIN、REQ(重新入队列)和超时的消息数量。
- deferred_count：重新入队并且还没有准备好重新发送的消息数量
- requeue_count: 超时或者客户端发送REQ的消息数量
- timeout_count: 超时未处理的消息数量
- message_count: channel处理的消息总数量

在topics->channels->clients ，是clients指的是连接到该channel上的所有客户端，主要的参数及其意思是：

- state: 客户端连接状态
- in_flight_count: 客户端正在处理的消息数量
- ready_count: 表示客户端能够处理的消息数量
- finish_count: 客户端处理完成的消息数量
- requeue_count: 客户端发送重新排队的消息数量，重新排队时因为消息没有被成功处理
- message_count: 客户端处理的消息数量
- connected: 客户端连接时长
- snappy, deflate 时配置消息压缩程度的
- tls\_\* 这些参数时当使用TLS连接时的参数

###   消息的生命周期
生产者发布一个消息之后，通过以下途径保证消息可靠性：

- client连接上nsqd，并通过RDY参数告知可以处理的message数量
- nsqd 把消息暂存起来或者推送到各个连接的channels，对于状态为REQ 和 timeout的都会被暂存起来
- 客户端消费消息之后，回复nsqd一个FIN表示消息成功处理，或者REQ表示表示这个消息将会继续重新加入到队列中, 如果处理时间超过配置的timeout时间，则nsqd会把这个消息当成超时处理
- 如果客户端一直没有回复直到超时，则这个消息会被当成超时消息重新加入到队列中
