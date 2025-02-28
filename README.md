
# 概念


MySQL 基于 **Binlog** 的主从复制（Master\-Slave Replication）是 MySQL 数据库中实现数据复制的一种机制。在这种复制模式下，主库（Master）记录所有对数据库的修改操作（如 INSERT、UPDATE、DELETE 等）到 **二进制日志（Binlog）**，从库（Slave）则读取这些日志并执行相同的操作，从而保持与主库的数据一致性。


### 1\. **基本概念：**


* **主库（Master）**：所有的数据修改操作都在主库上执行，主库会将这些修改记录到 Binlog 中。
* **从库（Slave）**：从库通过连接到主库，读取主库的 Binlog 并将其中的操作应用到自己的数据上，达到与主库一致的目的。


### 2\. **Binlog（二进制日志）**


Binlog 是 MySQL 用来记录所有修改数据库的事件的日志文件，包括：


* **数据修改事件**：例如 INSERT、UPDATE、DELETE 等。
* **DDL 事件**：例如 CREATE、ALTER 等操作。
* **日志文件**：Binlog 是一个文件序列，每个文件都有一个唯一的名称。它是记录所有数据库更改的核心。


Binlog 存储的是操作日志而非数据本身，因此从库需要根据这些操作来更新自己的数据。


### 3\. **工作流程：**


基于 Binlog 的复制模式大致可以分为以下几个步骤：


#### 1\. **主库记录 Binlog**


* 在主库中执行任何更改操作时，这些操作会被记录到主库的 Binlog 文件中。
* 这些操作是顺序记录的，并按时间顺序排列。


#### 2\. **从库连接主库**


* 从库通过连接主库来获取 Binlog 数据。这个连接通过 **IO 线程** 来实现，从库会定期向主库请求新的 Binlog 事件。
* 从库会记录主库当前的 **Binlog 位置**，并请求从该位置开始同步数据。


#### 3\. **从库读取 Binlog**


* 从库的 IO 线程从主库获取到最新的 Binlog 事件，并将这些事件存储到从库的本地 Binlog 文件中。


#### 4\. **从库执行 Binlog 事件**


* 从库的 **SQL 线程** 会读取本地存储的 Binlog，并根据日志中的操作来执行相应的 SQL 语句。这样，从库就能够与主库的数据保持一致。


#### 5\. **保持同步**


* 主库和从库通过 Binlog 的持续同步保持一致。每次主库有新的数据更新时，这些更新会通过 Binlog 被传播到从库。


### 4\. **主从复制的关键组件：**


* **Binlog（Binary Log）**：记录所有更改数据的操作，主库通过 Binlog 来传递更改的内容。
* **I/O 线程**：从库的 I/O 线程负责从主库读取 Binlog 事件，并将其写入到从库的本地 Binlog 中。
* **SQL 线程**：从库的 SQL 线程负责读取本地 Binlog，执行其中的操作，使从库的数据保持同步。


### 5\. **复制模式的类型：**


基于 Binlog 的主从复制可以分为不同的复制模式，主要有以下几种：


* **异步复制**：


	+ 在这种模式下，主库执行操作后，不等待从库确认即返回。主库不会等待从库是否已经同步完数据。
	+ **优点**：性能高，不会因为等待从库确认而增加延迟。
	+ **缺点**：如果主库故障，可能会丢失未同步的数据。
* **半同步复制（Semi\-Synchronous Replication）**：


	+ 在这种模式下，主库在执行操作后，至少等待一个从库确认已接收数据并写入本地 Binlog 后才返回。这比完全异步复制稍微安全一些，但仍然存在延迟。
	+ **优点**：比异步复制更安全，至少一个从库能够确认同步。
	+ **缺点**：会增加一定的延迟，可能影响性能。
* **全同步复制（Synchronous Replication）**：


	+ 在这种模式下，主库在执行操作后，必须等待所有从库确认数据已经同步才会返回。
	+ **优点**：可以保证主库和所有从库的数据完全一致。
	+ **缺点**：性能开销大，延迟较高，且需要更多的资源。


### 6\. **复制延迟与容错性：**


* **复制延迟**：由于主库和从库是异步同步的，因此在高并发的场景中，从库可能会出现延迟，导致主库和从库的数据不一致。这种延迟通常是由于从库处理速度较慢，或者网络问题等原因导致。
* **容错性**：基于 Binlog 的主从复制，主库发生故障时，需要手动或自动切换到从库。虽然从库可以保持主库的副本，但在主库故障时可能会丢失一定的事务，因此对高可用性的需求需要结合其他技术（如 **MHA**、**ProxySQL**、**Group Replication** 等）来提高容错性。


### 7\. **优缺点：**


#### 优点：


* **简洁性**：基于 Binlog 的复制设置较为简单，容易理解和实现。
* **性能**：相比于 GTID 复制模式，Binlog 复制模式的性能开销较小。
* **兼容性强**：Binlog 复制是 MySQL 的标准复制方式，几乎所有版本都支持。
* **适用广泛**：适用于大多数需要主从同步的场景，特别是读写分离和负载均衡。


#### 缺点：


* **故障恢复较慢**：如果主库发生故障，可能需要人工干预来恢复主从复制关系，且在切换过程中可能会丢失未同步的数据。
* **可能出现数据不一致**：在网络延迟或复制延迟较大的情况下，主从数据可能暂时不一致。
* **复制延迟**：高负载时，主从复制可能存在延迟，导致从库的数据滞后于主库。


### 总结：


基于 Binlog 的主从复制是 MySQL 中实现数据复制的常见方式，它通过记录主库的二进制日志，并将日志同步到从库，从而保持数据一致性。这种方式在大多数应用中运行稳定、性能良好，但需要注意故障恢复、复制延迟等问题，适用于高可用架构中进行读写分离、负载均衡等场景。


~~binlog二进制日志文件记录了主服务器上所有数据库的更改操作~~


# 实操，一个主库两个从库之间进行主从复制


参考：[https://blog.csdn.net/2401\_85648342/article/details/139765433](https://github.com)


#### 数据持久化管理\-路径规划



```
.
├── docker-compose.yml
├── master1
│   ├── conf
│   ├── data
│   └── logs
├── slave1
│   ├── conf
│   ├── data
│   └── logs
└── slave2
    ├── conf
    ├── data
    └── logs

```


> 创建相关文件夹



```
# 创建持久化目录
mkdir -p /opt/mysql-compose/{master1/{data,logs,conf},slave1/{data,logs,conf},slave2/{data,logs,conf}}

# 修改权限
chmod -R 777 /opt/mysql-compose/{master1/{data,logs},slave1/{data,logs},slave2/{data,logs}}

# 临时测试-删除持久化的数据
rm -rf /opt/mysql-compose/{master1/{data/*,logs/*},slave1/{data/*,logs/*},slave2/{data/*,logs/*}}

#rm -rf /opt/mysql-compose/{master1/data/*,slave1/data/*,slave2/data/*}

```


> 分别上传配置文件(my.cnf)至 conf 目录下


#### master1配置文件my.cnf如下



```
[mysqld]
# 服务器唯一id，默认值1
server-id=11
# 设置日志格式，默认值ROW
binlog_format=STATEMENT
# 二进制日志名，默认binlog
# log-bin=binlog
# 设置需要复制的数据库，默认复制全部数据库
binlog-do-db=testdb
# 设置不需要复制的数据库
#binlog-ignore-db=mysql
#binlog-ignore-db=infomation_schema
#binlog-ignore-db=sys
#binlog-ignore-db=performance_schema

```

#### slave1配置文件my.cnf如下



```
# 服务器唯一id，每台服务器的id必须不同，如果配置其他从机，注意修改id
server-id=12
# 中继日志名，默认xxxxxxxxxxxx-relay-bin
#relay-log=relay-bin

```

#### slave2配置文件my.cnf如下



```
# 服务器唯一id，每台服务器的id必须不同，如果配置其他从机，注意修改id
server-id=13
# 中继日志名，默认xxxxxxxxxxxx-relay-bin
#relay-log=relay-bin

```

#### docker\-compose.yml内容如下



```
#version: "3.5"
services:
    #mysql:
    #    image: registry.cn-shenzhen.aliyuncs.com/multiway/mysql:8.0.29
    #    container_name: mysql8
    #    ports:
    #      - "13306:3306"
    #    restart: always
    #    environment:
    #      - MYSQL_ROOT_PASSWORD=123456
    #      - TZ=Asia/Shanghai
    #    volumes:
    #      - /opt/mysql-compose/master1/conf:/etc/mysql/conf.d
    #      - /opt/mysql-compose/master1/logs:/var/log/mysql
    #      - /opt/mysql-compose/master1/data:/var/lib/mysql
    mysql_master1:
        image: registry.cn-shenzhen.aliyuncs.com/multiway/mysql:8.0.29
        container_name: mysql_master1
        ports:
          - "13306:3306"
        restart: always
        environment:
          - MYSQL_ROOT_PASSWORD=123456
          - TZ=Asia/Shanghai
        volumes:
          - ./master1/mysql:/etc/mysql
          - ./master1/logs:/var/log/mysql
          - ./master1/data:/var/lib/mysql
    mysql_slave1:
        image: registry.cn-shenzhen.aliyuncs.com/multiway/mysql:8.0.29
        container_name: mysql_slave1
        ports:
          - "13307:3306"
        restart: always
        environment:
          - MYSQL_ROOT_PASSWORD=123456
          - TZ=Asia/Shanghai
        volumes:
          - ./slave1/mysql:/etc/mysql
          - ./slave1/logs:/var/log/mysql
          - ./slave1/data:/var/lib/mysql
    mysql_slave2:
        image: registry.cn-shenzhen.aliyuncs.com/multiway/mysql:8.0.29
        container_name: mysql_slave2
        ports:
          - "13308:3306"
        restart: always
        environment:
          - MYSQL_ROOT_PASSWORD=123456
          - TZ=Asia/Shanghai
        volumes:
          - ./slave2/mysql:/etc/mysql
          - ./slave2/logs:/var/log/mysql
          - ./slave2/data:/var/lib/mysql
    
   

```


> 注意：/var/lib/mysql/auto.cnf文件中的server\-uuid是 MySQL 数据库服务器的唯一标识符（UUID）。这个标识符用于标识 MySQL 实例，尤其在复制（Replication）设置中，它可以帮助区分不同的数据库实例。在 MySQL 中，/var/lib/mysql/auto.cnf 是一个自动生成的配置文件，通常包含 MySQL 实例的 UUID 信息。你可以通过这个文件来查看 MySQL 服务器的 UUID。它是在 MySQL 启动时自动生成的，并且通常不需要手动修改。如果docker\-compose挂载本地目录已有挂载数据请检查，如果有重复的修改server\-uuid或者删除这个auto.cnf文件之后重启mysql服务



```
[auto]
server-uuid=bc8c658e-ce63-11ef-89ae-0242ac130004

```

#### 运行docker\-compose命令启动服务



```
# 进入docker-compose.yml的所在层级文件夹
cd /opt/mysql-compose

# 运行docker compose 容器服务
docker compose up -d

#停止docker compose 容器服务
docker compose down

#查看docker compose 容器服务状态
docker compose ps

```

### 使用数据库管理工具连接主库和从库数据库



> 查看主库状态，连接master1数据库执行下面sql语句



```
SHOW MASTER STATUS;

```


> 查看结果




| File | Position | Binlog\_Do\_DB | Binlog\_Ignore\_DB | Executed\_Gtid\_Set |
| --- | --- | --- | --- | --- |
| binlog.000005 | 157 | testdb |  |  |



> *注意：如果是指定的数据库比如testdb的话，先在主数据库master1创建数据库，并创建表添加数据后，导出脚本，然后从库slave1和slave2也要创建数据库testdb导入执行sql脚本,使主从库数据一致，执行主从复制操作之前停止其他服务对主库的读写操作，不然会造成数据丢失等问题；简单来说在主从复制操作开始之前保证主从数据库数据一致*



> 分别连接slave1和slave2数据库执行下面sql语句,设置或修复 MySQL 的主从复制关系



```
#1.重置从服务器的复制设置。
#功能: 清除当前从服务器的所有复制设置
#作用: 重置从服务器的复制状态，包括清除 MASTER_* 配置、复制相关的文件、状态标记等。如果之前从服务器已经在运行复制任务，执行这个命令会停止复制进程并清除复制的所有状态信息。
#使用场景: 这通常在配置新的复制关系，或者需要重新设置复制时使用。
RESET SLAVE;

#2.配置从服务器连接到指定的主服务器（192.168.137.2），并设置复制的起始点。
#功能: 设置从服务器的主服务器连接信息及复制位置。
#作用: 配置从服务器如何连接到主服务器，以及从哪个二进制日志文件和位置开始复制。

CHANGE MASTER TO 
MASTER_HOST='192.168.137.2',   # 指定主服务器的 IP 地址或主机名，表示从服务器将连接到这个主机
MASTER_PORT=13306,             # 指定主服务器的端口号，通常 MySQL 的默认端口是 3306，根据实际情况修改
MASTER_USER='root',            # 指定主服务器上用于连接的用户名，通常是具备复制权限的用户
MASTER_PASSWORD='123456',      # 指定上述用户的密码，用于认证连接
MASTER_LOG_FILE='binlog.000005',# 指定主服务器的二进制日志文件名，从该文件的指定位置开始复制数据。
MASTER_LOG_POS=157; #指定从主服务器的二进制日志文件中从哪个位置开始复制。位置是一个数字，表示从该位置开始的日志条目

#3.启动从服务器的复制进程。
#功能: 启动从服务器的复制进程
#作用: 在执行完 CHANGE MASTER TO 后，启动从服务器的复制任务，使得从服务器开始连接主服务器，并从指定的二进制日志文件位置开始复制数据。

START SLAVE;

#4.查看从服务器的复制状态。
#功能: 显示从服务器的复制状态
#作用: 查看从服务器的当前复制状态，包括是否成功连接到主服务器，复制是否正常进行，以及任何可能出现的错误。
该命令会返回一个包含多个字段的结果，常用的字段有 Slave_IO_Running 和 Slave_SQL_Running，这两者的值为yes，分别表示 I/O 线程和 SQL 线程是否正在运行，Last_Error 显示最后一个错误信息等。

SHOW SLAVE STATUS;

```

 本博客参考[wgetCloud机场](https://tabijibiyori.org)。转载请注明出处！
