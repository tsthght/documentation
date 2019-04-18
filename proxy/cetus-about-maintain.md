### Cetus中maintain/maintaining释疑

#### 1 概述

Cetus中有两个地方出现了“维护状态”，用户很容易搞混，分别如下：

```
## 设置Cetus的状态为 维护状态
set maintain true|false

## 设置后端某个DB的状态为 维护状态
update backends set state=maintaining where backend_ndx=1
```

下面将针对这两个操作及功能，分别详细阐述。

#### 2 Cetus的状态为维护状态

- 设置方法

设置Cetus的状态为维护/正常状态的方式如下：

```
## 设置Cetus维护状态
set maintain true|false
## 查看Cetus当前状态
show maintain status
```

- 功能的目的

加速客户端（例如lvs）与Cetus之间的空闲连接的断开。如果没有设置改参数，Cetus会等待`default-client-idle-timeout`秒，如果超过该阈值客户端仍旧没有断开，则Cetus才将该连接主动断开；如果设置了该参数，则Cetus会等待`default-maintained-client-idle-timeout`秒，超过该阈值则Cetus会将该连接主动断开。

```
## 查看default-client-idle-timeout，默认28800s
show variables like '%default-client-idle-timeout%'
## 查看default-maintained-client-idle-timeout，默认30s
show variables like '%default-maintained-client-idle-timeout%'
```

**特别注意**:

1 只有设置该参数后，访问的连接才生效。

2 该参数是**不会保存到配置文件**的，重启后，改参数会变成默认值`false`

- 的使用场景

该功能配合LVS使用，在Cetus升级等情况下需要从LV上中下线Cetus时，会通过该功能，加速客户端到Cetus的连接的关闭速度。

- 测试

本次测试的目：

```
验证 如果设置了set maintain true，则是否Cetus还会接受新连接
```

实验过程：

```
## 环境
主库 MySQL5.7：172.17.0.2
从库 MySQL5.7：172.17.0.4
读写分离版本Cetus：172.17.0.3
## 步骤
1 新建一条连接conn1，插入表一行数据
2 设置Cetus set maintain true
3 新建一条连接conn2，插入表一行数据
4 30s后,conn1 再插入一行数据
## 结果
conn2插入成功
conn1插入报错
ERROR 2013 (HY000): Lost connection to MySQL server during query
## 结论
该功能设置后，会加速空闲连接关闭，但是仍旧允许新连接的访问。
```

#### 3 DB的状态为维护状态

- 设置方法

设置后端DB状态为维护状态的方法如下：

```
## 设置后端某个DB的状态为 维护状态
update backends set state=maintaining where backend_ndx=1
## 查看各个后端的状态
select * from backends
```

- 功能的目的

该功能主要是某个后端DB需要做一些`下线维护`操作，需要将其状态设置为maintaining状态。如果设置了该状态，则会将后端DB**从配置文件中也删除**，下次启动，不会再有该后端DB。

maintaining状态和deleted状态的区别主要是：虽然都是人为操作，不再提供线下服务，但是maintaining可能后续维护后会再次人工操作使其上线提供服务；而deleted表示，该后端DB以后不会再提供线上服务了。

- 的使用场景

用于后台DB下线维护的场景。

- 测试

本次测试的目：

```
验证 如果设置了某台DB为maintainng，则是否该DB还会新建连接？是否已经建立的连接仍旧可以访问？
```

实验过程：

```
## 环境
主库 MySQL5.7：172.17.0.2
读写分离版本Cetus：172.17.0.3
## 步骤（默认set autocommit=1）
case1:
1 新建一条连接conn1，插入表一行数据
2 设置 DB 状态为maintaining
3 conn1再插入表一行数据
## 结果
第一次插入成功，第二次插入失败
ERROR 1203 (42000): service unavailable

case2:
1 新建一条连接conn1，开启事务，插入表一行数据
2 设置 DB 状态为maintaining
3 conn1再插入表一行数据
## 结果
第一次插入成功，第二次插入是比啊
ERROR 1105 (07000): (proxy) unable to continue processing command

## 结论
设置为maintaining状态后，新老连接都无法访问了。
```
