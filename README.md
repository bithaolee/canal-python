# canal-python

## 一.canal-python 简介

canal-python 是阿里巴巴开源项目 [Canal](https://github.com/alibaba/canal)是阿里巴巴mysql数据库binlog的增量订阅&消费组件 的 python 客户端。为 python 开发者提供一个更友好的使用 Canal 的方式。Canal 是mysql数据库binlog的增量订阅&消费组件。

基于日志增量订阅&消费支持的业务：

1. 数据库镜像
2. 数据库实时备份
3. 多级索引 (卖家和买家各自分库索引)
4. search build
5. 业务cache刷新
6. 价格变化等重要业务消息

关于 Canal 的更多信息请访问 https://github.com/alibaba/canal/wiki

## 二.应用场景

canal-python 作为Canal的客户端，其应用场景就是Canal的应用场景。关于应用场景在Canal介绍一节已有概述。举一些实际的使用例子：

1.代替使用轮询数据库方式来监控数据库变更，有效改善轮询耗费数据库资源。

2.根据数据库的变更实时更新搜索引擎，比如电商场景下商品信息发生变更，实时同步到商品搜索引擎 Elasticsearch、solr等

3.根据数据库的变更实时更新缓存，比如电商场景下商品价格、库存发生变更实时同步到redis

4.数据库异地备份、数据同步

5.根据数据库变更触发某种业务，比如电商场景下，创建订单超过xx时间未支付被自动取消，我们获取到这条订单数据的状态变更即可向用户推送消息。

6.将数据库变更整理成自己的数据格式发送到kafka等消息队列，供消息队列的消费者进行消费。

## 三.工作原理

canal-python  是 Canal 的 python 客户端，它与 Canal 是采用的Socket来进行通信的，传输协议是TCP，交互协议采用的是 Google Protocol Buffer 3.0。

## 四.工作流程

1.Canal连接到mysql数据库，模拟slave

2.canal-python 与 Canal 建立连接

2.数据库发生变更写入到binlog

5.Canal向数据库发送dump请求，获取binlog并解析

4.canal-python 向 Canal 请求数据库变更

4.Canal 发送解析后的数据给canal-python

5.canal-python收到数据，消费成功，发送回执。（可选）

6.Canal记录消费位置。

## 五.快速启动

### 安装Canal

Canal 的安装以及配置使用请查看 https://github.com/alibaba/canal/wiki/QuickStart

### 环境要求
python >= 3

### 构建canal python客户端

````shell
pip install canal-python
````

### 建立与Canal的连接
````python
import time

from canal.client import Client
from canal.protocol import EntryProtocol_pb2
from canal.protocol import CanalProtocol_pb2

# 建立与canal服务端的连接
client = Client()
client.connect(host='127.0.0.1', port=11111)   # canal服务端部署的主机IP与端口
client.check_valid(username=b'', password=b'')  # 自行填写配置的数据库账户密码
# destination是canal服务端的服务名称， filter即获取数据的过滤规则，采用正则表达式
client.subscribe(client_id=b'1001', destination=b'example', filter=b'.*\\..*')

while True:
    message = client.get(100)
    # entries是每个循环周期内获取到数据集
    entries = message['entries']
    for entry in entries:
        entry_type = entry.entryType
        if entry_type in [EntryProtocol_pb2.EntryType.TRANSACTIONBEGIN, EntryProtocol_pb2.EntryType.TRANSACTIONEND]:
            continue
        row_change = EntryProtocol_pb2.RowChange()
        row_change.MergeFromString(entry.storeValue)
        event_type = row_change.eventType
        header = entry.header
        # 数据库名
        database = header.schemaName
        # 表名
        table = header.tableName
        event_type = header.eventType
        # row是binlog解析出来的行变化记录，一般有三种格式，对应增删改
        for row in row_change.rowDatas:
            format_data = dict()
            # 根据增删改的其中一种情况进行数据处理
            if event_type == EntryProtocol_pb2.EventType.DELETE:
                format_data['before'] = dict()
                for column in row.beforeColumns:
                    format_data['before'][column.name] = column.value
            elif event_type == EntryProtocol_pb2.EventType.INSERT:
                format_data['after'] = dict()
                for column in row.afterColumns:
                    format_data['after'][column.name] = column.value
            else:
                # format_data['before'] = format_data['after'] = dict()  采用下面的写法应该更好
                format_data['before'] = dict()
                format_data['after'] = dict()
                for column in row.beforeColumns:
                    format_data['before'][column.name] = column.value
                for column in row.afterColumns:
                    format_data['after'][column.name] = column.value
            # data即最后获取的数据，包含库名，表明，事务类型，改动数据
            data = dict(
                db=database,
                table=table,
                event_type=event_type,
                data=format_data,
            )
            print(data)
    time.sleep(1)

client.disconnect()
````
这个demo间隔一秒获取一次服务端的增量数据，并作相应的解析，代码中已做了简单的注释帮助理解，最后获取的data就是某个sql语句改动某一行的完整记录，通常有三种情况：
````python
# 设库test中有表test1，分别有id（int）和name（varchar）字段
# insert操作：insert into test.test1 values (1，'a')
# 此时data中应是如下情况
data = {'db':'test', 'table':'test1', 'event_type':1, 'data':{'after':{'id':'1', 'name':'a'}}}
# update操作：update test.test1 set id=2, name='b' where id=1
# 此时的data
data = {'db':'test', 'table':'test1', 'event_type':2, 'data':{'before':{'id':'1', 'name':'a'}, 'after':{'id':'2', 'name':'b'}}}
# delete操作:delete from test.test1 where id=2
# 此时的data
data = {'db':'test', 'table':'test1', 'event_type':3, 'data':{'before':{'id':'2', 'name':'b'}}}
````


更多详情请查看 [Sample](https://github.com/haozi3156666/canal-python/blob/master/example.py)

